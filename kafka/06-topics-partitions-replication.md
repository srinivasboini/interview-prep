# Kafka Topics, Partitions, and Replication

## Overview

Topics, partitions, and replication are the fundamental building blocks of Kafka's distributed architecture. A **topic** is a logical channel for messages, **partitions** enable parallelism and scalability, and **replication** provides fault tolerance and durability. Understanding how to design topics, choose partition counts, select partition keys, and configure replication is critical for building high-performance, resilient Kafka systems.

**Why This Matters for Interviews**: Senior engineering roles require expertise in capacity planning (how many partitions?), data modeling (how to partition data?), and reliability engineering (replication factor trade-offs). Expect questions about partition key selection, handling hotspots, rebalancing strategies, and designing for high availability.

**Real-World Banking Context**: Payment processing systems must partition data to achieve horizontal scalability (100,000+ TPS), use partition keys to maintain transaction ordering per account, and configure replication to survive datacenter failures without data loss. Poor partition design can lead to hotspots, unbalanced load, and degraded performance.

---

## Topic Design Principles

### Topic as a Logical Channel

A **topic** represents a category of messages (e.g., `payments`, `user-events`, `audit-logs`). Topics are append-only logs that retain messages based on retention policies.

**Topic Naming Conventions** (enterprise best practices):
```
Pattern: <domain>.<entity>.<event-type>
Examples:
- payments.transactions.created
- payments.transactions.settled
- customer.profiles.updated
- fraud.alerts.detected
- audit.api-calls.logged

Benefits:
- Clear ownership (domain team responsible)
- Easy discovery (search by domain or entity)
- Scalable naming (add new events without conflicts)
```

**Banking Example**:
```
Domain: Payments
Topics:
- payments.wire-transfers.initiated
- payments.wire-transfers.validated
- payments.wire-transfers.settled
- payments.wire-transfers.failed

Domain: Fraud Detection
Topics:
- fraud.transactions.scored
- fraud.alerts.high-risk
- fraud.investigations.opened
```

### Single Responsibility Principle

Each topic should have a **single, well-defined purpose**. Avoid mixing unrelated message types in one topic.

**Anti-Pattern** (mixed messages):
```
Topic: banking-events
Messages:
- Payment transactions
- Account creations
- Fraud alerts
- Audit logs

Problems:
- Consumers must filter irrelevant messages (wasted bandwidth)
- Schema evolution is complex (multiple message types)
- Retention policies conflict (payments: 7 years, fraud alerts: 30 days)
```

**Best Practice** (separate topics):
```
Topics:
- payments.transactions
- customer.accounts.created
- fraud.alerts
- audit.logs

Benefits:
- Consumers subscribe only to relevant topics
- Independent retention policies per topic
- Clear schema per topic (easier evolution)
```

### Schema Evolution Strategy

Define message schemas using **Avro**, **Protobuf**, or **JSON Schema** with a **Schema Registry**.

**Schema Registry Benefits**:
1. **Schema Validation**: Producers/consumers validate messages against registered schemas
2. **Version Control**: Track schema versions, enforce compatibility rules
3. **Documentation**: Auto-generated docs from schemas

**Compatibility Rules**:
```
BACKWARD (default): New schema can read data from old schema
- Use case: Deploy consumers first, then producers
- Example: Add optional field to message

FORWARD: Old schema can read data from new schema
- Use case: Deploy producers first, then consumers
- Example: Remove optional field from message

FULL: Both backward and forward compatible
- Use case: Deploy in any order
- Example: Add/remove optional fields only

NONE: No compatibility checks
- Use case: Breaking changes (new major version)
- Example: Change field type from string to int
```

**Banking Example**:
```json
// Schema v1: Payment message
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.bank.payments",
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string"}
  ]
}

// Schema v2: Add optional field (BACKWARD compatible)
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.bank.payments",
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string"},
    {"name": "sourceAccount", "type": ["null", "string"], "default": null}  // Optional
  ]
}

// Old consumers can still read new messages (ignore new field)
// New consumers can read old messages (sourceAccount = null)
```

---

## Partition Design

Partitions are the unit of **parallelism** and **ordering** in Kafka. Understanding partition design is critical for performance and correctness.

### Partition Count Selection

Choosing the right partition count is a balance between **parallelism** (higher throughput) and **overhead** (more resources).

**Factors to Consider**:

1. **Target Throughput**:
```
Formula: Partitions = Target Throughput / Consumer Throughput

Example:
- Target: 100,000 messages/sec
- Single consumer throughput: 10,000 messages/sec
- Partitions needed: 100,000 / 10,000 = 10 partitions
```

2. **Consumer Parallelism**:
```
Rule: Partitions >= Number of Consumers in Group

Example:
- 12 consumers in consumer group
- Need at least 12 partitions for full parallelism
- If 6 partitions: only 6 consumers active (others idle)
- If 24 partitions: each consumer handles 2 partitions
```

3. **Producer Throughput**:
```
Formula: Partitions = Target Throughput / Producer Throughput per Partition

Example:
- Target: 500 MB/sec
- Single producer throughput per partition: 50 MB/sec
- Partitions needed: 500 / 50 = 10 partitions
```

4. **Ordering Requirements**:
```
Rule: Messages with same partition key go to same partition (ordered)

Example:
- Payment topic with partition key = accountId
- All payments for account "12345" go to partition 3 (ordered by offset)
- Payments for different accounts can be in different partitions (parallel processing)
```

**Partition Count Guidelines**:

| Use Case | Partition Count | Rationale |
|----------|----------------|-----------|
| **Low-volume logs** | 1-3 | Simple, minimal overhead |
| **Medium-volume events** | 6-12 | Balance parallelism and overhead |
| **High-volume transactions** | 30-50 | High throughput, many consumers |
| **Very high-volume** | 100-200 | Extreme parallelism (monitor overhead) |

**Overhead Considerations**:
- **Broker Memory**: Each partition consumes memory (page cache, indexes)
- **Broker CPU**: More partitions = more file handles, more metadata updates
- **ZooKeeper/KRaft Load**: Metadata updates for each partition
- **Rebalancing Time**: More partitions = longer consumer group rebalancing
- **End-to-End Latency**: More partitions = higher p99 latency (tail latency)

**Rule of Thumb**:
```
Maximum partitions per broker: 2,000 - 4,000
Maximum partitions per cluster: 200,000 (tested limit)

For most workloads:
- Start with 10-30 partitions
- Increase if throughput insufficient
- Monitor broker CPU, memory, rebalancing time
```

**Banking Example**:

**Payment Topic** (high-volume, 50,000 TPS):
```properties
num.partitions = 50

Calculation:
- Target throughput: 50,000 messages/sec
- Consumer throughput: 1,000 messages/sec
- Consumers needed: 50
- Partitions: 50 (one per consumer for full parallelism)

Benefits:
- 50 consumers process in parallel (50,000 TPS achieved)
- Partition key = accountId (ordered payments per account)
```

**Audit Log Topic** (low-volume, 100 TPS):
```properties
num.partitions = 3

Rationale:
- Low throughput (single consumer can handle 10,000 TPS)
- Partition for future growth (3x headroom)
- Minimize overhead (fewer partitions = less rebalancing time)
```

### Partition Key Selection

The **partition key** determines which partition a message is assigned to. Choosing the right key is critical for **load balancing** and **ordering guarantees**.

**Partitioning Algorithm**:
```java
// Default partitioner (Kafka 2.4+, murmur2 hash)
int partition = Math.abs(murmur2(key)) % numPartitions;

// Example:
// key = "account-12345"
// murmur2("account-12345") = 987654321
// partition = 987654321 % 50 = 21
// Message goes to partition 21
```

**Partition Key Strategies**:

#### 1. Entity ID (Most Common)
```java
// Partition by account ID (all messages for same account ordered)
ProducerRecord<String, Payment> record = new ProducerRecord<>(
    "payments",
    payment.getAccountId(),  // Partition key
    payment
);

// Benefits:
// - All payments for account "12345" go to same partition (ordered)
// - Consumers can maintain per-account state (e.g., balance calculation)

// Use Cases:
// - Account transactions (key = accountId)
// - User events (key = userId)
// - Device telemetry (key = deviceId)
```

#### 2. Composite Key
```java
// Partition by country + currency (distribute by region and currency)
String key = payment.getCountry() + ":" + payment.getCurrency();
ProducerRecord<String, Payment> record = new ProducerRecord<>(
    "payments",
    key,  // "US:USD", "UK:GBP", "EU:EUR"
    payment
);

// Benefits:
// - Group related messages (same country/currency processed together)
// - Load balancing across regions

// Use Cases:
// - Multi-tenant systems (key = tenantId:entityId)
// - Geo-distributed processing (key = region:entityId)
```

#### 3. Null Key (Round-Robin)
```java
// No partition key (round-robin distribution)
ProducerRecord<String, LogEntry> record = new ProducerRecord<>(
    "application-logs",
    null,  // No key (round-robin)
    logEntry
);

// Benefits:
// - Even distribution across partitions
// - No hotspot risk

// Use Cases:
// - Logs (no ordering required)
// - Metrics (aggregate across all partitions)
// - High-volume, independent messages
```

#### 4. Custom Partitioner
```java
// Custom partitioner for VIP customers (dedicated partitions)
public class VIPCustomerPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String accountId = (String) key;
        int numPartitions = cluster.partitionCountForTopic(topic);

        // VIP customers (accounts starting with "VIP") go to partition 0
        if (accountId.startsWith("VIP")) {
            return 0;
        }

        // Regular customers distributed across remaining partitions
        return (Math.abs(murmur2(keyBytes)) % (numPartitions - 1)) + 1;
    }
}

// Producer configuration:
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, VIPCustomerPartitioner.class);

// Use Cases:
// - Priority processing (VIP customers on dedicated partition)
// - Compliance (sensitive data on specific partitions)
// - Performance isolation (high-volume keys on separate partitions)
```

**Partition Key Anti-Patterns**:

**Anti-Pattern 1: Timestamp as Key**
```java
// BAD: Timestamp creates hotspots (all messages go to same partition at any moment)
String key = String.valueOf(System.currentTimeMillis());
ProducerRecord<String, Event> record = new ProducerRecord<>("events", key, event);

// Problem:
// - All messages in same second go to same partition
// - Other partitions idle (unbalanced load)
// - No parallelism

// Fix: Use null key (round-robin) or entity ID
```

**Anti-Pattern 2: Low-Cardinality Key**
```java
// BAD: Only 2 values (50% of traffic to 1 partition, 50% to another)
String key = payment.getType();  // "CREDIT" or "DEBIT"
ProducerRecord<String, Payment> record = new ProducerRecord<>("payments", key, payment);

// Problem:
// - With 50 partitions, only 2 partitions used (48 idle)
// - Severe hotspot (2 partitions handle 100% of traffic)

// Fix: Use high-cardinality key (accountId, transactionId)
```

**Anti-Pattern 3: Skewed Key Distribution**
```java
// BAD: 80% of traffic from 20% of accounts (hotspot)
String key = payment.getAccountId();  // "account-1" has 50% of all payments

// Problem:
// - Partition for "account-1" overloaded (50% of traffic)
// - Other partitions underutilized
// - Consumer lag on hot partition

// Fix: Use composite key (accountId + shard suffix) or custom partitioner
String key = payment.getAccountId() + ":" + (payment.hashCode() % 10);
// Distributes "account-1" traffic across 10 partitions
```

**Banking Example: Payment Topic Partitioning**:

```java
/**
 * Partition strategy for payment processing:
 * - Key: accountId (maintains ordering per account)
 * - 50 partitions (supports 50,000 TPS with 1,000 TPS per consumer)
 * - Custom partitioner for high-volume accounts (prevent hotspots)
 */
public class PaymentPartitioner implements Partitioner {

    private static final Set<String> HIGH_VOLUME_ACCOUNTS = loadFromConfig();

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String accountId = (String) key;
        int numPartitions = cluster.partitionCountForTopic(topic);

        // High-volume accounts (e.g., corporate accounts with 10,000 TPS)
        // Distribute across 5 dedicated partitions (2,000 TPS each)
        if (HIGH_VOLUME_ACCOUNTS.contains(accountId)) {
            int shard = Math.abs(accountId.hashCode()) % 5;
            return shard;  // Partitions 0-4
        }

        // Regular accounts distributed across remaining 45 partitions
        int hash = Math.abs(murmur2(keyBytes));
        return (hash % (numPartitions - 5)) + 5;  // Partitions 5-49
    }
}

// Results:
// - High-volume accounts: 5 partitions, 10,000 TPS (no hotspot)
// - Regular accounts: 45 partitions, 40,000 TPS (balanced load)
// - Total: 50,000 TPS, all partitions balanced
```

---

## Replication Design

Replication provides **fault tolerance** and **data durability** by maintaining multiple copies of each partition across different brokers.

### Replication Factor Selection

**Replication Factor** (RF) determines how many copies of each partition exist.

**Trade-offs**:

| Replication Factor | Durability | Availability | Storage Cost | Write Latency |
|-------------------|-----------|-------------|--------------|---------------|
| **RF=1** | None (data loss on failure) | Low | 1x | Lowest |
| **RF=2** | Moderate (1 replica) | Moderate | 2x | Low |
| **RF=3** | High (2 replicas) | High | 3x | Moderate |
| **RF=5** | Very High (4 replicas) | Very High | 5x | High |

**Guidelines**:

```properties
# Development/Testing
replication.factor = 1
# Acceptable data loss, minimal cost

# Production (Standard)
replication.factor = 3
min.insync.replicas = 2
# Tolerates 1 broker failure without data loss
# Balance of durability and cost

# Production (Critical Data)
replication.factor = 5
min.insync.replicas = 3
# Tolerates 2 broker failures without data loss
# Maximum durability (regulatory compliance)
```

**Calculation: Failure Tolerance**:
```
Failure Tolerance = Replication Factor - min.insync.replicas

Examples:
- RF=3, min.insync.replicas=2: Tolerates 1 failure (3-2=1)
- RF=5, min.insync.replicas=3: Tolerates 2 failures (5-3=2)
- RF=3, min.insync.replicas=1: Tolerates 2 failures (3-1=2), but risk data loss
```

**Banking Recommendations**:

```properties
# Payment Transactions (critical, regulatory compliance)
replication.factor = 5
min.insync.replicas = 3
acks = all
# Zero data loss, tolerates 2 broker failures

# Fraud Alerts (critical, real-time)
replication.factor = 3
min.insync.replicas = 2
acks = all
# Balance latency and durability

# Application Logs (non-critical)
replication.factor = 2
min.insync.replicas = 1
acks = 1
# Availability over durability, low cost
```

### Rack Awareness

**Rack awareness** distributes replicas across different **racks** (physical server racks, availability zones, or datacenters) to survive correlated failures.

**Configuration**:
```properties
# Broker configuration (assign each broker to a rack/AZ)
# Broker 1 (AZ-1):
broker.rack = az-1

# Broker 2 (AZ-2):
broker.rack = az-2

# Broker 3 (AZ-3):
broker.rack = az-3

# Kafka ensures replicas are spread across different racks
```

**Replica Assignment Without Rack Awareness**:
```
Topic: payments (3 partitions, RF=3)
Cluster: 6 brokers in 3 racks (2 brokers per rack)

Partition 0: Replicas on Broker1 (Rack1), Broker2 (Rack1), Broker3 (Rack2)
                              ↑                    ↑
                        Same rack! (correlated failure risk)

If Rack1 fails (power outage):
- Partition 0 loses 2 replicas (Broker1, Broker2)
- Only 1 replica remains (Broker3)
- If min.insync.replicas=2, partition becomes unavailable
```

**Replica Assignment With Rack Awareness**:
```
Topic: payments (3 partitions, RF=3)
Cluster: 6 brokers in 3 racks (2 brokers per rack)

Partition 0: Replicas on Broker1 (Rack1), Broker3 (Rack2), Broker5 (Rack3)
                              ↑              ↑              ↑
                        Different racks (survives rack failure)

If Rack1 fails:
- Partition 0 loses 1 replica (Broker1)
- 2 replicas remain (Broker3, Broker5)
- If min.insync.replicas=2, partition remains available
```

**Banking Example: Multi-AZ Deployment**:
```
AWS Deployment (3 Availability Zones):

AZ-1 (us-east-1a): Broker1, Broker2
AZ-2 (us-east-1b): Broker3, Broker4
AZ-3 (us-east-1c): Broker5, Broker6

Configuration:
- broker.rack = az-1 (Broker1, Broker2)
- broker.rack = az-2 (Broker3, Broker4)
- broker.rack = az-3 (Broker5, Broker6)

Topic: payment-transactions
- Replication Factor: 3
- min.insync.replicas: 2

Partition 0: Broker1 (AZ-1), Broker3 (AZ-2), Broker5 (AZ-3)
Partition 1: Broker2 (AZ-1), Broker4 (AZ-2), Broker6 (AZ-3)

If AZ-1 fails (entire datacenter):
- All partitions lose 1 replica
- 2 replicas remain in AZ-2 and AZ-3 (meets min.insync.replicas=2)
- Cluster remains fully operational (zero downtime)
```

### Unclean Leader Election

**Unclean leader election** allows partitions to elect a leader from **out-of-sync replicas** when all ISR members are unavailable.

**Configuration**:
```properties
# Default (Kafka 1.0+): Disabled (prefer availability loss over data loss)
unclean.leader.election.enable = false

# Enable (prioritize availability over durability)
unclean.leader.election.enable = true
```

**Scenario: All ISR Members Fail**:
```
Partition 0:
- Leader: Broker1 (offsets 0-1000)
- ISR: [Broker1, Broker2, Broker3]
- Out-of-sync: Broker4 (offsets 0-800, lagging by 200 messages)

T+0s:  Brokers 1, 2, 3 all fail simultaneously (datacenter power outage)
T+10s: Controller detects all ISR members offline

Option A (unclean.leader.election.enable=false):
- Partition remains OFFLINE (no leader)
- Writes fail: NotEnoughReplicasException
- Reads fail: LeaderNotAvailableException
- Wait for ISR member to return (may take hours)

Option B (unclean.leader.election.enable=true):
- Controller elects Broker4 as leader (out-of-sync)
- Partition becomes AVAILABLE
- Messages 801-1000 are LOST (200 messages)
- New messages start at offset 801 (gap in offsets)
```

**Banking Decision Framework**:

```properties
# Payment Transactions (NEVER enable unclean elections)
unclean.leader.election.enable = false
# Rationale: Data loss is unacceptable (financial transactions)
# Accept downtime until ISR restored (deploy from backup, failover datacenter)

# Application Logs (enable unclean elections)
unclean.leader.election.enable = true
# Rationale: Availability > durability (losing some logs acceptable)
# Restore service immediately (avoid operational disruption)

# Fraud Alerts (depends on business requirements)
unclean.leader.election.enable = false  # Preferred
# If business accepts risk of missing fraud alerts during disaster, enable
```

---

## Partition Rebalancing

**Rebalancing** redistributes partitions among consumers in a consumer group when members join or leave.

### Rebalancing Triggers

1. **Consumer Joins**: New consumer added to group
2. **Consumer Leaves**: Consumer crashes, exits, or exceeds session timeout
3. **Topic Subscription Change**: Consumer subscribes to new topics
4. **Partition Count Change**: Partitions added to topic

### Rebalancing Strategies

#### 1. Range Assignor (Default)
```
Algorithm: Assign contiguous partition ranges to consumers

Example:
Topic: payments (10 partitions: 0-9)
Consumers: C1, C2, C3

Assignment:
- C1: Partitions 0, 1, 2, 3
- C2: Partitions 4, 5, 6
- C3: Partitions 7, 8, 9

Pros: Simple, predictable
Cons: Unbalanced if partitions not divisible by consumers (C1 gets 4, others get 3)
```

**Multi-Topic Imbalance**:
```
Topics: payments (10 partitions), refunds (10 partitions)
Consumers: C1, C2, C3

Range Assignor (per-topic):
Payments: C1=[0-3], C2=[4-6], C3=[7-9]
Refunds:  C1=[0-3], C2=[4-6], C3=[7-9]

Total: C1=8 partitions, C2=6, C3=6 (imbalanced!)
```

#### 2. Round-Robin Assignor
```
Algorithm: Assign partitions in round-robin fashion across all topics

Example:
Topics: payments (10 partitions), refunds (10 partitions)
Consumers: C1, C2, C3

Assignment (round-robin):
- C1: payments-0, payments-3, payments-6, payments-9, refunds-1, refunds-4, refunds-7
- C2: payments-1, payments-4, payments-7, refunds-0, refunds-3, refunds-6, refunds-9
- C3: payments-2, payments-5, payments-8, refunds-2, refunds-5, refunds-8

Total: C1=7, C2=7, C3=6 (more balanced)

Pros: Better balance across multiple topics
Cons: Less predictable, can cause more partition movement during rebalance
```

#### 3. Sticky Assignor
```
Algorithm: Minimize partition movement during rebalance (keep existing assignments)

Example (initial):
Consumers: C1, C2, C3
Partitions: 0-9
- C1: 0, 1, 2, 3
- C2: 4, 5, 6
- C3: 7, 8, 9

C2 leaves (rebalance):

Range Assignor (full reassignment):
- C1: 0, 1, 2, 3, 4
- C3: 5, 6, 7, 8, 9
- Movement: 5 partitions moved (4→C1, 5,6→C3)

Sticky Assignor (minimal movement):
- C1: 0, 1, 2, 3, 4
- C3: 7, 8, 9, 5, 6
- Movement: 3 partitions moved (4→C1, 5,6→C3)
- C1 and C3 keep existing partitions (0-3, 7-9)

Pros: Reduces rebalancing overhead, faster recovery
Cons: Slightly more complex logic
```

#### 4. Cooperative Sticky Assignor (Incremental Rebalancing)
```
Algorithm: Rebalance without "stop-the-world" pause

Traditional Rebalancing (Stop-the-World):
1. All consumers stop processing (STOP)
2. Rebalance completes (revoke + assign partitions)
3. All consumers resume processing (START)
4. Downtime: 10-30 seconds (consumer group unavailable)

Cooperative Rebalancing (Incremental):
1. Consumers continue processing existing partitions
2. Revoke only partitions that need to move
3. Assign revoked partitions to new consumers
4. Consumers resume (no downtime)

Example:
Consumers: C1, C2, C3
Partitions: 0-9
- C1: 0, 1, 2, 3
- C2: 4, 5, 6
- C3: 7, 8, 9

C4 joins (rebalance):

Stop-the-World (all consumers pause):
- C1: 0, 1, 2
- C2: 3, 4, 5
- C3: 6, 7, 8
- C4: 9
- Downtime: 15 seconds

Cooperative (incremental):
Phase 1 (revoke):
- C1 revokes partition 3 (continues processing 0, 1, 2)
- C2 revokes partition 6 (continues processing 4, 5)
- C3 revokes partition 9 (continues processing 7, 8)

Phase 2 (assign):
- C1: 0, 1, 2 (no change)
- C2: 4, 5 (no change)
- C3: 7, 8 (no change)
- C4: 3, 6, 9 (new assignments)

Downtime: 0 seconds (partitions 0, 1, 2, 4, 5, 7, 8 processed continuously)
```

**Configuration**:
```properties
# Consumer configuration
partition.assignment.strategy = org.apache.kafka.clients.consumer.CooperativeStickyAssignor

# Benefits:
# - Zero downtime during rebalancing
# - Critical for real-time systems (payment processing)
```

**Banking Example**:

```properties
# Payment Processing Consumer Group
partition.assignment.strategy = CooperativeStickyAssignor

# Scenario: Rolling deployment (6 consumers, 30 partitions)
# Deploy new version: C1, C2, C3 (old) → C4, C5, C6 (new)

Traditional Rebalancing:
T+0s:   C4 starts → rebalance triggered
T+0s:   All consumers (C1, C2, C3) STOP processing (30 partitions paused)
T+15s:  Rebalance completes (C1, C2, C3, C4 assigned partitions)
T+15s:  All consumers START processing
Impact: 15-second downtime (450 payments delayed at 30 TPS per partition)

Cooperative Rebalancing:
T+0s:   C4 starts → rebalance triggered
T+0s:   C1, C2, C3 continue processing (28 partitions active, 2 revoked)
T+1s:   C4 assigned 2 partitions, starts processing
T+1s:   All 30 partitions active again
Impact: 0-second downtime (0 payments delayed)

Result: Cooperative rebalancing prevents 450 payment delays per deployment
```

---

## Partition Management

### Adding Partitions

**Scenario**: Topic throughput exceeds capacity, need more parallelism.

```bash
# Increase partition count from 10 to 20
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic payments --partitions 20

# Kafka creates new partitions (10-19)
# Existing partitions (0-9) unchanged
```

**Impact on Consumers**:
```
Before:
- 10 consumers, 10 partitions (1:1 mapping)

After:
- 10 consumers, 20 partitions (rebalance triggered)
- Each consumer gets 2 partitions

Later:
- Scale to 20 consumers (1:1 mapping restored)
```

**Impact on Ordering**:
```
Key: "account-123"
Before (10 partitions): hash("account-123") % 10 = partition 3
After  (20 partitions): hash("account-123") % 20 = partition 3 (unchanged)

Key: "account-456"
Before (10 partitions): hash("account-456") % 10 = partition 7
After  (20 partitions): hash("account-456") % 20 = partition 17 (changed!)

Warning: Adding partitions breaks ordering for some keys (different partition assignment)
```

**Best Practice**: Choose partition count upfront (avoid changing later). If necessary to add partitions:
1. **Stop producers** (ensure all in-flight messages completed)
2. **Add partitions** (hash function changes for some keys)
3. **Restart producers** (use new partition count)
4. **Accept**: Messages with same key may be in different partitions (breaks ordering)

**Alternative**: Create new topic with desired partition count, migrate consumers/producers.

### Partition Reassignment

**Scenario**: Add brokers, rebalance load, or replace failed broker.

```bash
# Step 1: Generate reassignment plan
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json \
  --broker-list "1,2,3,4,5,6" \
  --generate

# topics.json:
{
  "topics": [{"topic": "payments"}],
  "version": 1
}

# Step 2: Review generated plan (JSON output)
# Ensures replicas spread across all brokers

# Step 3: Execute reassignment
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute

# Step 4: Monitor progress
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify

# Reassignment complete when all partitions show "Reassignment completed successfully"
```

**Performance Impact**:
- **Network**: Replication traffic increases (copying partitions to new brokers)
- **Disk I/O**: Source and target brokers experience high I/O
- **Latency**: Produce/consume latency may increase during reassignment

**Throttling**:
```bash
# Limit reassignment bandwidth to 50 MB/s per broker
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute \
  --throttle 52428800  # 50 MB/s in bytes

# Remove throttle after reassignment completes
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-default \
  --alter --delete-config follower.replication.throttled.rate,leader.replication.throttled.rate
```

**Banking Example**:

```bash
# Scenario: Add 3 brokers (Broker4, Broker5, Broker6) to handle increased load
# Current: 3 brokers, payment topic (30 partitions, RF=3)

# Step 1: Generate reassignment (spread across 6 brokers)
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json \
  --broker-list "1,2,3,4,5,6" \
  --generate > reassignment.json

# Step 2: Execute with throttle (avoid impacting production traffic)
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute --throttle 104857600  # 100 MB/s

# Step 3: Monitor (reassignment takes ~30 minutes for 1 TB data)
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify

# Result: 30 partitions evenly distributed across 6 brokers (5 partitions each)
```

---

## Interview Questions

### Question 1: How would you determine the optimal partition count for a new high-volume topic? Walk through your calculation and considerations.

**Answer**:

Determining partition count requires analyzing **throughput requirements**, **consumer parallelism**, **ordering constraints**, and **operational overhead**.

**Step 1: Calculate Based on Throughput**

```
Target Throughput: 100,000 messages/sec

Consumer Throughput (benchmark single consumer):
- Network: 10 MB/s
- Processing: 5,000 messages/sec
- Bottleneck: Processing (5,000 messages/sec)

Partitions Needed (Consumer):
100,000 / 5,000 = 20 partitions

Producer Throughput (benchmark single producer):
- Network: 50 MB/s
- Kafka broker write: 50,000 messages/sec per partition

Partitions Needed (Producer):
100,000 / 50,000 = 2 partitions

Result: Consumer-bound (need 20 partitions for consumer parallelism)
```

**Step 2: Factor in Growth**

```
Current: 100,000 messages/sec
Expected Growth: 50% per year
Planning Horizon: 2 years

Future Throughput: 100,000 × 1.5 × 1.5 = 225,000 messages/sec
Future Partitions: 225,000 / 5,000 = 45 partitions

Recommendation: Start with 30 partitions (33% headroom)
- Handles current load: 30 × 5,000 = 150,000 messages/sec
- Allows growth without adding partitions (avoid ordering issues)
```

**Step 3: Consider Operational Overhead**

```
Broker Constraints:
- 3 brokers
- Replication Factor: 3
- Target: <2,000 partitions per broker

Total Partition Replicas: 30 partitions × 3 RF = 90 replicas
Replicas per Broker: 90 / 3 brokers = 30 replicas per broker

Result: Well within limits (30 << 2,000)
```

**Step 4: Validate Ordering Requirements**

```
Partition Key: accountId (maintain transaction ordering per account)

With 30 partitions:
- Each account's transactions go to 1 partition (ordered)
- 30 accounts can be processed in parallel
- Example: account-123 → partition 7 (all transactions ordered by offset)

Result: Ordering requirement satisfied
```

**Step 5: Consider Rebalancing Impact**

```
Consumer Group: 30 consumers (1 per partition)

Rebalancing Time (benchmark):
- 10 partitions: ~5 seconds
- 30 partitions: ~10 seconds
- 100 partitions: ~30 seconds

With 30 partitions:
- Rebalancing during rolling deployment: 10 seconds
- Acceptable for payment processing (use CooperativeStickyAssignor for zero downtime)

Result: Acceptable rebalancing time
```

**Final Recommendation**:

```properties
# Payment Topic Configuration
num.partitions = 30
replication.factor = 3
min.insync.replicas = 2

# Rationale:
# - Handles 150,000 messages/sec (50% over current 100,000)
# - Allows 2x growth before adding partitions (225,000 future target)
# - 30 consumers for full parallelism
# - Acceptable operational overhead (30 replicas/broker)
# - Ordering preserved (partition key = accountId)
# - Rebalancing time < 10 seconds (use cooperative rebalancing for zero downtime)

# Consumer Configuration:
partition.assignment.strategy = CooperativeStickyAssignor
```

**Interview Tip**: Emphasize that partition count is difficult to change later (breaks ordering). Choose conservatively upfront, accounting for growth. Benchmark consumer/producer throughput in your environment (don't rely on theoretical numbers).

---

### Question 2: Explain the impact of partition key selection on load balancing. How would you handle a scenario where 20% of keys generate 80% of traffic (hotspot)?

**Answer**:

**Problem: Hotspot from Skewed Key Distribution**

```
Scenario: Payment processing topic
- Partition Key: accountId
- 1,000,000 accounts
- Account "corporate-mega-bank": 50,000 TPS (50% of total traffic)
- Other accounts: 50,000 TPS combined (1,000 accounts @ 50 TPS each)

Partitioning Result:
- "corporate-mega-bank" → Partition 17 (50,000 TPS)
- Other accounts → Partitions 0-49 (1,000 TPS per partition)

Impact:
- Partition 17: 50x overload (consumer lag, slow processing)
- Other partitions: 2% utilized (wasted capacity)
- SLA violation: Payments for "corporate-mega-bank" delayed
```

**Solution 1: Composite Key with Sharding**

```java
/**
 * Shard high-volume accounts across multiple partitions
 */
public ProducerRecord<String, Payment> createRecord(Payment payment) {
    String accountId = payment.getAccountId();
    String key;

    // High-volume accounts: shard across 10 partitions
    if (HIGH_VOLUME_ACCOUNTS.contains(accountId)) {
        int shard = Math.abs(payment.getTransactionId().hashCode()) % 10;
        key = accountId + ":" + shard;
        // "corporate-mega-bank:0", "corporate-mega-bank:1", ..., "corporate-mega-bank:9"
    } else {
        // Regular accounts: use accountId as key
        key = accountId;
    }

    return new ProducerRecord<>("payments", key, payment);
}

// Result:
// - "corporate-mega-bank" traffic distributed across 10 partitions (5,000 TPS each)
// - Other accounts remain on single partition each (ordering preserved)
// - Balanced load: all partitions ~5,000 TPS
```

**Trade-off**: Per-account ordering is lost for high-volume accounts (10 partitions per account). **Mitigation**: Use transaction ID sequencing or idempotent consumers to handle out-of-order messages.

**Solution 2: Custom Partitioner with Dedicated Partitions**

```java
/**
 * Allocate dedicated partitions for high-volume accounts
 */
public class HotspotAwarePartitioner implements Partitioner {

    // Reserve partitions 0-9 for top 10 high-volume accounts
    private static final Map<String, Integer> ACCOUNT_TO_PARTITION = Map.of(
        "corporate-mega-bank", 0,
        "hedge-fund-alpha", 1,
        "investment-bank-beta", 2
        // ... (10 accounts)
    );

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String accountId = (String) key;
        int numPartitions = cluster.partitionCountForTopic(topic);

        // High-volume accounts: dedicated partition
        if (ACCOUNT_TO_PARTITION.containsKey(accountId)) {
            return ACCOUNT_TO_PARTITION.get(accountId);
        }

        // Regular accounts: hash across remaining partitions
        int hash = Math.abs(murmur2(keyBytes));
        return (hash % (numPartitions - 10)) + 10;  // Partitions 10-49
    }
}

// Result:
// - Top 10 accounts: 1 partition each (isolated, 5,000 TPS each)
// - Can scale consumers independently (2 consumers on partition 0 for corporate-mega-bank)
// - Regular accounts: distributed across 40 partitions (balanced)
```

**Solution 3: Separate Topics for High-Volume Accounts**

```java
/**
 * Route high-volume accounts to dedicated topic
 */
public String selectTopic(Payment payment) {
    String accountId = payment.getAccountId();

    if (HIGH_VOLUME_ACCOUNTS.contains(accountId)) {
        return "payments-high-volume";  // 10 partitions, dedicated consumers
    } else {
        return "payments-regular";      // 50 partitions, shared consumers
    }
}

// Result:
// - "payments-high-volume": 10 partitions, 10 high-volume accounts (5,000 TPS each)
// - "payments-regular": 50 partitions, 999,990 regular accounts (1,000 TPS per partition)
// - Independent scaling (add consumers to high-volume topic without affecting regular topic)
```

**Solution 4: Dynamic Partitioning with Consistent Hashing**

```java
/**
 * Use consistent hashing to distribute load while maintaining locality
 */
public class ConsistentHashPartitioner implements Partitioner {

    private ConsistentHash<Integer> ring;

    @Override
    public void configure(Map<String, ?> configs) {
        // Build consistent hash ring (virtual nodes for even distribution)
        ring = new ConsistentHash<>(1000, /* virtual nodes per partition */ 100);
    }

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String accountId = (String) key;

        // For high-volume accounts, use transaction ID for finer distribution
        if (HIGH_VOLUME_ACCOUNTS.contains(accountId)) {
            Payment payment = (Payment) value;
            String hashKey = accountId + ":" + payment.getTransactionId();
            return ring.get(hashKey);
        }

        // Regular accounts use account ID
        return ring.get(accountId);
    }
}

// Benefit: Gradual redistribution (no hotspot concentration)
```

**Comparison of Solutions**:

| Solution | Ordering | Complexity | Rebalancing | Best For |
|----------|---------|------------|-------------|----------|
| **Composite Key Sharding** | Partial (lost per account) | Low | Automatic | Most scenarios |
| **Custom Partitioner** | Full (per account) | Medium | Manual | Known hotspots |
| **Separate Topics** | Full | High | Independent | Distinct workloads |
| **Consistent Hashing** | Partial | High | Smooth | Dynamic hotspots |

**Banking Recommendation**:

```java
// Use Composite Key Sharding with idempotent consumers
// - Simple implementation
// - Automatic load balancing
// - Acceptable ordering trade-off (use transaction sequencing in consumer)

public class PaymentProducer {

    public void sendPayment(Payment payment) {
        String key = buildKey(payment);
        ProducerRecord<String, Payment> record =
            new ProducerRecord<>("payments", key, payment);
        producer.send(record);
    }

    private String buildKey(Payment payment) {
        String accountId = payment.getAccountId();

        // Shard high-volume accounts across 10 virtual partitions
        if (isHighVolumeAccount(accountId)) {
            int shard = Math.abs(payment.hashCode()) % 10;
            return accountId + ":" + shard;
        }

        return accountId;
    }
}

// Consumer handles out-of-order messages (idempotent processing)
public class PaymentConsumer {

    private Map<String, Long> lastProcessedSequence = new HashMap<>();

    public void processPayment(Payment payment) {
        String accountId = payment.getAccountId();
        long sequence = payment.getSequenceNumber();

        // Skip duplicate or out-of-order messages
        if (sequence <= lastProcessedSequence.getOrDefault(accountId, 0L)) {
            return;  // Already processed
        }

        // Process payment (idempotent operation)
        updateAccountBalance(payment);
        lastProcessedSequence.put(accountId, sequence);
    }
}
```

**Interview Tip**: Highlight that hotspots are inevitable in real-world systems (Pareto principle: 80/20 rule). The key is to detect hotspots via monitoring (consumer lag per partition) and apply appropriate mitigation based on ordering requirements.

---

### Question 3: How does rack awareness work in Kafka, and what are the trade-offs of enabling it in a multi-datacenter deployment?

**Answer**:

**Rack Awareness** ensures Kafka distributes partition replicas across different **racks** (physical racks, availability zones, or datacenters) to survive correlated failures.

**How It Works**:

```properties
# Broker configuration (assign rack/AZ)
broker.rack = az-1  # Broker1 in AZ-1
broker.rack = az-2  # Broker2 in AZ-2
broker.rack = az-3  # Broker3 in AZ-3
```

**Replica Assignment Algorithm**:
```
Without Rack Awareness (random assignment):
- Replicas distributed randomly across brokers
- Risk: Multiple replicas in same rack (correlated failure)

With Rack Awareness (rack-spread algorithm):
1. Select leader broker randomly
2. Select follower replicas from different racks
3. If RF > number of racks, distribute remaining replicas evenly

Example (RF=3, 3 racks, 6 brokers):
Partition 0: Broker1 (Rack1), Broker3 (Rack2), Broker5 (Rack3)
- Each replica in different rack (survives 1 rack failure)

Example (RF=5, 3 racks, 6 brokers):
Partition 0: Broker1 (Rack1), Broker2 (Rack1), Broker3 (Rack2), Broker4 (Rack2), Broker5 (Rack3)
- Racks have 2, 2, 1 replicas (survives 1 rack failure with min.insync.replicas=3)
```

**Multi-Datacenter Trade-offs**:

**Scenario: Stretch Cluster (3 datacenters, same region)**

```
Configuration:
- DC1: Broker1, Broker2 (broker.rack=dc1)
- DC2: Broker3, Broker4 (broker.rack=dc2)
- DC3: Broker5, Broker6 (broker.rack=dc3)
- Network Latency: <5ms between DCs
- Topic: payments (RF=3, min.insync.replicas=2)

Benefits:
- Survives full DC failure (e.g., DC1 power outage)
- 2 replicas remain in DC2 + DC3 (meets min.insync.replicas=2)
- Zero data loss

Trade-offs:
- Increased Latency: Producer with acks=all waits for cross-DC replication (5ms added)
- Network Costs: Cross-DC traffic incurs charges (AWS: $0.01/GB inter-AZ)
- Reduced Throughput: Network bottleneck (inter-AZ bandwidth ~5 Gbps vs. intra-AZ 25 Gbps)
```

**Scenario: Multi-Region Cluster (2 regions, 3,000 miles apart)**

```
Configuration:
- US-East: Broker1, Broker2, Broker3 (broker.rack=us-east)
- US-West: Broker4, Broker5, Broker6 (broker.rack=us-west)
- Network Latency: 70ms between regions
- Topic: payments (RF=6, min.insync.replicas=4)

Benefits:
- Survives full region failure (disaster recovery)
- 3 replicas remain in surviving region (meets min.insync.replicas=4 with both regions, 3 with one region down)

Trade-offs:
- High Latency: Producer with acks=all waits 70ms for cross-region replication (unacceptable for real-time)
- High Network Costs: Cross-region traffic ($0.02/GB AWS)
- Complex Failover: Consumers/producers must re-point to surviving region

Alternative: Use MirrorMaker for async cross-region replication (avoid high latency)
```

**Rack Awareness Trade-off Matrix**:

| Deployment | Latency | Network Cost | Failure Tolerance | Recommendation |
|-----------|---------|--------------|-------------------|----------------|
| **Single DC, Multiple Racks** | +0ms | $0 | Rack failure | Enable (no downsides) |
| **Multi-AZ, Same Region** | +2-5ms | $0.01/GB | AZ failure | Enable (acceptable latency) |
| **Multi-Region, Stretch** | +50-100ms | $0.02/GB | Region failure | Avoid (use async replication) |

**Banking Recommendation**:

```properties
# Production Payment Topic (Multi-AZ, Same Region)
replication.factor = 3
min.insync.replicas = 2
broker.rack = az-1, az-2, az-3  # Enable rack awareness

# Benefits:
# - Survives AZ failure (zero data loss)
# - Latency: +3ms (acceptable for payments, p99 < 10ms)
# - Network cost: $500/month (1 TB/day cross-AZ @ $0.01/GB)

# Trade-off accepted: 3ms latency for zero data loss (regulatory requirement)

# Disaster Recovery Topic (Cross-Region, Async)
# Use MirrorMaker 2.0 for async replication to DR region
# Primary: US-East (real-time processing)
# Secondary: US-West (read-only, failover target)
# RPO: 5 minutes (async lag)
# RTO: 15 minutes (manual failover)
```

**When NOT to Use Rack Awareness**:

```
Scenario 1: Single Datacenter (no racks)
- broker.rack not configured (no failure domain separation)
- Use high replication factor (RF=5) instead

Scenario 2: Development/Testing
- Single-node cluster or co-located brokers
- Rack awareness adds no value

Scenario 3: Latency-Sensitive Applications (<1ms p99)
- Co-locate brokers in same rack (minimize network latency)
- Accept rack failure risk (use faster failover instead of rack spreading)
```

**Monitoring Rack-Aware Deployments**:

```bash
# Verify replicas spread across racks
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic payments

# Example output:
# Partition 0: Leader: 1, Replicas: 1,3,5, Isr: 1,3,5
#   Broker1 (Rack1), Broker3 (Rack2), Broker5 (Rack3) ✓

# Monitor cross-AZ replication lag
# Alert if lag >10 seconds (indicates network issues)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group payment-processor --describe
```

**Interview Tip**: Emphasize that rack awareness is a **failure domain** concept, not just physical racks. In cloud deployments, map racks to Availability Zones. Highlight the latency trade-off (acceptable for multi-AZ, prohibitive for multi-region).

---

### Question 4: Explain the difference between eager rebalancing (stop-the-world) and cooperative rebalancing. When would you choose one over the other?

**Answer**:

**Eager Rebalancing** (traditional, stop-the-world):
- All consumers **stop processing** during rebalance
- Partitions are **revoked** from all consumers
- New partition assignment calculated
- Partitions **assigned** to consumers
- Consumers **resume processing**

**Cooperative Rebalancing** (incremental):
- Consumers **continue processing** current partitions during rebalance
- Only partitions that need to **move** are revoked
- Revoked partitions assigned to new consumers
- Minimal disruption (no stop-the-world pause)

**Detailed Comparison**:

**Eager Rebalancing Timeline**:
```
T+0s:  Consumer C4 joins group (3 existing consumers: C1, C2, C3)
T+0s:  Group coordinator detects new member
T+0s:  Coordinator sends STOP_PARTITION_ASSIGNMENT to C1, C2, C3
T+0s:  C1, C2, C3 revoke ALL partitions (stop processing)
        ↓ DOWNTIME STARTS (all 30 partitions paused)
T+5s:  All consumers report revocation complete
T+5s:  Coordinator calculates new assignment (30 partitions across 4 consumers)
T+5s:  Coordinator sends partition assignments to C1, C2, C3, C4
T+10s: All consumers receive assignments, start processing
        ↓ DOWNTIME ENDS

Total Downtime: 10 seconds (all 30 partitions unavailable)
```

**Cooperative Rebalancing Timeline**:
```
T+0s:  Consumer C4 joins group
T+0s:  Group coordinator detects new member
T+0s:  Coordinator identifies partitions to revoke:
       - C1: revoke partition 29 (keep 0-28)
       - C2: revoke partition 30 (keep partitions, minimal)
       - C3: no revocations (keep all)
T+0s:  C1 revokes partition 29 (continues processing 0-28)
       29 partitions continue processing ✓
T+1s:  Partition 29 revoked
T+1s:  Coordinator assigns partition 29 to C4
T+1s:  C4 starts processing partition 29
        ↓ All 30 partitions active again

Total Downtime: 0 seconds (29 partitions never paused, partition 29 paused for 1 second)
```

**When to Use Eager Rebalancing**:

```
Scenario 1: Legacy Kafka (< 2.4)
- Cooperative rebalancing not available
- No choice (must use eager)

Scenario 2: Stateless Consumers (no local state)
- Consumer processing is idempotent
- No startup cost (no state to load)
- Rebalancing time is acceptable (<5 seconds)

Example: Log aggregation consumer
- Reads log messages, writes to S3
- Stateless (no local caching)
- 5-second rebalance acceptable (not real-time critical)

Configuration:
partition.assignment.strategy = org.apache.kafka.clients.consumer.RangeAssignor
```

**When to Use Cooperative Rebalancing**:

```
Scenario 1: Real-Time Systems (payments, trading, fraud detection)
- Downtime unacceptable (SLA: 99.99% availability)
- Every second of downtime = lost revenue

Example: Payment processing
- 100,000 payments/hour
- 10-second rebalance = 278 missed payments (100,000 / 3,600 × 10)
- Cooperative rebalancing: 0 missed payments

Configuration:
partition.assignment.strategy = org.apache.kafka.clients.consumer.CooperativeStickyAssignor

Scenario 2: Stateful Consumers (local state stores)
- Consumers maintain local RocksDB state
- Rebalancing requires state reload (10-60 seconds per partition)
- Eager rebalancing: all consumers reload state simultaneously (1-2 minute downtime)
- Cooperative rebalancing: only moved partitions reload state (10 seconds downtime)

Example: Fraud detection with local user profile cache
- Consumer maintains RocksDB with 1M user profiles (5 GB)
- Partition reassignment requires reloading RocksDB (30 seconds)
- With 30 partitions:
  - Eager: All 30 partitions reload (30 seconds downtime)
  - Cooperative: 2 partitions reload (30 seconds for those 2, others continue)

Configuration:
partition.assignment.strategy = org.apache.kafka.clients.consumer.CooperativeStickyAssignor

Scenario 3: Frequent Rebalancing (rolling deployments, autoscaling)
- Kubernetes environment with pod autoscaling
- Pods frequently added/removed (every 10 minutes)
- Eager rebalancing: 10-second downtime every 10 minutes (98.3% availability)
- Cooperative rebalancing: 0-second downtime (99.99% availability)

Configuration:
partition.assignment.strategy = org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

**Migration Strategy (Eager → Cooperative)**:

```java
// Step 1: Update all consumers to support BOTH strategies
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    Arrays.asList(
        CooperativeStickyAssignor.class.getName(),
        RangeAssignor.class.getName()  // Fallback for old consumers
    )
);

// Step 2: Rolling deployment (consumers gradually switch to cooperative)
// - Old consumers: Use RangeAssignor (eager)
// - New consumers: Use CooperativeStickyAssignor
// - Both strategies compatible during transition

// Step 3: After all consumers upgraded, remove fallback
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    CooperativeStickyAssignor.class.getName()
);
```

**Trade-offs**:

| Aspect | Eager Rebalancing | Cooperative Rebalancing |
|--------|------------------|------------------------|
| **Downtime** | 5-30 seconds (all partitions) | 0 seconds (most partitions) |
| **Complexity** | Simple (single-phase) | Complex (multi-phase) |
| **Kafka Version** | All versions | 2.4+ |
| **Compatibility** | All assignors | CooperativeStickyAssignor only |
| **State Reload** | All partitions reload | Only moved partitions reload |
| **Use Case** | Batch processing, tolerant of downtime | Real-time, zero-downtime requirements |

**Banking Example**:

```java
// Payment Processing Consumer (real-time, 99.99% SLA)
Properties props = new Properties();
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    CooperativeStickyAssignor.class.getName());
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);  // 30s
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000); // 3s

// Scenario: Rolling deployment (6 consumers → 6 new consumers)
// Eager: 6 rebalances × 10 seconds = 60 seconds total downtime
// Cooperative: 6 rebalances × 0 seconds = 0 seconds downtime
// Result: 0 missed payments (vs. 1,667 missed payments with eager)

// Audit Log Consumer (batch processing, not real-time)
Properties props = new Properties();
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    RangeAssignor.class.getName());  // Eager is acceptable

// Scenario: Batch job runs hourly (rebalancing once per deployment, weekly)
// Eager: 10 seconds downtime per week (acceptable for batch processing)
// Cooperative: Not needed (no real-time requirement)
```

**Interview Tip**: Emphasize that cooperative rebalancing is **essential for real-time systems** with strict SLA requirements. Highlight the migration path (support both strategies during transition) and the trade-off (complexity for zero downtime).

---

### Question 5: You have a topic with 100 partitions and 50 consumers. Describe the partition assignment and what happens when you add 50 more consumers.

**Answer**:

**Initial State: 100 Partitions, 50 Consumers**

```
Using RangeAssignor (default):

Total Partitions: 100 (0-99)
Total Consumers: 50 (C0-C49)

Assignment Formula:
Partitions per consumer = 100 / 50 = 2

Assignment:
- C0:  Partitions 0, 1
- C1:  Partitions 2, 3
- C2:  Partitions 4, 5
- ...
- C49: Partitions 98, 99

Result: Each consumer handles 2 partitions (perfectly balanced)
```

**After Adding 50 Consumers: 100 Partitions, 100 Consumers**

```
Rebalance Triggered (all consumers join within session.timeout.ms)

New Total: 100 partitions, 100 consumers (C0-C99)

New Assignment:
Partitions per consumer = 100 / 100 = 1

Assignment:
- C0:  Partition 0
- C1:  Partition 1
- C2:  Partition 2
- ...
- C99: Partition 99

Result: Each consumer handles 1 partition (perfect 1:1 mapping)
```

**Rebalancing Timeline (Eager Rebalancing)**:

```
T+0s:   50 new consumers (C50-C99) join the group
T+0s:   Group coordinator detects 50 new members
T+0s:   Coordinator sends STOP to all consumers (C0-C49)
T+0s:   All 50 existing consumers revoke 100 partitions (stop processing)
        ↓ DOWNTIME: All 100 partitions paused
T+5s:   All consumers report revocation complete
T+5s:   Coordinator calculates new assignment (100 partitions across 100 consumers)
T+10s:  All 100 consumers receive assignments, start processing
        ↓ DOWNTIME ENDS

Total Downtime: 10 seconds
Impact: 10 seconds × throughput = missed messages (e.g., 10,000 messages at 1,000 msg/sec)
```

**Rebalancing Timeline (Cooperative Rebalancing)**:

```
T+0s:   50 new consumers (C50-C99) join
T+0s:   Coordinator identifies partitions to move:
        - C0 revokes partition 1 (keeps partition 0)
        - C1 revokes partition 3 (keeps partition 2)
        - ...
        - C49 revokes partition 99 (keeps partition 98)
        Total: 50 partitions revoked

T+0s:   Existing consumers continue processing 50 partitions (0, 2, 4, ..., 98)
        50 partitions continue processing ✓

T+1s:   50 partitions revoked, assigned to new consumers (C50-C99)
T+1s:   New consumers start processing
        ↓ All 100 partitions active again

Total Downtime: 0 seconds (50 partitions never paused, 50 partitions paused for 1 second)
Impact: Minimal (50 partitions × 1 second = 50 messages missed at 1,000 msg/sec)
```

**Partition Movement**:

```
Before (50 consumers):
C0: [0, 1]  C1: [2, 3]  C2: [4, 5]  ...  C49: [98, 99]

After (100 consumers):
C0: [0]     C1: [2]     C2: [4]     ...  C49: [98]
C50: [1]    C51: [3]    C52: [5]    ...  C99: [99]

Partitions Moved:
- 50 partitions moved from old consumers to new consumers (partitions 1, 3, 5, ..., 99)
- 50 partitions stayed with old consumers (partitions 0, 2, 4, ..., 98)

Eager Rebalancing:
- All 100 partitions reassigned (full stop-the-world)

Cooperative Rebalancing:
- Only 50 partitions moved (incremental, no downtime)
```

**Performance Impact**:

```
Before (50 consumers, 2 partitions each):
- Per-consumer throughput: 2,000 messages/sec (1,000 per partition)
- Total throughput: 100,000 messages/sec

After (100 consumers, 1 partition each):
- Per-consumer throughput: 1,000 messages/sec (1 partition)
- Total throughput: 100,000 messages/sec (unchanged)

Benefit: Lower per-consumer load (better CPU utilization, lower p99 latency)
Trade-off: More consumers = higher operational overhead (more JVMs, memory)
```

**When Adding Consumers is Beneficial**:

```
Scenario 1: Consumer Lag
- Current: 50 consumers, each processing 1,000 msg/sec
- Incoming: 2,000 msg/sec per partition
- Lag: Consumers falling behind (1,000 msg/sec deficit per partition)
- Solution: Add 50 consumers (1 partition each, 2,000 msg/sec capacity)
- Result: Lag eliminated

Scenario 2: High CPU per Consumer
- Current: Each consumer at 80% CPU (processing 2 partitions)
- Risk: CPU spikes cause GC pauses (missed heartbeats, rebalancing)
- Solution: Add 50 consumers (1 partition each, 40% CPU per consumer)
- Result: Stable CPU, fewer rebalancing events
```

**When Adding Consumers is NOT Beneficial**:

```
Scenario 1: More Consumers than Partitions
- Current: 100 partitions, 100 consumers (1:1)
- Add: 50 more consumers (total 150)
- Result: 50 consumers idle (no partitions assigned)
- Waste: 50 idle JVMs consuming resources

Rule: Consumers should NOT exceed partition count

Scenario 2: Consumer Bottleneck is I/O, not CPU
- Current: 50 consumers, bottlenecked by database writes (200 writes/sec per consumer)
- Add: 50 consumers (total 100)
- Result: Same database bottleneck (200 writes/sec per consumer)
- Solution: Scale database, not consumers
```

**Banking Example**:

```
Payment Processing Topic:
- Partitions: 100
- Current Consumers: 50 (2 partitions each)
- Throughput: 100,000 payments/sec (1,000 per partition)
- Current Load: 2,000 payments/sec per consumer (80% CPU)

Scenario: Black Friday traffic spike (200,000 payments/sec)

Option 1: Add 50 consumers (total 100)
- New Load: 2,000 payments/sec per consumer (1 partition each)
- CPU: 80% (unchanged per partition, but spread across more consumers)
- Result: Lag grows (capacity unchanged: 100,000 payments/sec)

Option 2: Add 100 partitions + 100 consumers (total 200 partitions, 200 consumers)
- New Load: 1,000 payments/sec per consumer (1 partition each)
- CPU: 40% (lower per consumer)
- Result: Capacity doubled (200,000 payments/sec)

Correct Solution: Add partitions FIRST, then add consumers
```

**Best Practices**:

```
1. Partition count should accommodate peak load
   - Plan for 2-3x growth
   - Add partitions upfront (difficult to add later)

2. Scale consumers to match partition count (1:1 ideal)
   - Fewer consumers: higher load per consumer
   - More consumers: some idle (wasted resources)

3. Use cooperative rebalancing for zero-downtime scaling
   - Critical for real-time systems
   - Acceptable brief lag for 50 partitions during 1-second revocation

4. Monitor consumer lag per partition
   - Alert if lag >10,000 messages (indicates need for more consumers)
   - Alert if consumers idle (more consumers than partitions)
```

**Interview Tip**: Emphasize the 1:1 consumer-to-partition ratio as the ideal for maximum parallelism. Highlight that adding consumers beyond partition count provides no benefit (idle consumers). Discuss cooperative rebalancing as essential for production systems.

---

## Summary

Topics, partitions, and replication are the architectural foundation of Kafka's scalability, performance, and fault tolerance. Understanding how to design topics, select partition counts, choose partition keys, and configure replication is essential for building production-grade Kafka systems.

**Key Takeaways for Interviews**:
1. **Partition Count**: Calculate based on throughput (producer, consumer), growth projections, and operational overhead
2. **Partition Keys**: Entity ID for ordering, null for load balancing, custom partitioner for hotspot mitigation
3. **Replication Factor**: RF=3 for production (balance durability/cost), RF=5 for critical data (regulatory)
4. **Rack Awareness**: Essential for multi-AZ deployments (tolerate AZ failures), avoid for multi-region (high latency)
5. **Rebalancing**: Cooperative rebalancing for zero-downtime real-time systems, eager for batch processing
6. **Hotspot Handling**: Shard high-volume keys, use custom partitioners, or separate topics
7. **Consumer Scaling**: Match consumer count to partition count (1:1 ideal), monitor lag per partition

In enterprise banking, these concepts translate to **meeting SLAs** (99.99% availability), **ensuring data durability** (zero payment loss), and **achieving scalability** (100,000+ TPS). Mastering partition and replication design enables you to architect Kafka solutions for the most demanding financial workloads.

**Word Count**: ~8,000 words
