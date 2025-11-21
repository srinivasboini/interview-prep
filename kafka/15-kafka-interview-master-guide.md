# Kafka Interview Master Guide

## Overview

This comprehensive interview guide consolidates knowledge from all previous sections into a structured question bank for senior engineering roles in banking and financial services. It includes system design scenarios, troubleshooting case studies, architecture trade-off discussions, and behavioral questions tailored for staff/principal engineering positions.

**Target Roles**: Senior Java Developer, Staff Engineer, Principal Engineer, Solutions Architect
**Interview Types**: Technical depth, system design, troubleshooting, behavioral
**Context**: Enterprise banking with 13+ years experience in payment processing, transaction systems, and event-driven architectures

---

## Part 1: System Design Scenarios

### Scenario 1: Design a Payment Processing System

**Question**: "Design a real-time payment processing system for a global bank handling 100,000 transactions per second. The system must guarantee exactly-once processing, provide real-time fraud detection, maintain audit trails for 7 years, and support multi-region deployments for disaster recovery. Walk me through your architecture using Kafka."

**How to Approach**:
1. **Clarify requirements** (functional and non-functional)
2. **Propose high-level architecture**
3. **Deep dive into critical components**
4. **Discuss trade-offs and alternatives**
5. **Address failure scenarios**

**Comprehensive Answer**:

**1. Requirements Clarification**:

*Functional Requirements*:
- Accept payment requests from multiple channels (online banking, mobile, ATM)
- Process debit/credit operations atomically
- Real-time fraud detection and scoring
- Account balance validation
- Transaction notification
- 7-year audit trail retention

*Non-Functional Requirements*:
- **Throughput**: 100,000 TPS peak (1.5x for burst capacity = 150,000 TPS)
- **Latency**: p99 < 100ms for payment authorization
- **Availability**: 99.99% (52 minutes downtime/year)
- **Consistency**: Exactly-once processing (no duplicate debits/credits)
- **Compliance**: PCI-DSS, SOX, GDPR compliant
- **Multi-Region**: Active-passive DR with RPO < 1 minute, RTO < 5 minutes

**2. High-Level Architecture**:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Payment Gateway Layer                     │
│              (API Gateway with rate limiting)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kafka Cluster (Primary Region)                │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │  payment-   │  │   fraud-     │  │   account-events    │   │
│  │  requests   │  │   scores     │  │   (compacted)       │   │
│  │  (50 part)  │  │   (50 part)  │  │   (100 partitions)  │   │
│  └─────────────┘  └──────────────┘  └─────────────────────┘   │
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │ payment-    │  │ notifications│  │   audit-events      │   │
│  │ results     │  │              │  │   (7 year retention)│   │
│  │ (50 part)   │  │ (20 part)    │  │   (200 partitions)  │   │
│  └─────────────┘  └──────────────┘  └─────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ MirrorMaker 2.0
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Kafka Cluster (DR Region)                        │
│              (Passive, ready for failover)                       │
└─────────────────────────────────────────────────────────────────┘

Processing Services:
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Payment          │  │ Fraud Detection  │  │ Account          │
│ Authorization    │  │ Service          │  │ Management       │
│ Service          │  │ (ML scoring)     │  │ Service          │
│ (Transactional)  │  │                  │  │ (State Store)    │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

**3. Deep Dive: Critical Components**

**Topic Design**:

```
payment-requests:
  partitions: 50 (2,000 TPS per partition at 100K TPS)
  replication-factor: 3
  min.insync.replicas: 2
  retention.ms: 24 hours (reprocessing window)
  cleanup.policy: delete
  compression.type: lz4

  Key: paymentId (UUID)
  Value: PaymentRequest (Avro schema)

  Partitioning Strategy:
    - Default partitioner (uniform distribution)
    - Monitor for hotspots, use custom partitioner if needed

account-events:
  partitions: 100 (partition by accountId for ordering)
  replication-factor: 3
  min.insync.replicas: 2
  retention.ms: 7 days
  cleanup.policy: compact
  segment.ms: 1 hour

  Key: accountId
  Value: AccountSnapshot (Avro schema)

  Purpose: Maintains current account balance/state
  Pattern: Event sourcing with compaction

audit-events:
  partitions: 200 (high write throughput)
  replication-factor: 3
  min.insync.replicas: 2
  retention.ms: 7 years (2,208,000,000 ms)
  cleanup.policy: delete
  compression.type: zstd (max compression)

  Tiered Storage:
    - Hot tier (Kafka): Last 30 days
    - Warm tier (S3): 31 days - 1 year
    - Cold tier (Glacier): 1-7 years

  Key: transactionId
  Value: AuditEvent (Avro schema with digital signature)
```

**Payment Authorization Service (Exactly-Once Processing)**:

```java
/**
 * Payment authorization service with transactional exactly-once processing.
 *
 * Flow:
 * 1. Consume payment request (read_committed isolation)
 * 2. Validate account balance from compacted account-events topic
 * 3. Check fraud score from fraud-scores topic
 * 4. Atomically produce: debit event, credit event, result, audit event
 * 5. Commit transaction (atomic across all outputs)
 *
 * Guarantees:
 * - Exactly-once: Same payment request processed once despite retries
 * - Atomicity: All output events produced or none (no partial state)
 * - Idempotency: Reprocessing produces same result
 */
@Service
public class PaymentAuthorizationService {

    private final KafkaProducer<String, SpecificRecord> transactionalProducer;
    private final KafkaConsumer<String, PaymentRequest> consumer;

    @PostConstruct
    public void init() {
        // Transactional producer configuration
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9093");
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG,
                  "payment-authorization-" + instanceId); // Unique per instance
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

        transactionalProducer = new KafkaProducer<>(props);
        transactionalProducer.initTransactions();

        // Consumer configuration
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        consumer = new KafkaConsumer<>(consumerProps);
    }

    public void processPayments() {
        while (true) {
            ConsumerRecords<String, PaymentRequest> records = consumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<String, PaymentRequest> record : records) {
                try {
                    transactionalProducer.beginTransaction();

                    PaymentRequest request = record.value();

                    // 1. Validate account balance (read from compacted topic)
                    AccountSnapshot debitAccount = getAccountSnapshot(request.getDebitAccountId());
                    if (debitAccount.getBalance() < request.getAmount()) {
                        producePaymentResult(request.getPaymentId(), "INSUFFICIENT_FUNDS");
                        continue;
                    }

                    // 2. Check fraud score
                    FraudScore fraudScore = getFraudScore(request.getPaymentId());
                    if (fraudScore.getScore() > 0.8) {
                        producePaymentResult(request.getPaymentId(), "FRAUD_SUSPECTED");
                        continue;
                    }

                    // 3. Atomically produce debit, credit, result, and audit events
                    AccountDebitEvent debitEvent = AccountDebitEvent.newBuilder()
                        .setAccountId(request.getDebitAccountId())
                        .setAmount(request.getAmount())
                        .setTransactionId(request.getPaymentId())
                        .setTimestamp(System.currentTimeMillis())
                        .build();

                    AccountCreditEvent creditEvent = AccountCreditEvent.newBuilder()
                        .setAccountId(request.getCreditAccountId())
                        .setAmount(request.getAmount())
                        .setTransactionId(request.getPaymentId())
                        .setTimestamp(System.currentTimeMillis())
                        .build();

                    PaymentResult result = PaymentResult.newBuilder()
                        .setPaymentId(request.getPaymentId())
                        .setStatus("SUCCESS")
                        .setTimestamp(System.currentTimeMillis())
                        .build();

                    AuditEvent auditEvent = AuditEvent.newBuilder()
                        .setTransactionId(request.getPaymentId())
                        .setEventType("PAYMENT_AUTHORIZED")
                        .setPayload(request.toString())
                        .setTimestamp(System.currentTimeMillis())
                        .setSignature(signEvent(request)) // Digital signature for non-repudiation
                        .build();

                    // Send all events in transaction
                    transactionalProducer.send(new ProducerRecord<>(
                        "account-events", request.getDebitAccountId(), debitEvent));
                    transactionalProducer.send(new ProducerRecord<>(
                        "account-events", request.getCreditAccountId(), creditEvent));
                    transactionalProducer.send(new ProducerRecord<>(
                        "payment-results", request.getPaymentId(), result));
                    transactionalProducer.send(new ProducerRecord<>(
                        "audit-events", request.getPaymentId(), auditEvent));

                    // Send consumer offsets to transaction
                    Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
                    offsets.put(
                        new TopicPartition(record.topic(), record.partition()),
                        new OffsetAndMetadata(record.offset() + 1)
                    );
                    transactionalProducer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());

                    // Commit transaction (atomic commit of all events + offset)
                    transactionalProducer.commitTransaction();

                } catch (ProducerFencedException | OutOfOrderSequenceException e) {
                    // Fatal errors - producer fenced by another instance
                    logger.error("Fatal producer error", e);
                    break;
                } catch (Exception e) {
                    // Transient errors - abort and retry
                    logger.warn("Transaction failed, aborting", e);
                    transactionalProducer.abortTransaction();
                }
            }
        }
    }
}
```

**Multi-Region Disaster Recovery Setup**:

```properties
# MirrorMaker 2.0 configuration for active-passive DR

clusters = us-east-1, us-west-2

# Cluster connection details
us-east-1.bootstrap.servers = kafka-us-east-1:9093
us-west-2.bootstrap.servers = kafka-us-west-2:9093

# Replication from primary (us-east-1) to DR (us-west-2)
us-east-1->us-west-2.enabled = true
us-east-1->us-west-2.topics = payment-requests, payment-results, account-events, audit-events
us-east-1->us-west-2.topics.blacklist = .*-internal, .*-retry

# Sync consumer group offsets for fast failover
us-east-1->us-west-2.sync.group.offsets.enabled = true
us-east-1->us-west-2.sync.group.offsets.interval.seconds = 60
us-east-1->us-west-2.emit.checkpoints.enabled = true

# Replication performance
us-east-1->us-west-2.max.tasks = 10
us-east-1->us-west-2.replication.factor = 3

# Heartbeat configuration for detecting primary failure
us-east-1->us-west-2.heartbeats.topic.replication.factor = 3
checkpoints.topic.replication.factor = 3

# Failover detection
# If primary cluster is unreachable for > 30 seconds, trigger failover
```

**4. Trade-offs and Alternatives**:

| Decision | Chosen Approach | Alternative | Trade-off Reasoning |
|----------|----------------|-------------|---------------------|
| **Consistency Model** | Exactly-once with transactions | At-least-once with idempotent consumer | Payments require exactly-once. No duplicate debits acceptable. Performance overhead (~20%) acceptable for correctness. |
| **Partition Count** | 50 partitions for payment-requests | 100 or 200 partitions | 50 partitions balances throughput (2K TPS/partition) with operational complexity. Can scale to 100 partitions later. |
| **Replication Factor** | RF=3, min.insync.replicas=2 | RF=5 or RF=2 | RF=3 with min.insync.replicas=2 provides durability (tolerates 1 broker failure) without excessive storage cost. |
| **DR Strategy** | Active-passive with MirrorMaker 2.0 | Active-active or stretch cluster | Active-passive simpler operationally. RPO <1 min acceptable for banking. Active-active adds complexity (conflict resolution, dual writes). |
| **Audit Retention** | Tiered storage (Kafka + S3 + Glacier) | Keep all 7 years in Kafka | Tiered storage reduces cost by 97%. Hot data (30 days) in Kafka for fast access. Warm/cold in S3 for compliance. |
| **Consumer Groups** | Separate consumer groups per service | Single consumer group for all processing | Separation allows independent scaling and deployment. Authorization service can scale to 50 instances without affecting fraud detection service. |

**5. Failure Scenarios**:

**Scenario A: Broker Failure**:
- **Detection**: UnderReplicatedPartitions metric alerts
- **Impact**: No data loss (RF=3, min.insync.replicas=2). Partitions automatically re-elect leaders from ISR.
- **Recovery**: Replace failed broker. Kafka reassigns replicas automatically.
- **RPO/RTO**: RPO=0 (no data loss), RTO=0 (automatic failover in <1 second)

**Scenario B: Payment Authorization Service Instance Failure**:
- **Detection**: Consumer group rebalance triggered
- **Impact**: In-flight transaction aborted (producer fenced). Remaining instances take over partitions.
- **Recovery**: Kubernetes restarts failed pod. Consumer rejoins group after rebalance.
- **RPO/RTO**: RPO=0 (transactional producer ensures no partial state), RTO=30 seconds (rebalance time)

**Scenario C: Primary Region Failure**:
- **Detection**: MirrorMaker 2.0 heartbeat failure, monitoring alerts
- **Manual Failover Process**:
  1. Confirm primary region is unreachable (avoid split-brain)
  2. Promote DR cluster to primary (update DNS, load balancer)
  3. Services connect to DR cluster using same consumer group IDs
  4. Consumer groups resume from last synced offsets
- **RPO**: <1 minute (last offset sync interval)
- **RTO**: 5 minutes (manual failover + service restart)

**Scenario D: Duplicate Payment Request (Network Retry)**:
- **Detection**: Idempotent producer detects duplicate sequence number
- **Impact**: Kafka deduplicates at broker level. Same paymentId processed once.
- **Recovery**: Producer retries transparently. Downstream sees single payment.
- **RPO/RTO**: N/A (prevention, not recovery)

**6. Key Interview Points to Emphasize**:
- **Exactly-once semantics** critical for financial transactions (justify transactional overhead)
- **Partition design** impacts scalability (accountId for ordering, paymentId for distribution)
- **DR strategy** balances cost and risk (active-passive for 99.99% availability)
- **Tiered storage** solves 7-year retention cost challenge (97% savings)
- **Security** integrated throughout (TLS, SASL/SCRAM, ACLs, digital signatures)
- **Monitoring** proactive (lag, under-replicated partitions, consumer health)

---

### Scenario 2: Design a Real-Time Fraud Detection System

**Question**: "Design a real-time fraud detection system that processes 200,000 transactions per second, applies machine learning models for scoring, and flags suspicious transactions within 50ms p99 latency. The system must support model updates without downtime and maintain historical feature data for investigation."

**Comprehensive Answer**:

**Architecture Overview**:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Transaction Stream                          │
│                   (payment-requests topic)                       │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              Feature Engineering Service (Kafka Streams)         │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ Windowed     │  │ Account      │  │ Merchant             │ │
│  │ Aggregation  │  │ Behavior     │  │ Risk Profiling       │ │
│  │ (1min, 1hr)  │  │ (State Store)│  │ (State Store)        │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    transaction-features topic                    │
│               (enriched with behavioral features)                │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│          ML Scoring Service (Parallel Consumers)                 │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ Model v1     │  │ Model v2     │  │ Ensemble             │ │
│  │ (XGBoost)    │  │ (Neural Net) │  │ (Voting)             │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                                   │
│  Model Loading: Watch /models directory for new versions         │
│  Hot Reload: Load new model without service restart              │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                      fraud-scores topic                          │
│            (partitioned by transactionId, 50 partitions)         │
└─────────────────────────────────────────────────────────────────┘
                     │
                     ├──────────────────────────────────┐
                     ▼                                  ▼
┌────────────────────────────────┐  ┌──────────────────────────────┐
│  Alert Service                 │  │ Investigation Database       │
│  (high-score transactions)     │  │ (Elasticsearch)              │
│  Threshold: score > 0.8        │  │ (historical features + scores│
└────────────────────────────────┘  └──────────────────────────────┘
```

**Feature Engineering with Kafka Streams**:

```java
/**
 * Real-time feature engineering using Kafka Streams.
 *
 * Features generated:
 * - Transaction velocity (count in last 1 min, 1 hour, 24 hours)
 * - Amount aggregates (sum, avg in time windows)
 * - Account behavior (transaction frequency, typical amounts)
 * - Merchant risk score (historical fraud rate)
 * - Cross-account patterns (multiple cards from same device)
 *
 * Latency Target: p99 < 20ms (leaves 30ms for ML scoring)
 */
@Service
public class FraudFeatureEngineering {

    public Topology buildTopology() {
        StreamsBuilder builder = new StreamsBuilder();

        // Input: Raw payment requests
        KStream<String, PaymentRequest> payments = builder.stream(
            "payment-requests",
            Consumed.with(Serdes.String(), paymentRequestSerde())
        );

        // Feature 1: Transaction velocity (windowed aggregation)
        KTable<Windowed<String>, TransactionVelocity> velocityOneMinute = payments
            .groupByKey()
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .aggregate(
                TransactionVelocity::new,
                (key, payment, velocity) -> {
                    velocity.incrementCount();
                    velocity.addAmount(payment.getAmount());
                    return velocity;
                },
                Materialized.with(Serdes.String(), velocitySerde())
            );

        // Feature 2: Account behavior (session-based, 30-minute inactivity gap)
        KTable<Windowed<String>, AccountBehavior> accountBehavior = payments
            .groupBy(
                (key, payment) -> payment.getDebitAccountId(),
                Grouped.with(Serdes.String(), paymentRequestSerde())
            )
            .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)))
            .aggregate(
                AccountBehavior::new,
                (key, payment, behavior) -> {
                    behavior.recordTransaction(payment);
                    return behavior;
                },
                (key, behavior1, behavior2) -> behavior1.merge(behavior2),
                Materialized.with(Serdes.String(), accountBehaviorSerde())
            );

        // Feature 3: Merchant risk profile (global table, compacted)
        GlobalKTable<String, MerchantRiskProfile> merchantRisk = builder.globalTable(
            "merchant-risk-profiles",  // Compacted topic updated by batch job
            Consumed.with(Serdes.String(), merchantRiskSerde())
        );

        // Feature 4: Device fingerprint patterns (stateful processing)
        KTable<String, DeviceProfile> deviceProfiles = payments
            .groupBy(
                (key, payment) -> payment.getDeviceFingerprint(),
                Grouped.with(Serdes.String(), paymentRequestSerde())
            )
            .aggregate(
                DeviceProfile::new,
                (deviceId, payment, profile) -> {
                    profile.addTransaction(payment);
                    // Flag: Multiple account IDs from same device
                    if (profile.getDistinctAccountCount() > 3) {
                        profile.setHighRisk(true);
                    }
                    return profile;
                },
                Materialized.with(Serdes.String(), deviceProfileSerde())
            );

        // Enrich payment with all features
        KStream<String, TransactionFeatures> enrichedPayments = payments
            .leftJoin(
                merchantRisk,
                (paymentKey, payment) -> payment.getMerchantId(),  // Foreign key
                (payment, merchantProfile) -> {
                    TransactionFeatures features = new TransactionFeatures();
                    features.setPaymentId(payment.getPaymentId());
                    features.setAmount(payment.getAmount());
                    features.setMerchantId(payment.getMerchantId());
                    features.setAccountId(payment.getDebitAccountId());
                    features.setDeviceFingerprint(payment.getDeviceFingerprint());

                    // Add merchant risk
                    if (merchantProfile != null) {
                        features.setMerchantFraudRate(merchantProfile.getHistoricalFraudRate());
                        features.setMerchantCategory(merchantProfile.getCategory());
                    }

                    return features;
                }
            )
            .selectKey((key, features) -> features.getPaymentId());

        // Add windowed features (velocity, behavior)
        // Note: Simplified - in production, use complex join with windowed tables
        enrichedPayments
            .mapValues(features -> {
                // Lookup velocity from state store
                ReadOnlyWindowStore<String, TransactionVelocity> velocityStore =
                    streams.store("velocity-store", QueryableStoreTypes.windowStore());

                WindowStoreIterator<TransactionVelocity> iterator = velocityStore.fetch(
                    features.getAccountId(),
                    Instant.now().minus(Duration.ofMinutes(1)),
                    Instant.now()
                );

                if (iterator.hasNext()) {
                    TransactionVelocity velocity = iterator.next().value;
                    features.setTxCountLast1Min(velocity.getCount());
                    features.setAmountSumLast1Min(velocity.getTotalAmount());
                }

                return features;
            })
            .to("transaction-features", Produced.with(Serdes.String(), transactionFeaturesSerde()));

        return builder.build();
    }

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "fraud-feature-engineering");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9093");
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, "exactly_once_v2");

        // Performance tuning for low latency
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 100);  // Fast commits
        props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024);  // 10 MB

        // State store configuration
        props.put(StreamsConfig.STATE_DIR_CONFIG, "/var/kafka-streams-state");
        props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);  // Fast recovery

        KafkaStreams streams = new KafkaStreams(new FraudFeatureEngineering().buildTopology(), props);
        streams.start();
    }
}
```

**ML Scoring Service with Hot Model Reload**:

```java
/**
 * ML scoring service with zero-downtime model updates.
 *
 * Approach:
 * - Models stored in shared storage (S3, NFS)
 * - File watcher detects new model versions
 * - Load new model in background thread
 * - Atomic swap of model reference (volatile)
 * - Old model garbage collected
 *
 * Performance:
 * - Model inference: <10ms p99
 * - Consumer parallelism: 50 threads (one per partition)
 * - Throughput: 4,000 TPS per instance (50 instances = 200K TPS)
 */
@Service
public class FraudScoringService {

    private volatile MLModel currentModel;  // Atomic swap for hot reload
    private final Path modelDirectory = Paths.get("/models");

    @PostConstruct
    public void init() throws IOException {
        // Load initial model
        currentModel = loadLatestModel();

        // Watch for new model versions
        WatchService watchService = FileSystems.getDefault().newWatchService();
        modelDirectory.register(watchService, StandardWatchEventKinds.ENTRY_CREATE);

        // Background thread for hot reload
        new Thread(() -> {
            while (true) {
                try {
                    WatchKey key = watchService.take();
                    for (WatchEvent<?> event : key.pollEvents()) {
                        if (event.context().toString().endsWith(".model")) {
                            logger.info("New model detected, reloading...");
                            MLModel newModel = loadLatestModel();
                            currentModel = newModel;  // Atomic swap
                            logger.info("Model reloaded successfully");
                        }
                    }
                    key.reset();
                } catch (Exception e) {
                    logger.error("Error reloading model", e);
                }
            }
        }).start();
    }

    private MLModel loadLatestModel() throws IOException {
        // Find latest model version
        Path latestModelPath = Files.list(modelDirectory)
            .filter(p -> p.toString().endsWith(".model"))
            .max(Comparator.comparing(p -> p.toFile().lastModified()))
            .orElseThrow(() -> new RuntimeException("No model found"));

        logger.info("Loading model: " + latestModelPath);
        return MLModel.load(latestModelPath);  // XGBoost, TensorFlow, etc.
    }

    public void consumeAndScore() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9093");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "fraud-scoring-service");
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);  // Batch scoring
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024);
        props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 10);  // Low latency

        KafkaConsumer<String, TransactionFeatures> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("transaction-features"));

        KafkaProducer<String, FraudScore> producer = new KafkaProducer<>(producerProps());

        while (true) {
            ConsumerRecords<String, TransactionFeatures> records = consumer.poll(Duration.ofMillis(100));

            // Batch inference for efficiency
            List<TransactionFeatures> batch = new ArrayList<>();
            for (ConsumerRecord<String, TransactionFeatures> record : records) {
                batch.add(record.value());
            }

            if (!batch.isEmpty()) {
                long start = System.nanoTime();

                // Parallel scoring (if model supports batch inference)
                List<FraudScore> scores = batch.parallelStream()
                    .map(features -> {
                        double score = currentModel.predict(features);  // <10ms p99

                        FraudScore fraudScore = FraudScore.newBuilder()
                            .setTransactionId(features.getPaymentId())
                            .setScore(score)
                            .setModelVersion(currentModel.getVersion())
                            .setFeatures(features.toString())
                            .setTimestamp(System.currentTimeMillis())
                            .build();

                        return fraudScore;
                    })
                    .collect(Collectors.toList());

                long latency = (System.nanoTime() - start) / 1_000_000;  // ms
                logger.debug("Batch scoring latency: {}ms for {} records", latency, batch.size());

                // Produce scores
                for (FraudScore score : scores) {
                    producer.send(new ProducerRecord<>(
                        "fraud-scores",
                        score.getTransactionId(),
                        score
                    ));
                }

                producer.flush();
                consumer.commitSync();
            }
        }
    }
}
```

**Latency Breakdown**:
```
Total Budget: 50ms p99

Feature Engineering (Kafka Streams):
  - Stream processing: 5ms
  - State store lookups: 10ms
  - Output to transaction-features: 5ms
  Subtotal: 20ms

ML Scoring Service:
  - Kafka consumer poll: 5ms
  - Model inference: 10ms
  - Producer send: 5ms
  Subtotal: 20ms

Alert Service:
  - Consumer poll: 5ms
  - Alert logic: 3ms
  - External notification: 2ms
  Subtotal: 10ms

Total: 50ms (meeting SLA)
```

**Key Points to Emphasize**:
- **Kafka Streams for stateful feature engineering** (windowed aggregations, joins)
- **Hot model reload without downtime** (volatile reference, file watcher)
- **Batch inference for efficiency** (100 records per poll)
- **Horizontal scaling** (50 partitions = 50 parallel consumers)
- **Latency budgeting** (break down 50ms target across components)

---

## Part 2: Troubleshooting Case Studies

### Case Study 1: Sudden Consumer Lag Spike

**Scenario**: "Your payment processing system is experiencing a sudden spike in consumer lag. The `payment-authorization-service` consumer group lag has grown from 0 to 500,000 messages in 10 minutes. Customers are complaining about delayed payment confirmations. Walk me through your troubleshooting approach."

**Systematic Troubleshooting Workflow**:

**Step 1: Assess Impact and Scope**
```bash
# Check current lag across all partitions
kafka-consumer-groups.sh --bootstrap-server kafka:9093 \
  --group payment-authorization-service --describe

GROUP                          TOPIC              PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
payment-authorization-service  payment-requests   0          1000000         1500000         500000
payment-authorization-service  payment-requests   1          1000000         1500000         500000
...

# Key observations:
# - All partitions lagging equally (suggests consumer-side issue, not data skew)
# - Lag growing at 1000 msg/sec per partition (50 partitions * 1000 = 50K msg/sec)
# - Producer rate: 100K msg/sec (normal)
# - Consumer rate dropped from 100K to 50K msg/sec (50% degradation)
```

**Step 2: Check Consumer Health**
```bash
# Check consumer group members
kafka-consumer-groups.sh --bootstrap-server kafka:9093 \
  --group payment-authorization-service --members --describe

CONSUMER-ID                                    HOST            PARTITIONS
consumer-1-abc123                              10.0.1.10       [0,1,2,3,4]
consumer-2-def456                              10.0.1.11       [5,6,7,8,9]
...

# Observations:
# - Expected 50 consumers (one per partition)
# - Only 25 consumers active (50% capacity loss!)
# - Recent rebalance 5 minutes ago (suspicious timing)

# Check consumer metrics
curl http://10.0.1.10:8080/actuator/metrics/kafka.consumer.records.consumed.rate
# Result: 1000 records/sec (down from 2000 records/sec baseline)

curl http://10.0.1.10:8080/actuator/metrics/kafka.consumer.poll.latency.max
# Result: 5000ms (up from 100ms baseline) - SMOKING GUN
```

**Step 3: Investigate Root Cause**

**Hypothesis A: Consumer Processing Slowdown**
```java
// Check application logs for slow processing
grep "Payment processing took" /var/log/payment-service.log | tail -100

2024-01-15 10:05:23 WARN PaymentProcessor - Payment processing took 4523ms (paymentId=abc-123)
2024-01-15 10:05:25 WARN PaymentProcessor - Payment processing took 4891ms (paymentId=def-456)

// Pattern: Every payment taking ~5 seconds (normally 50ms)
// Suggests external dependency slowdown

// Check downstream database
psql -h postgres-primary -c "SELECT * FROM pg_stat_activity WHERE state = 'active';"

  pid  | application_name | state  | wait_event_type | wait_event | query_start        | state_change
-------+------------------+--------+-----------------+------------+--------------------+-------------------
 12345 | payment-service  | active | Lock            | relation   | 2024-01-15 10:00:01 | 2024-01-15 10:00:01
 12346 | payment-service  | active | Lock            | relation   | 2024-01-15 10:00:02 | 2024-01-15 10:00:02
 ...

// ROOT CAUSE FOUND: Database connection pool exhausted + lock contention
// 25 consumers restarted due to Kubernetes memory pressure
// Remaining 25 consumers overwhelmed, each connection blocked on DB locks
```

**Hypothesis B: Consumer Rebalance Thrashing**
```bash
# Check for frequent rebalances
kafka-consumer-groups.sh --bootstrap-server kafka:9093 \
  --group payment-authorization-service --state

COORDINATOR (ID)    ASSIGNMENT-STRATEGY       STATE
10.0.2.5 (5)        range                     Stable

# Check consumer logs for rebalance events
grep "RebalanceListener" /var/log/payment-service.log | tail -50

2024-01-15 10:00:05 INFO ConsumerCoordinator - Revoking previously assigned partitions [0,1,2,3,4]
2024-01-15 10:00:07 INFO ConsumerCoordinator - Adding newly assigned partitions [0,1,2,3,4,5,6,7,8,9]

// Consumers doubled partition ownership after 25 pods crashed
// Max.poll.interval.ms=300000 (5 minutes)
// Processing one record = 5 seconds
// max.poll.records=500
// 500 * 5 = 2500 seconds to process poll batch > 300 seconds timeout
// Consumer kicked out of group, triggering rebalance
```

**Step 4: Immediate Mitigation**

```bash
# Option 1: Scale up consumer instances (if capacity issue)
kubectl scale deployment payment-authorization-service --replicas=50

# Option 2: Reduce max.poll.records (if processing slowdown)
# Update consumer config:
max.poll.records=50  # Down from 500
max.poll.interval.ms=600000  # Increase timeout to 10 minutes

# Restart consumers with new config
kubectl rollout restart deployment payment-authorization-service

# Option 3: Fix database issue (if DB bottleneck)
# Kill long-running queries
psql -h postgres-primary -c "SELECT pg_terminate_backend(12345);"

# Increase connection pool size temporarily
spring.datasource.hikari.maximum-pool-size=50  # Up from 20

# Option 4: Enable circuit breaker for degraded DB
@CircuitBreaker(name = "database", fallbackMethod = "fallbackProcessPayment")
public void processPayment(Payment payment) {
    // Database call
}
```

**Step 5: Verify Recovery**
```bash
# Monitor lag reduction
watch -n 5 'kafka-consumer-groups.sh --bootstrap-server kafka:9093 \
  --group payment-authorization-service --describe | grep payment-requests'

# Expected: Lag decreasing at 2000 msg/sec per partition
# Lag should return to 0 within 10 minutes (500K backlog / 50K consumption rate)

# Monitor consumer metrics
curl http://10.0.1.10:8080/actuator/metrics/kafka.consumer.poll.latency.max
# Expected: <200ms

# Monitor payment latency
curl http://10.0.1.10:8080/actuator/metrics/payment.processing.latency.p99
# Expected: <100ms
```

**Step 6: Root Cause Analysis and Prevention**

**Root Cause**:
1. Kubernetes evicted 25 payment-service pods due to memory pressure
2. Remaining 25 consumers inherited double partition load
3. Database connection pool exhausted (20 connections for 25 consumers * 2 partitions)
4. Lock contention on `accounts` table caused 5-second query latency
5. Slow processing caused `max.poll.interval.ms` timeout violations
6. Consumer kicked from group, triggering rebalances, worsening lag

**Prevention**:
```yaml
# 1. Increase memory limits and requests (prevent eviction)
resources:
  requests:
    memory: 2Gi
  limits:
    memory: 4Gi

# 2. Configure PodDisruptionBudget (maintain minimum capacity)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
spec:
  minAvailable: 40  # Ensure at least 40 of 50 pods always running
  selector:
    matchLabels:
      app: payment-authorization-service

# 3. Tune consumer configuration for resilience
max.poll.records=100  # Lower batch size
max.poll.interval.ms=600000  # 10 minutes (handle transient slowdowns)
session.timeout.ms=300000  # 5 minutes
heartbeat.interval.ms=100000  # 100 seconds

# 4. Increase database connection pool
spring.datasource.hikari.maximum-pool-size=50
spring.datasource.hikari.minimum-idle=10

# 5. Add circuit breaker for database calls
resilience4j.circuitbreaker.instances.database:
  slidingWindowSize: 10
  failureRateThreshold: 50
  waitDurationInOpenState: 10000
  permittedNumberOfCallsInHalfOpenState: 5

# 6. Set up proactive alerting
# Alert on:
# - Consumer lag > 10,000 messages
# - Consumer poll latency p99 > 500ms
# - UnderReplicatedPartitions > 0
# - Database connection pool usage > 80%
```

**Key Interview Points**:
- **Systematic approach**: Assess impact → Check health → Investigate root cause → Mitigate → Verify → Prevent
- **Correlation analysis**: Consumer rebalance timing + database slowdown + Kubernetes eviction
- **Multi-layer troubleshooting**: Application logs, Kafka metrics, database queries, Kubernetes events
- **Immediate vs long-term fixes**: Scale up (immediate) + optimize config (long-term)
- **Proactive monitoring**: Lag, latency, resource utilization

---

### Case Study 2: Under-Replicated Partitions After Broker Failure

**Scenario**: "A broker failed in your production Kafka cluster. After the failure, you notice 200 under-replicated partitions. The broker has been offline for 30 minutes. Walk me through your response."

**Step-by-Step Response**:

**Step 1: Assess Current State**
```bash
# Check cluster health
kafka-broker-api-versions.sh --bootstrap-server kafka:9093

Error: Broker 5 is not available

# Check under-replicated partitions
kafka-topics.sh --bootstrap-server kafka:9093 --describe --under-replicated-partitions

Topic: payment-requests  Partition: 0   Leader: 1   Replicas: 1,2,5   Isr: 1,2
Topic: payment-requests  Partition: 5   Leader: 2   Replicas: 2,5,3   Isr: 2,3
...

# Key observations:
# - Broker 5 is down
# - 200 partitions have replica on broker 5
# - All partitions still available (leaders elected from ISR)
# - min.insync.replicas=2, current ISR≥2, so writes still succeeding

# Check if broker 5 is controller
kafka-metadata.sh --bootstrap-server kafka:9093 --describe --broker 5

# If controller: Automatic controller election already occurred
```

**Step 2: Determine Recovery Strategy**

**Decision Matrix**:

| Scenario | Action | Rationale |
|----------|--------|-----------|
| Broker recoverable in <1 hour | Wait for broker to rejoin | Automatic replica catch-up. No manual intervention needed. |
| Broker unrecoverable (hardware failure) | Replace broker, reassign replicas | Permanent failure requires new broker. |
| Broker stuck in zombie state | Force remove from cluster, replace | Avoid split-brain. Clean removal before replacement. |

**Step 3: Recovery Path A - Broker Comes Back Online**

```bash
# Scenario: Broker 5 restarts after 45 minutes

# 1. Broker rejoins cluster
# Check broker is online
kafka-broker-api-versions.sh --bootstrap-server kafka:9093 | grep "Broker 5"

# 2. Replicas automatically catch up
# Monitor under-replicated partitions
watch -n 10 'kafka-topics.sh --bootstrap-server kafka:9093 --describe --under-replicated-partitions | wc -l'

# Expected: Under-replicated count decreases over 10-30 minutes
# 200 → 150 → 100 → 50 → 0

# 3. Monitor replica lag
# Each replica must catch up to leader's log-end-offset
# Catch-up rate depends on:
# - replica.fetch.max.bytes (default 1MB)
# - num.replica.fetchers (default 1, consider increasing)

# Increase replica fetchers for faster catch-up
kafka-configs.sh --bootstrap-server kafka:9093 --entity-type brokers --entity-name 5 \
  --alter --add-config num.replica.fetchers=4

# 4. Verify ISR restoration
kafka-topics.sh --bootstrap-server kafka:9093 --describe --topic payment-requests

Topic: payment-requests  Partition: 0   Leader: 1   Replicas: 1,2,5   Isr: 1,2,5  ✅
```

**Step 4: Recovery Path B - Broker Permanently Failed**

```bash
# Scenario: Broker 5 hardware failure, needs replacement

# 1. Provision new broker (broker-6)
# Start new broker with fresh broker.id=6

# 2. Reassign replicas from broker-5 to broker-6
# Generate reassignment JSON
kafka-reassign-partitions.sh --bootstrap-server kafka:9093 --generate \
  --broker-list 0,1,2,3,4,6 \  # Exclude failed broker 5
  --topics-to-move-json-file topics-to-move.json

# topics-to-move.json:
{
  "topics": [
    {"topic": "payment-requests"},
    {"topic": "account-events"},
    {"topic": "audit-events"}
  ],
  "version": 1
}

# Output: Current assignment + Proposed assignment
# Save proposed assignment to reassignment.json

# 3. Execute reassignment
kafka-reassign-partitions.sh --bootstrap-server kafka:9093 --execute \
  --reassignment-json-file reassignment.json

# 4. Monitor reassignment progress
kafka-reassign-partitions.sh --bootstrap-server kafka:9093 --verify \
  --reassignment-json-file reassignment.json

# Expected output:
# Status of partition reassignment:
# Reassignment of partition payment-requests-0 is still in progress.
# Reassignment of partition payment-requests-1 is completed successfully.
# ...

# 5. Remove failed broker from cluster metadata
# After all reassignments complete:
kafka-configs.sh --bootstrap-server kafka:9093 --entity-type brokers --entity-name 5 --delete
```

**Step 5: Optimize Reassignment Performance**

```bash
# Reassignment can take hours for large partitions
# Tune throttling to balance speed vs impact on live traffic

# Set replication throttle (bytes/sec per broker)
kafka-configs.sh --bootstrap-server kafka:9093 --entity-type brokers --entity-name 6 \
  --alter --add-config follower.replication.throttled.rate=100000000  # 100 MB/sec

kafka-configs.sh --bootstrap-server kafka:9093 --entity-type brokers --entity-name 1,2,3,4 \
  --alter --add-config leader.replication.throttled.rate=100000000

# Set topic-level throttle
kafka-configs.sh --bootstrap-server kafka:9093 --entity-type topics --entity-name payment-requests \
  --alter --add-config leader.replication.throttled.replicas=*,follower.replication.throttled.replicas=*

# Monitor replication rate
kafka-broker-api-versions.sh --bootstrap-server kafka:9093 --command-config admin.properties

# After reassignment completes, remove throttle
kafka-configs.sh --bootstrap-server kafka:9093 --entity-type brokers --entity-name 6 \
  --alter --delete-config follower.replication.throttled.rate

kafka-configs.sh --bootstrap-server kafka:9093 --entity-type topics --entity-name payment-requests \
  --alter --delete-config leader.replication.throttled.replicas,follower.replication.throttled.replicas
```

**Step 6: Post-Recovery Validation**

```bash
# 1. Verify no under-replicated partitions
kafka-topics.sh --bootstrap-server kafka:9093 --describe --under-replicated-partitions
# Expected: No output

# 2. Verify all topics have correct replication factor
kafka-topics.sh --bootstrap-server kafka:9093 --describe | grep "ReplicationFactor:3" | wc -l
# Expected: All topics

# 3. Verify preferred leaders
kafka-preferred-replica-election.sh --bootstrap-server kafka:9093 --all-topics
# Rebalances leaders to preferred replica (first in ISR)

# 4. Check cluster balance
kafka-log-dirs.sh --bootstrap-server kafka:9093 --broker-list 0,1,2,3,4,6 --describe

# Expected: Roughly equal partition distribution across brokers

# 5. Monitor producer/consumer health
# Check for any errors during recovery
# Verify no message loss (compare producer acks with consumer offsets)
```

**Key Interview Points**:
- **min.insync.replicas=2 ensures availability** even with one replica down
- **Automatic vs manual recovery** (wait for broker vs reassign partitions)
- **Throttling reassignments** prevents overwhelming cluster during recovery
- **Preferred leader election** after recovery for optimal load distribution
- **Monitoring throughout recovery** (under-replicated partitions, replication lag)

---

## Part 3: Architecture Trade-off Discussions

### Trade-off 1: At-Least-Once vs Exactly-Once Semantics

**Question**: "When would you choose at-least-once semantics over exactly-once in a payment processing system? Walk me through the trade-offs."

**Comprehensive Analysis**:

**Option A: At-Least-Once Semantics**

**Configuration**:
```java
// Producer
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, false);
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);

// Consumer
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
// Manual commit after processing
consumer.commitSync();
```

**Guarantees**:
- Message delivered **at least once** (may be duplicated on retry)
- No message loss (acks=all, retries)
- Requires idempotent consumers (handle duplicates)

**When to Use**:
1. **Idempotent operations**: Operations that produce same result when repeated
   - Example: Updating account balance to a specific value (not incrementing)
   ```java
   // Idempotent: Absolute value
   account.setBalance(1000.00);  // Repeating has no effect

   // NOT idempotent: Relative change
   account.debit(100.00);  // Repeating debits $100 each time (BAD!)
   ```

2. **Downstream deduplication**: Consumer has built-in deduplication logic
   ```java
   @Transactional
   public void processPayment(PaymentEvent event) {
       // Check if already processed (database unique constraint)
       if (paymentRepository.existsByPaymentId(event.getPaymentId())) {
           logger.info("Duplicate payment {}, skipping", event.getPaymentId());
           return;  // Idempotent
       }

       // Process payment
       paymentRepository.save(new Payment(event));
   }
   ```

3. **Non-critical data**: Metrics, logs, analytics where occasional duplication is acceptable
   - Pageview events: Duplicate counted as slightly inflated metric
   - Application logs: Duplicate log line is noise, not critical

**Pros**:
- **Higher throughput**: 20-30% faster than exactly-once (no transactional overhead)
- **Lower latency**: p99 latency 30-50ms vs 50-100ms for exactly-once
- **Simpler**: No transactional producer/consumer coordination
- **Better availability**: No transaction coordinator dependency

**Cons**:
- **Duplicate risk**: Network retries, broker restarts, consumer rebalances cause duplicates
- **Requires idempotent consumers**: Application logic must handle duplicates
- **Audit trail gaps**: Duplicate processing may cause reconciliation issues

---

**Option B: Exactly-Once Semantics**

**Configuration**:
```java
// Producer
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "payment-producer-1");
props.put(ProducerConfig.ACKS_CONFIG, "all");

producer.initTransactions();

// Consumer
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
```

**Guarantees**:
- Message delivered **exactly once** (no duplicates, no loss)
- Atomic writes across multiple partitions
- Consumer offsets committed atomically with outputs

**When to Use**:
1. **Financial transactions**: Debits/credits must execute exactly once
   ```java
   producer.beginTransaction();
   producer.send(new ProducerRecord<>("accounts-debit", accountA, debit));
   producer.send(new ProducerRecord<>("accounts-credit", accountB, credit));
   producer.commitTransaction();
   // Both debit and credit committed atomically, or neither
   ```

2. **State updates**: Kafka Streams aggregations, joins, windowed computations
   ```java
   StreamsBuilder builder = new StreamsBuilder();
   builder.stream("payments")
       .groupByKey()
       .aggregate(...);  // Exactly-once ensures correct aggregates
   ```

3. **Compliance requirements**: Audit trails, regulatory reporting requiring exact counts
   - SOX compliance: Every financial transaction must be recorded exactly once
   - PCI-DSS: Payment card transactions must have exact audit trail

**Pros**:
- **No duplicates**: Guaranteed exactly-once delivery
- **Atomic multi-partition writes**: All-or-nothing transactions
- **Simplified application logic**: No deduplication needed in consumer
- **Compliance-friendly**: Meets strict audit requirements

**Cons**:
- **Lower throughput**: 20-30% overhead vs at-least-once
- **Higher latency**: Transaction coordinator adds 20-50ms p99
- **Increased complexity**: Transactional ID management, coordinator state
- **Lower availability**: Transaction coordinator failure blocks writes

---

**Decision Framework for Payment Processing**:

| Use Case | Semantic | Rationale |
|----------|----------|-----------|
| **Account debit/credit** | Exactly-once | Financial accuracy critical. No duplicates allowed. |
| **Payment notifications** | At-least-once | Duplicate notification acceptable (user sees "paid" twice, no harm). Idempotent: Notification service deduplicates by paymentId. |
| **Fraud score calculation** | At-least-once | Duplicate scoring is waste of CPU but doesn't affect correctness. Idempotent: Score is absolute value, not incremental. |
| **Audit logging** | Exactly-once | Compliance requires exact counts. Duplicate audit records violate SOX. |
| **Payment request ingestion** | Exactly-once | Prevents duplicate debits from API retries. |

**Hybrid Approach** (Recommended):
```java
// Payment pipeline with mixed semantics:

// 1. Ingestion: Exactly-once (prevent duplicate debits from API retries)
@Transactional
public void ingestPaymentRequest(PaymentRequest request) {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("payment-requests", request));
    producer.commitTransaction();
}

// 2. Authorization: Exactly-once (debit+credit atomic)
public void authorizePayment(PaymentRequest request) {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("accounts-debit", debitEvent));
    producer.send(new ProducerRecord<>("accounts-credit", creditEvent));
    producer.send(new ProducerRecord<>("audit-events", auditEvent));
    producer.commitTransaction();
}

// 3. Notification: At-least-once (idempotent, higher throughput)
public void sendNotification(PaymentResult result) {
    // Idempotent producer (no transactions)
    producer.send(new ProducerRecord<>("notifications", result));
    producer.flush();

    // Consumer deduplicates by paymentId
    if (notificationService.wasNotificationSent(result.getPaymentId())) {
        return;  // Skip duplicate
    }
    notificationService.send(result);
}
```

**Key Interview Points**:
- **Exactly-once has 20-30% overhead** but essential for financial operations
- **At-least-once with idempotent consumers** is viable optimization for non-critical paths
- **Hybrid approach** balances correctness and performance
- **Understand trade-offs**: Throughput vs correctness, latency vs guarantees

---

### Trade-off 2: Log Compaction vs Time-Based Retention

**Question**: "You're designing a topic to store account balances. Should you use log compaction or time-based retention? Explain your reasoning."

**Comprehensive Analysis**:

**Option A: Log Compaction**

**Configuration**:
```
# Topic: account-balances
partitions: 100 (partition by accountId)
replication-factor: 3
cleanup.policy: compact
min.cleanable.dirty.ratio: 0.5
segment.ms: 3600000  # 1 hour
min.compaction.lag.ms: 60000  # 1 minute
delete.retention.ms: 86400000  # 1 day

Key: accountId
Value: AccountSnapshot (current balance, timestamp)
```

**How It Works**:
```
Before Compaction:
Offset  Key        Value
0       acct-123   {balance: 1000, timestamp: T0}
1       acct-456   {balance: 2000, timestamp: T0}
2       acct-123   {balance: 1050, timestamp: T1}  # Updated
3       acct-789   {balance: 500, timestamp: T1}
4       acct-123   {balance: 1100, timestamp: T2}  # Updated
5       acct-456   {balance: 2100, timestamp: T2}  # Updated

After Compaction:
Offset  Key        Value
4       acct-123   {balance: 1100, timestamp: T2}  # Latest for acct-123
5       acct-456   {balance: 2100, timestamp: T2}  # Latest for acct-456
3       acct-789   {balance: 500, timestamp: T1}   # Latest for acct-789

// Old values for acct-123 and acct-456 deleted
// Only latest value per key retained
```

**When to Use**:
1. **Current state storage**: Topics storing latest entity state
   - Account balances (current balance, not history)
   - Customer profiles (current address, not past addresses)
   - Inventory levels (current stock, not every change)

2. **Event sourcing with snapshots**: Rebuilding current state from events
   - New consumer can replay compacted log to restore all account balances
   - No need to replay millions of historical transactions

3. **Changelog topics**: Kafka Streams state store backups
   ```java
   // Kafka Streams automatically creates compacted changelog topics
   StreamsBuilder builder = new StreamsBuilder();
   KTable<String, Account> accounts = builder.table(
       "account-balances",
       Materialized.as("account-balances-store")
   );
   // Changelog: account-balances-store-changelog (compacted)
   ```

**Pros**:
- **Space efficient**: Only latest value per key retained (90%+ space savings for frequently updated keys)
- **Fast state restoration**: New consumers read only current state (minutes vs hours)
- **Indefinite retention**: Can keep topic forever without unbounded growth
- **Self-healing**: Topic contains minimal data needed to restore state

**Cons**:
- **Loses history**: Cannot replay past values (e.g., "what was balance yesterday?")
- **Delayed compaction**: Messages retained until compaction runs (segment.ms + compaction lag)
- **Higher broker CPU**: Compaction process adds 10-20% CPU overhead
- **Tombstone management**: Deleting keys requires null value + delete.retention.ms wait

---

**Option B: Time-Based Retention**

**Configuration**:
```
# Topic: account-transactions
partitions: 100 (partition by accountId)
replication-factor: 3
cleanup.policy: delete
retention.ms: 2592000000  # 30 days
segment.ms: 86400000  # 1 day

Key: transactionId
Value: Transaction (amount, timestamp, account)
```

**How It Works**:
```
Retention: 30 days

Day 1:  Transaction-1 (acct-123, debit $100)
Day 5:  Transaction-2 (acct-123, credit $50)
Day 10: Transaction-3 (acct-123, debit $200)
Day 31: Transaction-1 deleted (>30 days old)
Day 35: Transaction-2 deleted (>30 days old)
...

// All transactions retained for 30 days, then deleted
```

**When to Use**:
1. **Append-only event log**: Immutable event history
   - Payment transactions (debit/credit history)
   - User activity logs (clickstream, page views)
   - Sensor data (IoT readings)

2. **Audit trail**: Regulatory compliance requiring historical data
   - SOX: 7-year retention of financial transactions
   - GDPR: Right to access historical personal data
   - PCI-DSS: 1-year transaction history

3. **Time-series analytics**: Analyzing trends over time windows
   - "What was the transaction volume last month?"
   - "Show me all failed payments in Q4 2024"

**Pros**:
- **Preserves history**: All events retained for retention period
- **Simple semantics**: Easy to understand and reason about
- **Lower broker CPU**: No compaction overhead
- **Predictable**: Time-based deletion is deterministic

**Cons**:
- **Space inefficient**: Duplicate keys consume space (e.g., account updated 1000 times = 1000 messages)
- **Slow state restoration**: New consumers must replay all events (hours for large topics)
- **Bounded retention**: Must choose retention period (trade-off: storage cost vs history depth)
- **No deduplication**: Duplicate events retained until expiration

---

**Decision Framework for Account Balances**:

**Scenario 1: Real-Time Balance Tracking**
```java
// Requirement: Consumer needs current balance for 100M accounts

// Option A: Log Compaction (RECOMMENDED)
// account-balances topic (compacted)
// 100M accounts * 1 KB per account = 100 GB (manageable)
// New consumer replays 100M messages to restore state (10 minutes)

// Option B: Time-Based Retention
// account-transactions topic (30-day retention)
// 100M accounts * 10 updates/day * 30 days = 30B messages (1 TB+)
// New consumer must replay 30B messages and aggregate to compute balances (hours)

// VERDICT: Log compaction wins for current state use case
```

**Scenario 2: Historical Transaction Analysis**
```java
// Requirement: Auditors need to see all transactions in last 7 years

// Option A: Log Compaction
// account-balances topic only has current balance
// Historical transactions lost (cannot answer "what was balance on 2020-01-01?")

// Option B: Time-Based Retention (RECOMMENDED)
// account-transactions topic (7-year retention)
// All debit/credit events retained for analysis
// Can recompute historical balances: replay transactions up to date

// VERDICT: Time-based retention wins for audit use case
```

**Hybrid Approach** (Best Practice):
```java
// Two topics serving different purposes:

// 1. account-balances (compacted)
//    - Purpose: Current balance for real-time authorization
//    - Cleanup: Log compaction
//    - Consumers: Payment authorization service (needs latest balance)
//    - Size: 100M accounts * 1 KB = 100 GB
@StreamsApplication
public class AccountBalanceAggregator {
    public Topology buildTopology() {
        StreamsBuilder builder = new StreamsBuilder();

        // Consume transaction events
        KStream<String, Transaction> transactions = builder.stream("account-transactions");

        // Aggregate to current balance
        KTable<String, AccountBalance> balances = transactions
            .groupBy((key, txn) -> txn.getAccountId())
            .aggregate(
                AccountBalance::new,
                (accountId, txn, balance) -> {
                    if (txn.getType() == DEBIT) {
                        balance.debit(txn.getAmount());
                    } else {
                        balance.credit(txn.getAmount());
                    }
                    return balance;
                },
                Materialized.as("account-balances-store")
            );

        // Materialize to compacted topic
        balances.toStream().to("account-balances");

        return builder.build();
    }
}

// 2. account-transactions (time-based retention)
//    - Purpose: Audit trail, historical analysis
//    - Cleanup: Delete after 7 years
//    - Consumers: Audit service, analytics pipeline
//    - Size: 100M accounts * 10 txn/day * 365 days * 7 years * 1 KB = 250 TB
//               (Tiered storage: Kafka 30 days, S3 7 years)

// Payment authorization service
@Service
public class PaymentAuthorizationService {
    public boolean authorizePayment(Payment payment) {
        // Read current balance from compacted topic (fast)
        AccountBalance balance = accountBalanceStore.get(payment.getAccountId());

        if (balance.getAmount() < payment.getAmount()) {
            return false;  // Insufficient funds
        }

        // Produce transaction to append-only log (audit trail)
        producer.send(new ProducerRecord<>(
            "account-transactions",
            payment.getTransactionId(),
            new Transaction(payment.getAccountId(), DEBIT, payment.getAmount())
        ));

        return true;
    }
}
```

**Key Interview Points**:
- **Log compaction for current state** (space efficient, fast restoration)
- **Time-based retention for audit trails** (preserves history, compliance)
- **Hybrid approach** leverages strengths of both (common in production)
- **Understand trade-offs**: Storage cost vs history depth, restoration time vs data completeness

---

## Part 4: Common Pitfalls and Best Practices

### Pitfall 1: Unbounded Key Cardinality in State Stores

**Problem**: "You're using Kafka Streams to aggregate user events. After running in production for a month, your application crashes with OutOfMemoryError. What happened?"

**Root Cause**:
```java
// BAD: Unbounded state store
StreamsBuilder builder = new StreamsBuilder();
KStream<String, ClickEvent> clicks = builder.stream("user-clicks");

// Aggregate clicks by sessionId
KTable<String, SessionAggregate> sessions = clicks
    .groupBy((key, click) -> click.getSessionId())  // Problem: sessionId never expires
    .aggregate(
        SessionAggregate::new,
        (sessionId, click, agg) -> {
            agg.addClick(click);
            return agg;
        },
        Materialized.as("session-store")  // State store grows unboundedly
    );

// After 1 month:
// - 1 billion unique sessionIds
// - 1 KB per session
// - State store size: 1 TB (doesn't fit in memory)
// - Result: OutOfMemoryError
```

**Why It Happens**:
- **State stores are held in memory** (backed by RocksDB on disk)
- **No automatic expiration** for keys in state stores
- **Unbounded cardinality**: sessionId, userId, deviceId keep growing
- **Memory pressure**: JVM heap exhausted, application crashes

**How to Fix**:

**Solution A: Use Windowed Aggregations**
```java
// GOOD: Time-bounded state with windowed aggregation
KTable<Windowed<String>, SessionAggregate> sessions = clicks
    .groupBy((key, click) -> click.getSessionId())
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))  // 1-hour windows
    .aggregate(
        SessionAggregate::new,
        (sessionId, click, agg) -> {
            agg.addClick(click);
            return agg;
        },
        Materialized.as("session-store")
    );

// Windowed aggregation:
// - retention.ms=24 hours (configurable)
// - Old windows automatically purged after 24 hours
// - State store bounded: max 24 hours of sessions
// - Size: 1M sessions/hour * 24 hours * 1 KB = 24 GB (manageable)
```

**Solution B: Use Session Windows**
```java
// GOOD: Session windows with inactivity gap
KTable<Windowed<String>, SessionAggregate> sessions = clicks
    .groupBy((key, click) -> click.getSessionId())
    .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)))
    .aggregate(
        SessionAggregate::new,
        (sessionId, click, agg) -> {
            agg.addClick(click);
            return agg;
        },
        (sessionId, agg1, agg2) -> agg1.merge(agg2),  // Merge sessions
        Materialized.as("session-store")
    );

// Session windows:
// - Automatically closed after 30 minutes of inactivity
// - Closed sessions purged after retention period
// - State store contains only active sessions
```

**Solution C: Implement Manual Expiration with Punctuator**
```java
// GOOD: Manual expiration for non-windowed aggregations
@StreamsProcessor
public class SessionAggregateProcessor extends AbstractProcessor<String, ClickEvent> {

    private KeyValueStore<String, SessionAggregate> stateStore;

    @Override
    public void init(ProcessorContext context) {
        this.stateStore = (KeyValueStore) context.getStateStore("session-store");

        // Schedule periodic cleanup (every 10 minutes)
        context.schedule(
            Duration.ofMinutes(10),
            PunctuationType.WALL_CLOCK_TIME,
            timestamp -> {
                long expirationTime = timestamp - Duration.ofHours(24).toMillis();

                // Scan state store, delete expired entries
                try (KeyValueIterator<String, SessionAggregate> iterator = stateStore.all()) {
                    while (iterator.hasNext()) {
                        KeyValue<String, SessionAggregate> entry = iterator.next();
                        if (entry.value.getLastUpdateTime() < expirationTime) {
                            stateStore.delete(entry.key);  // Expire old session
                            logger.debug("Expired session: {}", entry.key);
                        }
                    }
                }
            }
        );
    }

    @Override
    public void process(String key, ClickEvent click) {
        String sessionId = click.getSessionId();
        SessionAggregate agg = stateStore.get(sessionId);

        if (agg == null) {
            agg = new SessionAggregate(sessionId);
        }

        agg.addClick(click);
        agg.setLastUpdateTime(System.currentTimeMillis());
        stateStore.put(sessionId, agg);
    }
}
```

**Solution D: Use External Store with TTL**
```java
// ALTERNATIVE: Use Redis with TTL instead of Kafka state store
@Service
public class SessionAggregatorService {

    @Autowired
    private RedisTemplate<String, SessionAggregate> redisTemplate;

    public void processClick(ClickEvent click) {
        String sessionId = click.getSessionId();
        SessionAggregate agg = redisTemplate.opsForValue().get(sessionId);

        if (agg == null) {
            agg = new SessionAggregate(sessionId);
        }

        agg.addClick(click);

        // Store with 24-hour TTL (automatic expiration)
        redisTemplate.opsForValue().set(
            sessionId,
            agg,
            Duration.ofHours(24)
        );
    }
}
```

**Key Interview Points**:
- **State stores don't automatically expire** (unlike Redis TTL)
- **Use windowed aggregations** for time-bounded state
- **Use punctuators** for manual cleanup
- **Monitor state store size** (state.dir disk usage, RocksDB metrics)

---

### Pitfall 2: Incorrect Partition Key Causes Hotspots

**Problem**: "Your payment processing system has 50 partitions, but one consumer is processing 80% of messages while others are idle. What's the issue?"

**Root Cause**:
```java
// BAD: Partition by merchantId
ProducerRecord<String, Payment> record = new ProducerRecord<>(
    "payments",
    payment.getMerchantId(),  // Problem: Large merchants dominate traffic
    payment
);

// Distribution:
// Merchant-Amazon: 80% of payments → Partition-5 (hotspot)
// Merchant-SmallShop1: 0.1% of payments → Partition-10 (idle)
// Merchant-SmallShop2: 0.1% of payments → Partition-15 (idle)
// ...
// Result: One consumer overwhelmed, others idle
```

**Why It Happens**:
- **Kafka partitions by hash(key) % numPartitions**
- **Skewed key distribution**: Some keys have much higher volume (power law)
- **Partition affinity**: Same key always routes to same partition
- **Consumer parallelism limited**: One consumer per partition

**How to Fix**:

**Solution A: Composite Key with Shard Suffix**
```java
// GOOD: Add shard suffix for high-volume keys
public class ShardedPartitioner implements Partitioner {

    private static final Set<String> HIGH_VOLUME_MERCHANTS = Set.of(
        "merchant-amazon", "merchant-walmart", "merchant-target"
    );

    private static final int SHARDS_PER_HIGH_VOLUME_KEY = 10;

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {

        String merchantId = (String) key;
        int numPartitions = cluster.partitionCountForTopic(topic);

        if (HIGH_VOLUME_MERCHANTS.contains(merchantId)) {
            // Shard high-volume merchant across multiple partitions
            // amazon-0, amazon-1, amazon-2, ..., amazon-9
            int shard = ThreadLocalRandom.current().nextInt(SHARDS_PER_HIGH_VOLUME_KEY);
            String shardedKey = merchantId + "-" + shard;
            return Math.abs(shardedKey.hashCode()) % numPartitions;
        } else {
            // Normal partitioning for low-volume merchants
            return Math.abs(merchantId.hashCode()) % numPartitions;
        }
    }
}

// Result:
// Merchant-Amazon traffic spread across 10 partitions (8% each)
// Other merchants: 1 partition each
// Balanced consumer load
```

**Solution B: Use Random Partitioning (Lose Ordering)**
```java
// GOOD: Random partitioning for maximum balance (if ordering not needed)
ProducerRecord<String, Payment> record = new ProducerRecord<>(
    "payments",
    null,  // null key → random partition
    payment
);

// Or explicit random partition:
int partition = ThreadLocalRandom.current().nextInt(numPartitions);
ProducerRecord<String, Payment> record = new ProducerRecord<>(
    "payments",
    partition,
    null,  // Key not used for partitioning
    payment
);

// Trade-off:
// ✅ Perfect load balancing
// ❌ No ordering guarantee (payments from same merchant out-of-order)
```

**Solution C: Dedicated Topic for High-Volume Keys**
```java
// GOOD: Route high-volume keys to separate topic
@Service
public class PaymentRouter {

    public void publishPayment(Payment payment) {
        String topic;

        if (HIGH_VOLUME_MERCHANTS.contains(payment.getMerchantId())) {
            topic = "payments-high-volume";  // 100 partitions
        } else {
            topic = "payments-standard";  // 50 partitions
        }

        producer.send(new ProducerRecord<>(topic, payment.getMerchantId(), payment));
    }
}

// Trade-off:
// ✅ Isolate high-volume traffic (avoid impacting small merchants)
// ✅ Scale independently (different partition counts, consumer groups)
// ❌ Operational complexity (manage two topics)
```

**Solution D: Repartition Stream with Composite Key**
```java
// GOOD: Repartition in Kafka Streams to break hotspots
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Payment> payments = builder.stream("payments");  // Partitioned by merchantId

// Repartition by paymentId for balanced processing
KStream<String, Payment> repartitioned = payments
    .selectKey((merchantId, payment) -> payment.getPaymentId())  // Change key
    .through("payments-repartitioned");  // Repartition topic (50 partitions)

// Now processing is balanced across 50 partitions (one payment per key)
repartitioned
    .mapValues(payment -> processPayment(payment))
    .to("payment-results");
```

**Monitoring Hotspots**:
```java
// Expose per-partition metrics
@Component
public class PartitionMetrics {

    @Scheduled(fixedRate = 60000)  // Every minute
    public void reportPartitionLoad() {
        Map<TopicPartition, Long> partitionRecordCounts = getPartitionRecordCounts();

        for (Map.Entry<TopicPartition, Long> entry : partitionRecordCounts.entrySet()) {
            meterRegistry.gauge(
                "kafka.partition.records.count",
                Tags.of("topic", entry.getKey().topic(), "partition", String.valueOf(entry.getKey().partition())),
                entry.getValue()
            );
        }

        // Alert if any partition has >2x average load
        long avgLoad = partitionRecordCounts.values().stream()
            .mapToLong(Long::longValue)
            .average()
            .orElse(0);

        partitionRecordCounts.forEach((partition, count) -> {
            if (count > 2 * avgLoad) {
                logger.warn("Hotspot detected: {} has {} records (avg: {})",
                    partition, count, avgLoad);
            }
        });
    }
}
```

**Key Interview Points**:
- **Default partitioner uses hash(key) % numPartitions** (can create hotspots)
- **Power law distribution common** in real-world data (Amazon processes 80% of payments)
- **Solutions**: Shard high-volume keys, random partitioning, dedicated topics
- **Trade-offs**: Load balancing vs ordering guarantees

---

## Part 5: Behavioral Questions for Senior Roles

### Behavioral 1: Handling Production Outage

**Question**: "Tell me about a time when your Kafka-based system had a production outage. How did you handle it, and what did you learn?"

**STAR Framework Answer**:

**Situation**:
"In my previous role at UBS, we had a payment processing system handling 50,000 transactions per second using Kafka. The system had been running smoothly for 6 months."

**Task**:
"One Friday evening at 6 PM, we started receiving alerts that payment confirmations were delayed by 30+ minutes. Customers were calling the support line complaining about missing confirmations. I was on-call that week, so I was responsible for triaging and resolving the issue."

**Action**:
"I followed a systematic troubleshooting approach:

1. **Assessed impact** (5 minutes):
   - Checked consumer lag: `payment-notification-service` group had 2 million message lag (normally 0)
   - Verified producer health: Payment requests still being produced at normal rate
   - Scoped blast radius: Only notifications delayed, payments themselves were processing correctly

2. **Investigated root cause** (15 minutes):
   - Checked consumer metrics: All consumers stuck in rebalance loop
   - Reviewed logs: `ConsumerCoordinator: Member consumer-5 sending LeaveGroup request`
   - Identified pattern: Consumer-5 repeatedly joining and leaving group every 30 seconds
   - Checked Kubernetes: Consumer-5 pod in CrashLoopBackoff state
   - Found exception: `OutOfMemoryError: Java heap space`

3. **Implemented immediate mitigation** (10 minutes):
   - Killed problematic consumer-5 pod: `kubectl delete pod payment-notification-service-5`
   - Prevented auto-restart: Scaled deployment down to 9 replicas (from 10)
   - Result: Remaining 9 consumers stabilized, began processing backlog at 5,000 msg/sec
   - ETA to clear backlog: 2M messages / 5K msg/sec = 400 seconds (7 minutes)

4. **Root cause analysis** (next 2 hours):
   - Analyzed heap dump from crashed pod
   - Found memory leak: Consumer was caching notification templates in HashMap without expiration
   - Code review revealed recent change: Developer added template caching for performance optimization
   - Bug: Cache had no size limit or TTL, grew unbounded until OOM

5. **Implemented permanent fix** (Monday):
   - Replaced HashMap with Caffeine cache with max size and TTL:
   ```java
   // Before (memory leak):
   private Map<String, NotificationTemplate> templateCache = new HashMap<>();

   // After (bounded cache):
   private LoadingCache<String, NotificationTemplate> templateCache = Caffeine.newBuilder()
       .maximumSize(1000)
       .expireAfterWrite(Duration.ofHours(1))
       .build(key -> loadTemplate(key));
   ```
   - Added JVM memory monitoring: `jvm.memory.used`, `jvm.gc.pause` metrics
   - Configured heap dump on OOM: `-XX:+HeapDumpOnOutOfMemoryError`
   - Implemented circuit breaker for notification service: Graceful degradation instead of crash

6. **Post-mortem and prevention**:
   - Conducted blameless post-mortem with team
   - Added mandatory code review checklist item: \"Are there any unbounded data structures?\"
   - Implemented load testing with realistic data volume before production deployment
   - Set up proactive alerting: JVM heap usage >80%, consumer rebalance rate >1/hour"

**Result**:
"We resolved the immediate issue in 30 minutes (from alert to backlog cleared). Zero data loss due to Kafka's durability. Customer impact was limited to delayed notifications (30-45 minutes), but no financial impact (payments were processed correctly). The permanent fix prevented similar issues, and we've had zero OOM incidents in the 18 months since. The post-mortem process strengthened the team's on-call response and code review practices."

**Key Interview Points to Emphasize**:
- **Systematic approach**: Assess impact → Investigate → Mitigate → Fix → Prevent
- **Prioritization**: Immediate mitigation (stop bleeding) before root cause analysis
- **Communication**: Alerted stakeholders, provided ETAs, blameless post-mortem
- **Learning**: Turned incident into team learning opportunity (checklist, load testing)
- **Kafka knowledge**: Consumer rebalance, lag monitoring, durability guarantees

---

### Behavioral 2: Architectural Decision with Trade-offs

**Question**: "Describe a time when you had to make a difficult architectural decision involving Kafka. How did you evaluate trade-offs and justify your choice?"

**STAR Framework Answer**:

**Situation**:
"At JP Morgan, we were migrating a legacy payment reconciliation system to Kafka. The existing system processed 100,000 reconciliation events per day in nightly batch jobs. We wanted real-time reconciliation (within 5 minutes of payment) to detect fraud faster."

**Task**:
"I was the technical lead responsible for designing the Kafka architecture. The key decision was: Should we use Kafka Streams for stateful aggregation, or plain consumers with an external database (PostgreSQL)? Both approaches could meet the functional requirements, but had different trade-offs."

**Action**:

**1. Evaluated two architectural approaches:**

**Option A: Kafka Streams with State Stores**
```
Architecture:
- Kafka Streams application
- Stateful aggregation (KTable)
- State stores backed by RocksDB
- Changelog topics for recovery

Pros:
- Exactly-once processing (critical for reconciliation)
- Automatic failover (standby replicas)
- Scalability (partition-based parallelism)
- No external database dependency

Cons:
- Team unfamiliar with Kafka Streams (learning curve)
- Operational complexity (state store backups, recovery)
- State size concerns (100M accounts * 1 KB = 100 GB)
```

**Option B: Plain Consumers + PostgreSQL**
```
Architecture:
- KafkaConsumer (poll-process-commit loop)
- PostgreSQL for aggregation state
- Connection pooling (HikariCP)
- Read-committed isolation level

Pros:
- Team familiar with PostgreSQL (low learning curve)
- SQL queries for debugging/auditing
- Mature operational tooling (backups, monitoring)
- Flexible schema evolution

Cons:
- External database dependency (single point of failure)
- At-least-once processing (requires idempotency)
- Scalability concerns (database write bottleneck)
- Higher latency (network round-trip per message)
```

**2. Conducted proof-of-concept (POC) for both approaches:**

Spent 1 week building POCs with representative data (10M reconciliation events). Measured:
- **Throughput**: Kafka Streams: 50K msg/sec, PostgreSQL: 20K msg/sec
- **Latency**: Kafka Streams: p99 50ms, PostgreSQL: p99 200ms
- **Recovery time**: Kafka Streams: 5 min (standby replicas), PostgreSQL: 30 sec (database always available)
- **Operational complexity**: Kafka Streams: High (monitoring state stores, changelog topics), PostgreSQL: Medium (familiar tooling)

**3. Made recommendation:**

Recommended **Option B (Plain Consumers + PostgreSQL)** despite lower throughput. Justification:

- **Team familiarity**: Team had 5 years of PostgreSQL experience, zero Kafka Streams experience. Learning curve would delay project by 2-3 months.
- **Requirements met**: 20K msg/sec throughput sufficient for 100K events/day (avg 1 event/sec, peak 10 events/sec).
- **Debuggability**: SQL queries for reconciliation mismatch investigation critical for auditors.
- **Risk mitigation**: PostgreSQL high-availability (primary + replica) acceptable for 99.9% SLA.

However, added **future migration path** to Kafka Streams:
- Architected consumers with clean separation (processing logic in service layer, Kafka consumer in adapter layer)
- Documented Kafka Streams design for future reference
- Committed to revisiting decision if throughput requirements increased 10x

**4. Addressed concerns:**

Leadership questioned PostgreSQL bottleneck risk. I mitigated with:
- **Horizontal partitioning**: Sharded PostgreSQL database by accountId (10 shards)
- **Connection pooling**: HikariCP with 50 connections per shard
- **Load testing**: Validated 50K msg/sec capacity (2.5x headroom over requirement)
- **Monitoring**: Set up alerts on database CPU >70%, connection pool usage >80%

**Result**:
"We delivered the real-time reconciliation system on time (3 months). It's been running in production for 2 years, processing 200K events/day (2x growth) with p99 latency <100ms and 99.95% availability. We haven't needed to migrate to Kafka Streams yet, though we've revisited the decision annually. The team's PostgreSQL expertise allowed rapid troubleshooting during incidents, which justified the trade-off."

**Key Interview Points to Emphasize**:
- **Evaluated multiple options** with clear pros/cons
- **POC before decision**: Measured real performance, not assumptions
- **Context-driven**: Chose based on team skills, requirements, risk tolerance (not "best practice")
- **Communicated trade-offs**: Justified decision to stakeholders with data
- **Future-proofed**: Architected for evolution (clean code boundaries, documented alternative)

---

## Part 6: Banking-Specific Interview Scenarios

### Banking Scenario 1: PCI-DSS Compliance

**Question**: "Design a Kafka-based payment card processing system that complies with PCI-DSS requirements. How would you handle sensitive cardholder data?"

**Comprehensive Answer**:

**PCI-DSS Key Requirements**:
1. **Encrypt cardholder data at rest and in transit**
2. **Restrict access to cardholder data** (need-to-know basis)
3. **Maintain audit trail** of all access to cardholder data
4. **Regularly test security** systems and processes
5. **Protect stored cardholder data** (truncation, hashing, tokenization)

**Architecture**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Payment Gateway (PCI Scope)                   │
│                                                                   │
│  1. Receive card data (PAN, CVV, expiry)                        │
│  2. Tokenize PAN → Token (vault service)                        │
│  3. Encrypt CVV → Encrypted blob                                │
│  4. Publish tokenized payload to Kafka                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Kafka Cluster (Reduced PCI Scope)                │
│                                                                   │
│  Topic: payment-requests                                         │
│  Payload:                                                        │
│    - Token: "tok_abc123" (NOT raw PAN)                          │
│    - Encrypted CVV: "encrypted_blob"                            │
│    - Amount, merchant, timestamp                                │
│                                                                   │
│  Security:                                                       │
│    - TLS 1.3 encryption (in-transit)                            │
│    - SASL/SCRAM authentication                                  │
│    - ACLs (role-based access)                                   │
│    - No PAN stored in Kafka                                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ├─────────────────────────────────────────┐
                         ▼                                         ▼
┌─────────────────────────────────┐  ┌───────────────────────────────┐
│  Authorization Service          │  │  Fraud Detection Service      │
│  (PCI Scope)                    │  │  (Non-PCI Scope)              │
│                                 │  │                               │
│  1. Detokenize token → PAN     │  │  - Consumes tokenized data    │
│  2. Decrypt CVV                 │  │  - No access to raw PAN/CVV   │
│  3. Call card network (Visa)    │  │  - ML scoring on metadata     │
│  4. Tokenize response           │  │                               │
└─────────────────────────────────┘  └───────────────────────────────┘
```

**Key Design Decisions**:

**1. Tokenization (Reduce PCI Scope)**:
```java
/**
 * Tokenize PAN before publishing to Kafka.
 *
 * PCI-DSS Requirement 3.4: Render PAN unreadable anywhere it is stored
 *
 * Approach: Format-Preserving Encryption (FPE) tokenization
 * - Token looks like PAN (16 digits, passes Luhn check)
 * - Irreversible without vault access
 * - Reduces PCI scope (Kafka cluster outside PCI boundary)
 */
@Service
public class PaymentGateway {

    @Autowired
    private TokenVaultClient tokenVault;  // PCI-compliant vault (e.g., HashiCorp Vault)

    public void processPaymentRequest(PaymentCardRequest request) {
        // 1. Tokenize PAN
        String token = tokenVault.tokenize(request.getPan());

        // 2. Encrypt CVV (not tokenized, never stored)
        String encryptedCvv = encryptCvv(request.getCvv());

        // 3. Create tokenized payload (safe for Kafka)
        PaymentRequest kafkaPayload = PaymentRequest.newBuilder()
            .setPaymentId(UUID.randomUUID().toString())
            .setToken(token)  // Token, not PAN
            .setEncryptedCvv(encryptedCvv)
            .setAmount(request.getAmount())
            .setMerchantId(request.getMerchantId())
            .setTimestamp(System.currentTimeMillis())
            .build();

        // 4. Publish to Kafka (PAN never touches Kafka)
        producer.send(new ProducerRecord<>("payment-requests", kafkaPayload));

        // 5. Audit log
        auditLog.log(AuditEvent.builder()
            .action("TOKENIZE_PAN")
            .userId(request.getUserId())
            .timestamp(System.currentTimeMillis())
            .paymentId(kafkaPayload.getPaymentId())
            .build());
    }

    private String encryptCvv(String cvv) {
        // Encrypt CVV with AES-256-GCM
        // Key stored in HSM (Hardware Security Module)
        return encryptionService.encrypt(cvv);
    }
}
```

**2. Encryption (In-Transit and At-Rest)**:
```properties
# PCI-DSS Requirement 4: Encrypt transmission of cardholder data across open, public networks

# Broker configuration (TLS 1.3)
listeners=SASL_SSL://0.0.0.0:9093
ssl.protocol=TLSv1.3
ssl.enabled.protocols=TLSv1.3
ssl.cipher.suites=TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256

# Certificate configuration
ssl.keystore.location=/var/private/kafka.keystore.jks
ssl.keystore.password=${KEYSTORE_PASSWORD}  # Stored in secrets manager
ssl.key.password=${KEY_PASSWORD}
ssl.truststore.location=/var/private/kafka.truststore.jks
ssl.truststore.password=${TRUSTSTORE_PASSWORD}

# Mutual TLS (client authentication)
ssl.client.auth=required

# Disk encryption (at-rest)
# PCI-DSS Requirement 3.5: Document and implement procedures to protect keys
# - Use LUKS (Linux Unified Key Setup) for full-disk encryption
# - Keys stored in HSM (Hardware Security Module)
# Command: cryptsetup luksFormat /dev/sdb --key-file=/path/to/hsm-key
```

**3. Access Control (Role-Based ACLs)**:
```bash
# PCI-DSS Requirement 7: Restrict access to cardholder data by business need-to-know

# Authorization Service: Full access (read tokenized data, call detokenization vault)
kafka-acls.sh --bootstrap-server kafka:9093 --add \
  --allow-principal User:authorization-service \
  --operation Read --topic payment-requests \
  --resource-pattern-type literal

# Fraud Detection Service: Read-only, no detokenization access
kafka-acls.sh --bootstrap-server kafka:9093 --add \
  --allow-principal User:fraud-service \
  --operation Read --topic payment-requests \
  --resource-pattern-type literal

# Analytics Service: Deny access (no business need)
kafka-acls.sh --bootstrap-server kafka:9093 --add \
  --deny-principal User:analytics-service \
  --operation Read --topic payment-requests \
  --resource-pattern-type literal

# Principle: Least privilege (grant only necessary permissions)
```

**4. Audit Logging**:
```java
/**
 * PCI-DSS Requirement 10: Track and monitor all access to network resources and cardholder data
 *
 * Must log:
 * - User identification
 * - Type of event
 * - Date and time
 * - Success or failure indication
 * - Origination of event
 * - Identity or name of affected data, system component, or resource
 */
@Service
public class AuditLogger {

    @Autowired
    private KafkaProducer<String, AuditEvent> auditProducer;

    public void logCardholderDataAccess(String userId, String action, String paymentId, boolean success) {
        AuditEvent event = AuditEvent.newBuilder()
            .setEventId(UUID.randomUUID().toString())
            .setUserId(userId)
            .setAction(action)  // e.g., "DETOKENIZE_PAN", "READ_PAYMENT", "AUTHORIZE_PAYMENT"
            .setPaymentId(paymentId)
            .setTimestamp(System.currentTimeMillis())
            .setSuccess(success)
            .setSourceIp(getClientIp())
            .setSourceService(getServiceName())
            .build();

        // Publish to immutable audit log (7-year retention)
        auditProducer.send(new ProducerRecord<>("audit-events", event));
    }
}

// Audit topic configuration
// PCI-DSS Requirement 10.7: Retain audit trail history for at least one year
kafka-topics.sh --create --topic audit-events \
  --partitions 50 \
  --replication-factor 3 \
  --config retention.ms=220752000000 \  # 7 years (regulatory requirement)
  --config min.insync.replicas=2 \
  --config cleanup.policy=delete \
  --config compression.type=zstd  # High compression for long retention
```

**5. Key Rotation**:
```java
/**
 * PCI-DSS Requirement 3.6: Fully document and implement all key-management processes
 *
 * - Rotate encryption keys annually
 * - Maintain key version in message metadata
 * - Support decryption with old keys during rotation period
 */
@Service
public class EncryptionService {

    private final Map<Integer, SecretKey> keys = new ConcurrentHashMap<>();
    private volatile int currentKeyVersion = 1;

    @Scheduled(cron = "0 0 0 1 1 *")  // January 1st annually
    public void rotateKey() {
        int newVersion = currentKeyVersion + 1;
        SecretKey newKey = generateKey();  // Generate from HSM
        keys.put(newVersion, newKey);
        currentKeyVersion = newVersion;

        logger.info("Rotated encryption key to version {}", newVersion);
        auditLog.log("KEY_ROTATION", newVersion);
    }

    public String encrypt(String plaintext) {
        int version = currentKeyVersion;
        SecretKey key = keys.get(version);

        byte[] encrypted = aesCipher.encrypt(plaintext.getBytes(), key);

        // Prepend key version to ciphertext
        return version + ":" + Base64.getEncoder().encodeToString(encrypted);
    }

    public String decrypt(String ciphertext) {
        // Extract key version from ciphertext
        String[] parts = ciphertext.split(":");
        int version = Integer.parseInt(parts[0]);
        byte[] encrypted = Base64.getDecoder().decode(parts[1]);

        // Decrypt with correct key version
        SecretKey key = keys.get(version);
        return new String(aesCipher.decrypt(encrypted, key));
    }
}
```

**6. Monitoring and Alerting**:
```java
/**
 * PCI-DSS Requirement 10.6: Review logs and security events for all system components
 * PCI-DSS Requirement 11.4: Use intrusion-detection and/or intrusion-prevention techniques
 */
@Component
public class SecurityMonitoring {

    @Scheduled(fixedRate = 60000)  // Every minute
    public void detectAnomalies() {
        // 1. Detect unauthorized access attempts
        List<AuditEvent> deniedAccess = auditRepository.findBySuccessFalse(
            Instant.now().minus(Duration.ofMinutes(5))
        );

        if (deniedAccess.size() > 10) {
            alertService.send("High rate of denied access attempts: " + deniedAccess.size());
        }

        // 2. Detect unusual detokenization volume
        long detokenizations = auditRepository.countByAction(
            "DETOKENIZE_PAN",
            Instant.now().minus(Duration.ofMinutes(5))
        );

        if (detokenizations > 1000) {  // Threshold based on baseline
            alertService.send("Unusual detokenization volume: " + detokenizations);
        }

        // 3. Detect access from unexpected IPs
        List<String> allowedIps = List.of("10.0.1.0/24", "10.0.2.0/24");
        auditRepository.findByActionAndTimestampAfter("DETOKENIZE_PAN", Instant.now().minus(Duration.ofMinutes(5)))
            .forEach(event -> {
                if (!isIpAllowed(event.getSourceIp(), allowedIps)) {
                    alertService.send("Cardholder data accessed from unexpected IP: " + event.getSourceIp());
                }
            });
    }
}
```

**Key Interview Points**:
- **Tokenization reduces PCI scope** (Kafka cluster outside PCI boundary)
- **TLS 1.3 + mTLS for in-transit encryption** (PCI-DSS Requirement 4)
- **Disk encryption for at-rest protection** (LUKS, HSM-backed keys)
- **Role-based ACLs** enforce least privilege (PCI-DSS Requirement 7)
- **Immutable audit trail** with 7-year retention (PCI-DSS Requirement 10)
- **Key rotation** and version management (PCI-DSS Requirement 3.6)

---

## Summary

This master guide covers:
1. **System Design**: Payment processing, fraud detection (with architecture, code, trade-offs)
2. **Troubleshooting**: Consumer lag, under-replicated partitions (systematic workflows)
3. **Trade-offs**: Exactly-once vs at-least-once, compaction vs retention (decision frameworks)
4. **Pitfalls**: Unbounded state stores, partition hotspots (root causes and fixes)
5. **Behavioral**: Production outages, architectural decisions (STAR framework)
6. **Banking**: PCI-DSS compliance (tokenization, encryption, audit)

**Interview Preparation Tips**:
- **Practice system design** on whiteboard (time-boxed: 45 minutes)
- **Memorize key metrics** (throughput, latency, replication factor trade-offs)
- **Know troubleshooting workflows** (assess → investigate → mitigate → prevent)
- **Prepare 3-4 STAR stories** (outages, architectural decisions, team collaboration)
- **Emphasize banking context** (compliance, audit trails, exactly-once for financial data)

**Final Checklist for Interviews**:
- [ ] Can explain Kafka internals (replication, ISR, controller, log segments)
- [ ] Can design a production system end-to-end (topics, partitions, consumers, monitoring)
- [ ] Can troubleshoot common issues (lag, under-replicated partitions, rebalances)
- [ ] Can discuss trade-offs (consistency vs availability, throughput vs latency)
- [ ] Can demonstrate operational experience (monitoring, alerting, incident response)
- [ ] Can relate to banking domain (PCI-DSS, SOX, audit trails, exactly-once)

**Good luck with your interviews!**
