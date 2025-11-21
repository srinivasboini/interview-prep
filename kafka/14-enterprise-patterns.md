# Kafka Enterprise Patterns

## Overview

Enterprise Kafka deployments require sophisticated architectural patterns to meet stringent requirements for high availability, disaster recovery, data governance, and operational excellence. These patterns address real-world challenges in large-scale organizations: multi-datacenter replication, zero-downtime deployments, regulatory compliance, and cost optimization.

**Why This Matters for Interviews**: Staff and principal engineering roles require understanding of enterprise architecture patterns, disaster recovery strategies, and operating Kafka at scale. Expect questions about multi-region deployments, blue-green strategies, data governance, and meeting business continuity requirements (RPO/RTO).

**Real-World Banking Context**: Financial institutions require 99.99% availability (< 1 hour downtime per year), disaster recovery capabilities (survive datacenter loss), regulatory compliance (data residency, audit trails), and cost optimization (manage petabytes of data). Enterprise patterns enable meeting these requirements while maintaining operational simplicity.

---

## Multi-Datacenter Replication

### MirrorMaker 2.0 Architecture

**MirrorMaker 2.0** (MM2) replicates data between Kafka clusters (cross-datacenter, cross-region).

```
Source Cluster (us-east-1)          Target Cluster (us-west-2)
┌─────────────────────────┐         ┌──────────────────────────┐
│  Topic: payments        │         │  Topic: us-east-1.payments│
│  ├─ Partition 0         │────────→│  ├─ Partition 0          │
│  ├─ Partition 1         │  Mirror │  ├─ Partition 1          │
│  └─ Partition 2         │  Maker  │  └─ Partition 2          │
│                         │   2.0   │                          │
│  Consumer Offsets       │────────→│  Consumer Offsets        │
│  ACLs                   │────────→│  ACLs                    │
└─────────────────────────┘         └──────────────────────────┘
```

**Key Features**:
- **Active-Active**: Replicate in both directions
- **Offset Translation**: Consumer offsets replicated and translated
- **ACL Sync**: Security policies replicated
- **Topic Renaming**: Prefix target topics (us-east-1.payments)

**Configuration**:

```properties
# MirrorMaker 2.0 Configuration (mm2.properties)

# Clusters
clusters = us-east-1, us-west-2

us-east-1.bootstrap.servers = kafka-east:9092
us-west-2.bootstrap.servers = kafka-west:9092

# Replication flows
us-east-1->us-west-2.enabled = true
us-west-2->us-east-1.enabled = false  # One-way replication

# Topics to replicate (regex)
us-east-1->us-west-2.topics = payments.*, transactions.*

# Topic rename (prefix with source cluster)
replication.policy.class = org.apache.kafka.connect.mirror.DefaultReplicationPolicy

# Consumer group offset sync
us-east-1->us-west-2.sync.group.offsets.enabled = true
us-east-1->us-west-2.sync.group.offsets.interval.seconds = 60

# Performance tuning
tasks.max = 10
replication.factor = 3
```

**Start MirrorMaker 2.0**:
```bash
# Dedicated MirrorMaker cluster (Kafka Connect workers)
connect-mirror-maker.sh mm2.properties
```

### Replication Patterns

**Pattern 1: Active-Passive (Disaster Recovery)**

```
┌─────────────────────────────────────────────────────────────┐
│              Active-Passive Replication                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Primary Region (us-east-1): ACTIVE                         │
│    - Producers write to us-east-1                           │
│    - Consumers read from us-east-1                          │
│    - RPO: 1 minute (replication lag)                        │
│                                                              │
│  Secondary Region (us-west-2): PASSIVE                      │
│    - MirrorMaker replicates us-east-1 → us-west-2          │
│    - No producers/consumers (standby)                       │
│    - RTO: 5 minutes (manual failover)                      │
│                                                              │
│  Failover Process:                                          │
│    1. Detect us-east-1 failure                              │
│    2. Stop producers/consumers pointing to us-east-1        │
│    3. Reconfigure to us-west-2                              │
│    4. Resume producers/consumers                            │
│    5. Start reverse replication (us-west-2 → us-east-1)    │
└─────────────────────────────────────────────────────────────┘
```

**Use Case**: Cost-effective DR (secondary cluster idle)

**Pattern 2: Active-Active (Geo-Distributed)**

```
┌─────────────────────────────────────────────────────────────┐
│              Active-Active Replication                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Region A (us-east-1): ACTIVE                               │
│    - Producers: US customers                                │
│    - Consumers: Regional processing                         │
│    - Replicate A → B (async)                                │
│                                                              │
│  Region B (eu-central-1): ACTIVE                            │
│    - Producers: EU customers                                │
│    - Consumers: Regional processing                         │
│    - Replicate B → A (async)                                │
│                                                              │
│  Conflict Resolution:                                        │
│    - Use unique keys (region prefix)                        │
│    - Last-write-wins (timestamp)                            │
│    - Application-level reconciliation                       │
└─────────────────────────────────────────────────────────────┘
```

**Use Case**: Low-latency global access, data residency (GDPR)

**Pattern 3: Hub-and-Spoke (Aggregation)**

```
┌─────────────────────────────────────────────────────────────┐
│              Hub-and-Spoke Replication                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Regional Clusters (Spokes):                                │
│    - us-east-1: Branch transactions                         │
│    - us-west-2: Branch transactions                         │
│    - eu-central-1: Branch transactions                      │
│    - ap-southeast-1: Branch transactions                    │
│                                                              │
│  Central Cluster (Hub):                                     │
│    - Aggregates all regional data                           │
│    - Global analytics, reporting                            │
│    - Replicate: All spokes → Hub                           │
│                                                              │
│  Benefits:                                                   │
│    - Regional latency (local producers/consumers)           │
│    - Centralized analytics (hub)                            │
│    - Scalable (add spokes independently)                    │
└─────────────────────────────────────────────────────────────┘
```

**Use Case**: Global bank with regional branches (aggregate to HQ)

### Offset Translation

**Challenge**: Consumer offsets differ between source and target clusters

**MirrorMaker 2.0 Solution**: Automatic offset translation

```java
// Consumer in us-east-1
KafkaConsumer<String, Payment> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("payments"));

// Process messages
while (true) {
    ConsumerRecords<String, Payment> records = consumer.poll(Duration.ofMillis(100));
    // Process...
    consumer.commitSync();  // Offset: 10,000
}

// Failover to us-west-2
// MirrorMaker syncs offsets to us-west-2.checkpoints.internal topic
// - Source topic: payments, offset: 10,000
// - Target topic: us-east-1.payments, translated offset: 9,950

// Resume in us-west-2
props.put("bootstrap.servers", "kafka-west:9092");
KafkaConsumer<String, Payment> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("us-east-1.payments"));

// MirrorMaker translates offset automatically
// Consumer resumes from translated offset 9,950 (minimal reprocessing)
```

---

## Disaster Recovery Strategies

### RPO and RTO Requirements

**RPO (Recovery Point Objective)**: Maximum acceptable data loss
**RTO (Recovery Time Objective)**: Maximum acceptable downtime

**Banking Requirements**:
```
Payment Processing:
- RPO: 1 minute (max 60 seconds of transactions lost)
- RTO: 5 minutes (max 5 minutes downtime)

Audit Logs:
- RPO: 0 (zero data loss, compliance)
- RTO: 1 hour (acceptable for audit queries)

Analytics:
- RPO: 1 hour (re-run batch jobs)
- RTO: 4 hours (non-critical)
```

### DR Strategy: Active-Passive with Fast Failover

```bash
# Primary Region (us-east-1): Active
# - Producers write to payments topic
# - Consumers read from payments topic
# - MirrorMaker replicates to us-west-2

# Secondary Region (us-west-2): Passive
# - MirrorMaker receives replicated data
# - No active producers/consumers (standby)

# Failover Automation (detect primary failure)
# 1. Health check fails in us-east-1
if ! curl -f http://kafka-east:9092/health; then
  echo "Primary cluster unhealthy, initiating failover"

  # 2. Update DNS (point to us-west-2)
  aws route53 change-resource-record-sets --hosted-zone-id Z123 \
    --change-batch '{
      "Changes": [{
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "kafka.example.com",
          "Type": "CNAME",
          "TTL": 60,
          "ResourceRecords": [{"Value": "kafka-west.example.com"}]
        }
      }]
    }'

  # 3. Notify applications (restart consumers)
  kubectl rollout restart deployment/payment-consumer

  # 4. Start reverse replication (us-west-2 → us-east-1)
  # Prepare for failback when us-east-1 recovers

  echo "Failover complete, RTO: 5 minutes"
fi

# RPO: 1 minute (replication lag)
# RTO: 5 minutes (DNS propagation + consumer restart)
```

### Stretch Clusters (Low Latency)

**Stretch Cluster**: Single Kafka cluster across multiple AZs (same region)

```
┌─────────────────────────────────────────────────────────────┐
│              Stretch Cluster Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Single Kafka Cluster (3 AZs in us-east-1):                │
│    - AZ-A: Broker 1, 2                                      │
│    - AZ-B: Broker 3, 4                                      │
│    - AZ-C: Broker 5, 6                                      │
│                                                              │
│  Partition Replicas (RF=3):                                 │
│    - Partition 0: Broker1 (AZ-A), Broker3 (AZ-B), Broker5 (AZ-C)│
│    - Ensures each partition has replica in each AZ          │
│                                                              │
│  Benefits:                                                   │
│    - RPO: 0 (synchronous replication, no data loss)        │
│    - RTO: 0 (automatic failover, no downtime)              │
│    - Latency: <5ms (same region)                           │
│                                                              │
│  Trade-off:                                                  │
│    - Network latency: +3ms (cross-AZ replication)          │
│    - Network cost: $0.01/GB cross-AZ                        │
└─────────────────────────────────────────────────────────────┘
```

**Configuration**:
```properties
# Enable rack awareness
broker.rack = az-a  # Broker 1, 2
broker.rack = az-b  # Broker 3, 4
broker.rack = az-c  # Broker 5, 6

# Topic configuration
replication.factor = 3
min.insync.replicas = 2

# Result: Survives AZ failure with zero downtime/data loss
```

**Use Case**: Payment processing (zero RPO/RTO required, acceptable 3ms latency)

---

## Blue-Green Deployments

**Blue-Green** enables zero-downtime Kafka upgrades/migrations.

### Pattern: Parallel Clusters

```
┌─────────────────────────────────────────────────────────────┐
│              Blue-Green Deployment                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Phase 1: Blue Cluster (Production)                         │
│    - Producers → Blue                                       │
│    - Consumers → Blue                                       │
│    - Version: Kafka 3.0                                     │
│                                                              │
│  Phase 2: Green Cluster (New Version)                       │
│    - Deploy Green cluster (Kafka 3.5)                       │
│    - MirrorMaker: Blue → Green                              │
│    - Validate: Green cluster healthy                        │
│                                                              │
│  Phase 3: Dual-Write                                        │
│    - Producers → Blue + Green (both)                        │
│    - Consumers → Blue (unchanged)                           │
│    - Duration: 24 hours (ensure data in both clusters)     │
│                                                              │
│  Phase 4: Consumer Migration                                │
│    - Migrate consumers: Blue → Green (gradual rollout)     │
│    - Monitor: Consumer lag, errors                          │
│    - Rollback available: Point consumers back to Blue       │
│                                                              │
│  Phase 5: Producer Migration                                │
│    - Stop dual-write, producers → Green only                │
│    - Blue cluster deprecated                                │
│                                                              │
│  Phase 6: Decommission Blue                                 │
│    - After 7 days (retention), delete Blue cluster          │
└─────────────────────────────────────────────────────────────┘
```

**Implementation**:

```java
/**
 * Dual-write producer (writes to both Blue and Green clusters)
 */
public class DualWriteProducer {

    private KafkaProducer<String, Payment> blueProducer;
    private KafkaProducer<String, Payment> greenProducer;

    public void send(Payment payment) {
        ProducerRecord<String, Payment> record =
            new ProducerRecord<>("payments", payment.getId(), payment);

        // Write to Blue (production)
        blueProducer.send(record, (metadata, exception) -> {
            if (exception != null) {
                logger.error("Blue write failed", exception);
            }
        });

        // Write to Green (new cluster)
        greenProducer.send(record, (metadata, exception) -> {
            if (exception != null) {
                logger.error("Green write failed", exception);
                // Don't fail request (Green is not critical yet)
            }
        });
    }
}

/**
 * Feature flag-based consumer (switch clusters via config)
 */
public class PaymentConsumer {

    public void start() {
        String clusterEndpoint = config.get("kafka.cluster");  // Blue or Green

        Properties props = new Properties();
        props.put("bootstrap.servers", clusterEndpoint);

        KafkaConsumer<String, Payment> consumer = new KafkaConsumer<>(props);
        // ... consume messages
    }
}

// Deployment:
// - Deploy new version with cluster=Green
// - Monitor for 24 hours
// - Rollback: Deploy with cluster=Blue if issues
```

**Benefits**:
- Zero downtime (gradual migration)
- Fast rollback (repoint to Blue)
- Validation (test Green before full cutover)

---

## Tiered Storage

**Tiered Storage** offloads old segments to object storage (S3, Azure Blob) to reduce local disk costs.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Tiered Storage Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Kafka Broker:                                              │
│    - Local Disk (Hot Tier): Last 7 days                    │
│      - Fast access (SSD)                                    │
│      - Recent data (consumers actively reading)            │
│                                                              │
│    - Object Storage (Cold Tier): 7 days - 7 years          │
│      - S3/Azure Blob (cheap)                                │
│      - Historical data (infrequent access)                  │
│                                                              │
│  Tiering Process:                                           │
│    1. Segment closes (after 7 days or 1 GB)                │
│    2. Segment uploaded to S3 (async)                        │
│    3. Local segment deleted (after upload confirmed)        │
│    4. Metadata retained (segment location in S3)            │
│                                                              │
│  Consumer Read:                                             │
│    - Recent data: Read from local disk (fast)               │
│    - Historical data: Fetch from S3 (slower)                │
│                                                              │
│  Cost Savings:                                              │
│    - Local: 1 TB SSD = $100/month                          │
│    - S3: 100 TB = $2,300/month (vs $10,000 SSD)           │
│    - Savings: 77% for long retention                        │
└─────────────────────────────────────────────────────────────┘
```

**Configuration**:

```properties
# Enable tiered storage (Kafka 3.6+)
remote.log.storage.system.enable = true

# S3 configuration
remote.log.storage.manager.class.name = org.apache.kafka.server.log.remote.storage.RemoteLogStorageManager

# Local retention (hot tier)
local.retention.ms = 604800000  # 7 days local

# Total retention (hot + cold)
retention.ms = 220752000000  # 7 years total

# Tiering interval
remote.log.manager.task.interval.ms = 60000  # Check every minute
```

**Banking Use Case**:

```
Payment Topic (1 TB/day ingestion):
- Hot Tier (7 days): 7 TB local SSD
- Cold Tier (7 years): 2,555 TB S3

Cost Breakdown:
- SSD (7 days): 7 TB × $100/TB = $700/month
- S3 Standard (90 days): 90 TB × $23/TB = $2,070/month
- S3 Glacier (6.75 years): 2,465 TB × $4/TB = $9,860/month

Total: $12,630/month (vs. $255,500 all-SSD, 95% savings!)
```

---

## Data Governance

### Schema Governance

**Centralized Schema Management** (covered in Schema Registry chapter):
- Schema approval workflow (Git PR + review)
- Compatibility enforcement (FULL mode)
- Deprecation policy (90-day notice)
- Breaking change process (dual-write migration)

### Topic Governance

**Topic Naming Convention**:
```
Pattern: <domain>.<entity>.<event-type>

Examples:
- payments.transactions.created
- payments.transactions.settled
- fraud.alerts.high-risk
- customer.profiles.updated

Benefits:
- Clear ownership (domain team)
- Easy discovery (search by domain)
- ACL management (grant by domain)
```

**Topic Lifecycle**:
```
1. Request: Team submits topic request (JIRA ticket)
2. Review: Platform team reviews (naming, retention, config)
3. Approval: Approved topics created via Terraform
4. Monitoring: Usage metrics tracked (producers, consumers)
5. Deprecation: Unused topics deleted after 90 days (grace period)
```

**Topic Configuration Standards**:
```yaml
# Standard topic configurations (Terraform)
module "payment_topic" {
  source = "./kafka-topic"

  topic_name = "payments.transactions.created"
  partitions = 30
  replication_factor = 3
  min_insync_replicas = 2

  # Standard configs
  retention_ms = 604800000  # 7 days (standard)
  compression_type = "lz4"
  cleanup_policy = "delete"

  # Tags for cost allocation
  tags = {
    domain = "payments"
    criticality = "high"
    compliance = "pci-dss"
  }
}
```

### Data Lineage

**Track Data Flow** (producer → topic → consumer):

```yaml
# Data lineage metadata
lineage:
  payments.transactions.created:
    producers:
      - payment-gateway-service (v2.3.0)
      - mobile-app-backend (v1.5.0)
    consumers:
      - fraud-detection-service (v3.1.0)
      - analytics-pipeline (v2.0.0)
      - audit-log-archiver (v1.2.0)
    dependencies:
      - customer.profiles.updated (join)
      - accounts.balances (lookup)
```

**Benefits**:
- Impact analysis (which consumers affected by schema change?)
- Troubleshooting (trace message flow)
- Compliance (audit data access)

---

## Cost Optimization

### Storage Cost Reduction

**1. Compression** (lz4, zstd):
```properties
compression.type = lz4
# Reduces storage by 60-70% (JSON messages)
# Cost: 1 PB uncompressed → 300 TB compressed (70% savings)
```

**2. Log Compaction** (changelog topics):
```properties
cleanup.policy = compact
# Retains only latest value per key
# Example: 1M customer updates → 100K unique customers (90% reduction)
```

**3. Tiered Storage** (S3 for old data):
```properties
local.retention.ms = 604800000  # 7 days local
retention.ms = 220752000000     # 7 years total (S3)
# Cost: 95% savings vs. all-SSD
```

**4. Retention Tuning** (delete old data):
```properties
# Reduce retention for non-critical topics
retention.ms = 86400000  # 1 day (was 7 days)
# Savings: 86% reduction in storage
```

### Compute Cost Reduction

**1. Right-Sizing** (match broker size to load):
```bash
# Before: 10 × r5.4xlarge (16 vCPU, 128 GB RAM) = $12,000/month
# After: 10 × r5.2xlarge (8 vCPU, 64 GB RAM) = $6,000/month
# Savings: 50% (if CPU/memory usage < 50%)
```

**2. Spot Instances** (for non-critical consumers):
```yaml
# Kubernetes deployment (spot instances)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics-consumer
spec:
  template:
    spec:
      nodeSelector:
        node-type: spot  # 70% cheaper than on-demand
```

**3. Auto-Scaling** (scale consumers based on lag):
```yaml
# Horizontal Pod Autoscaler (Kubernetes)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-consumer
spec:
  minReplicas: 10
  maxReplicas: 50
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
      target:
        type: Value
        value: "10000"  # Scale up if lag > 10K

# Savings: 50% (average 25 pods vs. static 50 pods)
```

---

## Interview Questions

### Question 1: Design a multi-region Kafka architecture for a global bank that must meet data residency requirements (GDPR) while providing low-latency access worldwide.

**Answer**:

**Architecture: Active-Active with Regional Isolation**

```
┌─────────────────────────────────────────────────────────────┐
│          Multi-Region Architecture (Data Residency)          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Region: EU (Frankfurt)                                     │
│    - Kafka Cluster: eu-central-1                            │
│    - Producers: EU customers only                           │
│    - Consumers: EU services                                 │
│    - Data: EU customer transactions (GDPR compliant)        │
│    - Replication: None (data stays in EU)                  │
│                                                              │
│  Region: US (Virginia)                                      │
│    - Kafka Cluster: us-east-1                               │
│    - Producers: US customers only                           │
│    - Consumers: US services                                 │
│    - Data: US customer transactions                         │
│    - Replication: None (data stays in US)                  │
│                                                              │
│  Global Analytics (Aggregated, Anonymized):                 │
│    - Hub Cluster: Global analytics (anonymized data)       │
│    - Replication: EU → Hub, US → Hub                       │
│    - Data: Anonymized aggregates (no PII)                  │
│                                                              │
│  Benefits:                                                   │
│    - GDPR Compliance: EU data never leaves EU               │
│    - Low Latency: Regional clusters (< 10ms)               │
│    - Global Analytics: Aggregated in hub (no PII)          │
└─────────────────────────────────────────────────────────────┘
```

**Configuration**:
```properties
# EU Cluster: No replication outside EU
# - Topics: payments.eu.*
# - Retention: 7 years (GDPR requirement)

# US Cluster: No replication outside US
# - Topics: payments.us.*

# MirrorMaker 2.0: Anonymize before replication
# - Source: payments.eu.transactions
# - Transform: Remove PII (name, email, address)
# - Target: global-analytics.transactions.anonymized
```

**Result**: GDPR compliance + global analytics

---

### Question 2: How would you perform a zero-downtime migration from Kafka 2.8 to 3.5 in a production environment processing 100,000 messages/sec?

**Answer**:

**Blue-Green Migration Strategy**:

**Phase 1: Deploy Green Cluster (Kafka 3.5)**
```bash
# Deploy new Kafka 3.5 cluster (parallel to existing)
# - Same partition count
# - Same replication factor
# - MirrorMaker: Blue (2.8) → Green (3.5)
```

**Phase 2: Dual-Write (Producers)**
```java
// Update producers to write to both clusters
producer.send(recordBlue, callbackBlue);
producer.send(recordGreen, callbackGreen);
// Duration: 48 hours (ensure both clusters have data)
```

**Phase 3: Consumer Migration (Gradual)**
```bash
# Week 1: Migrate 10% of consumers to Green
kubectl set env deployment/payment-consumer KAFKA_CLUSTER=green --replicas=3

# Monitor lag, errors
# If issues: Rollback (kubectl set env KAFKA_CLUSTER=blue)

# Week 2: Migrate 50% to Green
# Week 3: Migrate 100% to Green
```

**Phase 4: Producer Migration**
```bash
# Stop dual-write, producers → Green only
# Blue cluster receives no new data
```

**Phase 5: Decommission Blue**
```bash
# After retention period (7 days), delete Blue cluster
```

**Rollback Plan**: Repoint consumers to Blue (< 5 minutes)

**Total Downtime**: 0 seconds

---

### Question 3: Design a cost-effective storage strategy for a Kafka cluster that must retain 5 PB of data for 7 years while keeping recent data (30 days) fast.

**Answer**:

**Tiered Storage Strategy**:

```
┌─────────────────────────────────────────────────────────────┐
│          Tiered Storage (5 PB, 7-year retention)            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Tier 1: Local NVMe SSD (0-7 days)                         │
│    - Storage: 50 TB (7 days × 7 TB/day)                    │
│    - Latency: 1ms                                           │
│    - Cost: 50 TB × $300/TB = $15,000/month                 │
│                                                              │
│  Tier 2: Local SATA HDD (7-30 days)                        │
│    - Storage: 161 TB (23 days × 7 TB/day)                  │
│    - Latency: 10ms                                          │
│    - Cost: 161 TB × $30/TB = $4,830/month                  │
│                                                              │
│  Tier 3: S3 Standard (30-90 days)                          │
│    - Storage: 420 TB (60 days × 7 TB/day)                  │
│    - Latency: 100ms                                         │
│    - Cost: 420 TB × $23/TB = $9,660/month                  │
│                                                              │
│  Tier 4: S3 Glacier Deep Archive (90 days - 7 years)      │
│    - Storage: 4,369 TB (6.75 years × 7 TB/day)             │
│    - Latency: 12 hours (retrieval)                         │
│    - Cost: 4,369 TB × $1/TB = $4,369/month                 │
│                                                              │
│  Total Cost: $33,859/month (vs. $1,500,000 all-SSD)       │
│  Savings: 97.7%                                             │
└─────────────────────────────────────────────────────────────┘
```

**Configuration**:
```properties
# Kafka Tiered Storage
local.retention.ms = 604800000      # 7 days on SSD
local.retention.bytes = 50000000000000  # 50 TB

retention.ms = 220752000000         # 7 years total
```

**Result**: $33K/month (97% cheaper than all-SSD)

---

## Summary

Enterprise Kafka patterns enable meeting stringent requirements for availability, disaster recovery, compliance, and cost optimization. Key takeaways:

1. **Multi-DC Replication**: MirrorMaker 2.0 (active-passive, active-active, hub-and-spoke)
2. **Disaster Recovery**: RPO/RTO targets (stretch clusters for zero RPO/RTO)
3. **Blue-Green**: Zero-downtime migrations (dual-write + gradual consumer migration)
4. **Tiered Storage**: 95% cost savings (S3 for old data)
5. **Governance**: Schema management, topic standards, data lineage
6. **Cost Optimization**: Compression, compaction, tiered storage, right-sizing

In banking systems, these patterns enable global operations, regulatory compliance, and cost-effective operations at petabyte scale.

**Word Count**: ~5,000 words
