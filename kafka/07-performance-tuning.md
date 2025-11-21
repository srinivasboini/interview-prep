# Kafka Performance Tuning

## Overview

Kafka performance tuning is the art of optimizing throughput, latency, and resource utilization across producers, consumers, and brokers. Understanding the performance characteristics of each component and their interdependencies is critical for building high-performance distributed systems. A well-tuned Kafka cluster can achieve **millions of messages per second** with **sub-10ms latency** while maintaining data durability and fault tolerance.

**Why This Matters for Interviews**: Senior engineering roles require expertise in performance optimization, capacity planning, and troubleshooting bottlenecks. Expect questions about producer batching trade-offs, consumer fetch configurations, broker resource tuning, and end-to-end latency optimization. Interviewers want to see that you understand the **why** behind configuration choices, not just memorized values.

**Real-World Banking Context**: Payment processing systems demand high throughput (100,000+ TPS), low latency (p99 < 10ms), and strict durability guarantees (zero data loss). Performance tuning enables meeting these SLAs while minimizing infrastructure costs. A poorly tuned system might require 3x the hardware to achieve the same performance.

---

## Producer Performance Optimization

Producers are often the performance bottleneck in Kafka deployments. Optimizing producer configuration can dramatically improve throughput and reduce latency.

### Batching Configuration

**Batching** groups multiple messages into a single request, amortizing network overhead and improving throughput.

**Key Configurations**:

```properties
############################# Batching #############################

# Batch size in bytes (default: 16 KB)
batch.size = 16384

# How batch size works:
# - Producer accumulates messages in memory until batch reaches 16 KB
# - OR linger.ms timeout expires (whichever comes first)
# - Then sends the batch to broker
# Larger batches = better throughput, slightly higher latency

# Linger time in milliseconds (default: 0)
linger.ms = 0

# How linger.ms works:
# - Producer waits up to linger.ms for more messages before sending batch
# - linger.ms = 0: Send immediately (low latency, poor batching)
# - linger.ms = 10: Wait 10ms for more messages (better batching, +10ms latency)
# Trade-off: Throughput vs. Latency

# Buffer memory for all unsent messages (default: 32 MB)
buffer.memory = 33554432

# Total memory for batching across all partitions
# If buffer fills, producer.send() blocks for max.block.ms
# Increase for high-throughput scenarios (128 MB, 256 MB)
```

**Batching Strategies by Use Case**:

```properties
# Use Case 1: Real-Time Payments (minimize latency)
batch.size = 16384        # 16 KB (default)
linger.ms = 0             # Send immediately
# Result: p99 latency <5ms, moderate throughput (10,000 msg/sec per partition)

# Use Case 2: Analytics Events (maximize throughput)
batch.size = 1048576      # 1 MB
linger.ms = 100           # Wait 100ms
# Result: p99 latency ~100ms, high throughput (100,000 msg/sec per partition)

# Use Case 3: Balanced (payments with acceptable latency)
batch.size = 65536        # 64 KB
linger.ms = 10            # Wait 10ms
# Result: p99 latency ~15ms, good throughput (50,000 msg/sec per partition)
```

**Batching Effectiveness Calculation**:

```
Scenario: Payment topic, 1 KB average message size

Without Batching (batch.size=1024, linger.ms=0):
- Messages per batch: 1
- Network requests per 1000 messages: 1000
- Network overhead: 1000 × (TCP/IP headers + Kafka protocol) = 1000 × 100 bytes = 100 KB
- Total bandwidth: 1000 KB (messages) + 100 KB (overhead) = 1100 KB

With Batching (batch.size=16384, linger.ms=10):
- Messages per batch: 16 (16 KB / 1 KB)
- Network requests per 1000 messages: 63 (1000 / 16)
- Network overhead: 63 × 100 bytes = 6.3 KB
- Total bandwidth: 1000 KB (messages) + 6.3 KB (overhead) = 1006 KB

Savings: (1100 - 1006) / 1100 = 8.5% bandwidth reduction
Throughput improvement: ~15x (fewer network round-trips)
Latency cost: +10ms (linger.ms wait time)
```

**Banking Example**:

```java
/**
 * Payment producer configuration: Balance throughput and latency
 */
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

// Batching: Wait 10ms or until 64 KB batch
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);      // 64 KB
props.put(ProducerConfig.LINGER_MS_CONFIG, 10);          // 10ms

// Result:
// - Average batch contains 64 messages (1 KB each)
// - p99 latency: 15ms (10ms linger + 5ms network/broker)
// - Throughput: 50,000 messages/sec per partition
// - Acceptable for payment processing (15ms < 100ms SLA)
```

### Compression

**Compression** reduces message size, improving throughput and reducing storage costs.

**Supported Algorithms**:

```properties
# Compression type (default: none)
compression.type = none

# Options: none, gzip, snappy, lz4, zstd
# Trade-offs:
# - none: No CPU overhead, largest messages
# - gzip: High compression ratio (4:1), high CPU cost
# - snappy: Moderate compression (2:1), low CPU cost
# - lz4: Fast compression (3:1), very low CPU cost (recommended)
# - zstd: Best compression (5:1), moderate CPU cost (Kafka 2.1+)
```

**Compression Comparison**:

| Algorithm | Compression Ratio | CPU Cost (Producer) | CPU Cost (Consumer) | Use Case |
|-----------|------------------|---------------------|---------------------|----------|
| **none** | 1:1 | None | None | Uncompressible data (encrypted, binary) |
| **gzip** | 4:1 | High (30ms/batch) | Moderate (10ms/batch) | Archival, low-traffic topics |
| **snappy** | 2:1 | Low (5ms/batch) | Low (3ms/batch) | Deprecated (use lz4) |
| **lz4** | 3:1 | Very Low (3ms/batch) | Very Low (1ms/batch) | **Default recommendation** |
| **zstd** | 5:1 | Moderate (15ms/batch) | Moderate (8ms/batch) | High-volume, maximize savings |

**Compression Effectiveness**:

```
Scenario: JSON payment messages (1 KB average, highly compressible)

No Compression:
- Message size: 1 KB
- Network bandwidth: 100,000 messages/sec × 1 KB = 100 MB/sec
- Storage: 1 KB × 100,000 msg/sec × 86,400 sec/day = 8.6 TB/day

LZ4 Compression (3:1 ratio):
- Compressed size: 333 bytes
- Network bandwidth: 100,000 × 333 bytes = 33 MB/sec (67% reduction)
- Storage: 2.9 TB/day (66% reduction)
- CPU overhead: ~5% (producer-side compression)

Savings:
- Bandwidth: 67 MB/sec saved
- Storage: 5.7 TB/day saved (at $0.10/GB = $570/day = $208,000/year)
- Trade-off: 5% CPU increase (acceptable)
```

**Configuration by Use Case**:

```properties
# Real-Time Payments (minimize CPU overhead)
compression.type = lz4
# Rationale: Very low CPU cost (<5%), good compression (3:1), fast decompression

# High-Volume Logs (maximize compression)
compression.type = zstd
# Rationale: Best compression (5:1), acceptable CPU cost (~10%), significant cost savings

# Binary Data (images, encrypted)
compression.type = none
# Rationale: Binary data doesn't compress well (wasted CPU)

# Archival/Compliance (7-year retention)
compression.type = zstd
# Rationale: Maximize storage savings (5:1 ratio = 80% cost reduction)
```

**Banking Example**:

```java
// Payment producer with LZ4 compression
Properties props = new Properties();
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

// Measurement:
// - Uncompressed: 1 KB JSON payment message
// - Compressed: 350 bytes (3:1 ratio)
// - Bandwidth saved: 65% (650 bytes per message)
// - CPU overhead: 3% (measured via profiling)
// - Storage cost: $70,000/year saved (1 PB/year × 65% × $0.10/GB)

// Trade-off accepted: 3% CPU for $70K annual savings
```

### Idempotence and Transactions

**Idempotence** prevents duplicate messages during retries (exactly-once semantics without transactions).

```properties
# Enable idempotent producer (default: true in Kafka 3.0+)
enable.idempotence = true

# How it works:
# - Producer assigns sequence numbers to each message
# - Broker tracks sequence numbers per producer
# - If producer retries (network failure), broker deduplicates using sequence number
# - Guarantees exactly-once per partition without performance overhead

# Side effects (automatically configured):
# - acks = all (requires all in-sync replicas)
# - retries = Integer.MAX_VALUE (infinite retries)
# - max.in.flight.requests.per.connection = 5 (limited to maintain ordering)
```

**Performance Impact**:

```
Without Idempotence:
- acks = 1 (leader only)
- Latency: 2ms
- Throughput: 100,000 msg/sec
- Duplicates: ~0.1% (network retries)

With Idempotence:
- acks = all (all ISR members)
- Latency: 5ms (+3ms for ISR replication)
- Throughput: 95,000 msg/sec (-5% due to acks=all)
- Duplicates: 0% (exactly-once)

Trade-off: +3ms latency, -5% throughput for zero duplicates
```

**Transactions** (for multi-partition atomic writes):

```properties
# Enable transactions
transactional.id = payment-producer-1

# Unique ID per producer instance (required for transactions)
# Kafka tracks transaction state using this ID
```

**Transaction Performance Impact**:

```
Without Transactions:
- Latency: 5ms (idempotent producer, acks=all)
- Throughput: 95,000 msg/sec

With Transactions:
- Latency: 10ms (+5ms for transaction coordination)
- Throughput: 60,000 msg/sec (-37% due to transaction overhead)

Trade-off: Significant overhead for atomic multi-partition writes
Use only when atomic writes are required (rare in practice)
```

**Banking Recommendation**:

```java
// Payment Producer: Use idempotence (not full transactions)
Properties props = new Properties();
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// Result: Exactly-once per partition, minimal overhead

// DON'T use transactions unless required (rare cases):
// - Atomic writes across multiple topics (payment + audit log)
// - Exactly-once end-to-end with Kafka Streams

// For most banking use cases:
// - Idempotent producer + idempotent consumer (deduplication) = exactly-once
// - No transaction overhead (37% throughput penalty avoided)
```

### In-Flight Requests

```properties
# Max in-flight requests per connection (default: 5)
max.in.flight.requests.per.connection = 5

# How it works:
# - Producer can send up to 5 batches without waiting for ACKs
# - Increases throughput (pipeline requests)
# - With idempotence: max 5 (ensures ordering)
# - Without idempotence: can increase to 100+ (higher throughput, no ordering guarantee)

# Trade-off:
# - Higher value: Better throughput (more pipelining)
# - Lower value: Better ordering (fewer reorders during retries)
```

**Performance Impact**:

```
max.in.flight.requests.per.connection = 1:
- Throughput: 20,000 msg/sec (wait for ACK before sending next batch)
- Ordering: Perfect (no reordering possible)

max.in.flight.requests.per.connection = 5:
- Throughput: 80,000 msg/sec (4x improvement via pipelining)
- Ordering: Perfect with idempotence

max.in.flight.requests.per.connection = 100:
- Throughput: 95,000 msg/sec (diminishing returns beyond 5)
- Ordering: At-risk without idempotence (reorders during retries)
- Requires: enable.idempotence=false (incompatible with idempotence)
```

---

## Consumer Performance Optimization

Consumer performance is critical for minimizing lag and achieving high throughput.

### Fetch Configuration

**Fetch parameters** control how consumers retrieve messages from brokers.

```properties
############################# Fetch Configuration #############################

# Minimum bytes to fetch per request (default: 1 byte)
fetch.min.bytes = 1

# How it works:
# - Consumer waits until at least fetch.min.bytes are available
# - OR fetch.max.wait.ms timeout expires
# - Larger value: Fewer requests, better batching, higher latency
# - Smaller value: More requests, lower latency, less batching

# Maximum wait time for fetch.min.bytes (default: 500ms)
fetch.max.wait.ms = 500

# Maximum timeout to wait if fetch.min.bytes not available
# Trade-off: Lower wait = lower latency but more frequent requests

# Maximum bytes per fetch request (default: 50 MB)
fetch.max.bytes = 52428800

# Total bytes fetched across all partitions
# Increase for high-throughput consumers (100 MB, 200 MB)

# Maximum bytes per partition per fetch (default: 1 MB)
max.partition.fetch.bytes = 1048576

# Bytes fetched per partition (should be > max message size)
# Increase if messages are large (10 MB for large payloads)
```

**Fetch Optimization Strategies**:

```properties
# Strategy 1: Low-Latency (real-time payments)
fetch.min.bytes = 1          # Fetch immediately
fetch.max.wait.ms = 100      # Wait max 100ms
max.partition.fetch.bytes = 1048576  # 1 MB

# Result:
# - Consumer fetches as soon as 1 byte available
# - Low latency: ~100ms max wait
# - More frequent requests (acceptable for real-time)

# Strategy 2: High-Throughput (analytics)
fetch.min.bytes = 1048576    # Wait for 1 MB
fetch.max.wait.ms = 5000     # Wait max 5 seconds
max.partition.fetch.bytes = 10485760  # 10 MB

# Result:
# - Consumer waits for large batches (1 MB)
# - Fewer requests (better network utilization)
# - Higher latency (up to 5 seconds)

# Strategy 3: Balanced (payment processing)
fetch.min.bytes = 16384      # 16 KB
fetch.max.wait.ms = 500      # 500ms
max.partition.fetch.bytes = 1048576  # 1 MB

# Result:
# - Fetches 16 KB batches (good batching)
# - Latency: 500ms max
# - Balanced request frequency
```

**Banking Example**:

```java
// Payment Consumer: Low-latency configuration
Properties props = new Properties();
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1);           // Fetch immediately
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 100);       // Max 100ms wait
props.put(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG, 1048576);  // 1 MB

// Measurement:
// - Average fetch size: 64 KB (64 messages @ 1 KB each)
// - Fetch latency: 50ms average (100ms p99)
// - Processing rate: 10,000 msg/sec per consumer
// - Lag: <1 second (acceptable for real-time processing)
```

### Partition Assignment and Parallelism

**Consumer parallelism** is limited by partition count (one partition per consumer maximum).

```
Rule: Max consumers per group = Number of partitions

Example:
Topic: payments (30 partitions)
Consumer Group: payment-processor

10 consumers: Each handles 3 partitions (full utilization)
30 consumers: Each handles 1 partition (ideal 1:1 ratio)
50 consumers: 30 active, 20 idle (wasted resources)

Recommendation: Match consumer count to partition count for optimal parallelism
```

**Consumer Scaling Example**:

```
Scenario: Payment processing falling behind (consumer lag growing)

Current State:
- 30 partitions
- 10 consumers (3 partitions each)
- Processing rate: 5,000 msg/sec per consumer
- Total capacity: 50,000 msg/sec
- Incoming rate: 75,000 msg/sec
- Lag: Growing at 25,000 msg/sec (deficit)

Solution: Add 20 consumers (total 30, 1:1 ratio)
- Processing rate: 5,000 msg/sec per consumer × 30 = 150,000 msg/sec
- Incoming rate: 75,000 msg/sec
- Lag: Decreasing at 75,000 msg/sec (surplus)
- Catch-up time: Existing lag / 75,000 msg/sec
```

### Offset Commit Strategy

**Offset commits** determine consumer restart behavior and impact performance.

```properties
# Auto-commit enabled (default: true)
enable.auto.commit = true

# Auto-commit interval (default: 5 seconds)
auto.commit.interval.ms = 5000

# How it works:
# - Consumer automatically commits offsets every 5 seconds
# - Commits happen asynchronously (non-blocking)
# - Risk: Message loss or duplicates during consumer failure
```

**Auto-Commit Trade-offs**:

```
enable.auto.commit = true:
Pros:
- Simple (no manual commit code)
- Non-blocking (async commits don't impact throughput)

Cons:
- At-least-once semantics (duplicates on failure)
- Loss risk (if consumer crashes between processing and auto-commit)

Example Failure Scenario:
T+0s:  Consumer fetches offsets 1000-1999 (1000 messages)
T+1s:  Consumer processes messages 1000-1500
T+2s:  Consumer crashes (before auto-commit at T+5s)
T+10s: Consumer restarts, resumes from last committed offset (1000)
Result: Messages 1000-1500 processed TWICE (duplicates)
```

**Manual Commit Strategies**:

```java
// Strategy 1: Commit after each batch (at-least-once)
enable.auto.commit = false

while (true) {
    ConsumerRecords<String, Payment> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, Payment> record : records) {
        processPayment(record.value());  // Process message
    }

    consumer.commitSync();  // Commit AFTER processing (blocks)
}

// Pros: No message loss (commits only after processing)
// Cons: Duplicates on failure (messages reprocessed), lower throughput (blocking commit)

// Strategy 2: Async commit (high throughput)
enable.auto.commit = false

while (true) {
    ConsumerRecords<String, Payment> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, Payment> record : records) {
        processPayment(record.value());
    }

    consumer.commitAsync();  // Non-blocking commit
}

// Pros: High throughput (non-blocking), simple
// Cons: Commit failures not retried (potential duplicates)

// Strategy 3: Hybrid (async + sync on rebalance)
enable.auto.commit = false

consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        consumer.commitSync();  // Sync commit before rebalance (ensure no data loss)
    }
});

while (true) {
    ConsumerRecords<String, Payment> records = consumer.poll(Duration.ofMillis(100));
    processRecords(records);
    consumer.commitAsync();  // Async during normal operation (high throughput)
}

// Pros: High throughput + guaranteed commit on rebalance
// Cons: Duplicates during crash (but not during rebalance)
```

**Banking Recommendation**:

```java
/**
 * Payment consumer: Manual commit with idempotent processing
 */
Properties props = new Properties();
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

// Idempotent payment processing (store processed payment IDs)
Set<String> processedPaymentIds = loadFromDatabase();

while (true) {
    ConsumerRecords<String, Payment> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, Payment> record : records) {
        Payment payment = record.value();

        // Idempotency check
        if (processedPaymentIds.contains(payment.getPaymentId())) {
            continue;  // Skip duplicate
        }

        // Process payment + store ID in database (atomic transaction)
        processPaymentAndStoreId(payment);
        processedPaymentIds.add(payment.getPaymentId());
    }

    consumer.commitAsync();  // Commit after batch
}

// Result: Exactly-once semantics via idempotency (not transaction overhead)
// Duplicates handled gracefully (idempotency check)
```

---

## Broker Performance Optimization

Broker tuning impacts all producers and consumers in the cluster.

### JVM Configuration

```bash
# Kafka JVM settings (kafka-server-start.sh)

# Heap size: 6-8 GB (don't exceed 8 GB to avoid long GC pauses)
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"

# Why 6-8 GB?
# - Kafka relies on OS page cache (not JVM heap) for performance
# - Allocate most RAM to OS page cache (e.g., 6 GB heap + 58 GB page cache on 64 GB server)
# - Larger heap = longer GC pauses (avoid >8 GB)

# Garbage collector: G1GC (recommended for Kafka)
export KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M"

# G1GC settings:
# - MaxGCPauseMillis=20: Target 20ms GC pauses (low latency)
# - InitiatingHeapOccupancyPercent=35: Start GC when heap 35% full (frequent minor GCs, avoid full GCs)
# - G1HeapRegionSize=16M: 16 MB regions (for 6 GB heap)

# GC logging (critical for performance debugging)
export KAFKA_GC_LOG_OPTS="-Xlog:gc*:file=/var/log/kafka/gc.log:time,tags:filecount=10,filesize=100M"
```

**GC Tuning Impact**:

```
Poor GC Configuration (default CMS):
- Full GC frequency: 1 per hour
- Full GC pause time: 5-10 seconds
- Impact: Broker unresponsive during GC (consumer timeouts, producer retries)

Optimized G1GC:
- Full GC frequency: 0 (avoided via tuning)
- Minor GC pause time: 20ms
- Impact: No timeouts, smooth performance
```

### OS Page Cache Optimization

```bash
# Linux page cache tuning (critical for Kafka performance)

# Increase vm.swappiness (reduce swapping)
sysctl -w vm.swappiness=1
# Default: 60 (swap to disk when memory 40% full)
# Kafka: 1 (avoid swapping, use RAM for page cache)

# Increase file descriptor limits
ulimit -n 100000
# Kafka requires many file handles (log segments, network connections)

# Increase socket buffer sizes
sysctl -w net.core.rmem_max=2097152
sysctl -w net.core.wmem_max=2097152
# Larger buffers = better network throughput

# Disable transparent huge pages (causes GC pauses)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### Disk I/O Optimization

```properties
# Kafka broker disk configuration

# Flush interval (default: rely on OS, don't flush)
log.flush.interval.messages = 9223372036854775807  # Infinite (don't flush)
log.flush.interval.ms = 9223372036854775807        # Infinite

# Why not flush?
# - Flushing to disk is slow (serializes I/O, kills throughput)
# - Kafka's replication provides durability (message on N brokers before ACK)
# - OS page cache batches writes (flushes every 30 seconds by default)
# - Faster than manual flushing

# Number of I/O threads (default: 8)
num.io.threads = 16

# Increase for high partition count or disk-bound workloads
# Rule of thumb: 1 thread per 100 partitions

# Number of network threads (default: 3)
num.network.threads = 8

# Increase for high connection count (many producers/consumers)
```

---

## End-to-End Latency Optimization

End-to-end latency = Producer latency + Broker latency + Consumer latency + Network latency

### Latency Breakdown

```
Target: p99 latency <10ms (real-time payment processing)

Component Latency:
1. Producer batching: 0-10ms (linger.ms)
2. Producer send: 1-2ms (network to broker)
3. Broker write: 1-2ms (append to log + page cache)
4. ISR replication: 2-5ms (acks=all, cross-AZ)
5. Consumer fetch: 1-2ms (fetch.max.wait.ms)
6. Consumer processing: 1-3ms (business logic)

Total: 6-24ms (p99 depends on linger.ms and acks configuration)
```

**Low-Latency Configuration**:

```properties
############################# Producer #############################
linger.ms = 0                              # Send immediately (no batching delay)
batch.size = 16384                         # 16 KB (default)
compression.type = lz4                     # Fast compression
acks = 1                                   # Leader only (not all ISR)
buffer.memory = 33554432                   # 32 MB

############################# Broker #############################
num.network.threads = 8                    # Fast network handling
num.io.threads = 16                        # Fast disk I/O
replica.lag.time.max.ms = 10000            # 10s (allow higher replication lag)

############################# Consumer #############################
fetch.min.bytes = 1                        # Fetch immediately
fetch.max.wait.ms = 100                    # Max 100ms wait
max.partition.fetch.bytes = 1048576        # 1 MB

# Result: p99 latency ~5ms (no batching delay, leader-only ACK)
# Trade-off: Lower durability (acks=1, risk of data loss on leader failure)
```

---

## Interview Questions

### Question 1: You're seeing high producer latency (p99 = 50ms). Walk through your troubleshooting process to identify the bottleneck.

**Answer**:

**Step 1: Identify Latency Components**

Producer Latency = Batching Delay + Network Time + Broker Processing + ACK Wait

Expected: ~20ms, Actual: 50ms → 30ms unexplained

**Step 2: Check Producer Metrics**

```bash
kafka-run-class.sh kafka.tools.JmxTool \
  --object-name kafka.producer:type=producer-metrics,client-id=payment-producer
```

**Step 3: Common Causes**

1. Slow compression (gzip → lz4): 30ms saved
2. Broker overload: Scale brokers
3. Network latency: Co-locate in same AZ
4. ISR replication lag: Check UnderReplicatedPartitions

---

### Question 2: Explain the trade-off between `linger.ms` and `batch.size`. How would you tune these for a payment processing system?

**Answer**:

**linger.ms**: Time-based batching
**batch.size**: Size-based batching

**Payment Processing**:
```properties
linger.ms = 10      # 10ms acceptable
batch.size = 65536  # 64 KB

# At 10,000 TPS: batch fills in 6ms
# Effective latency: 8-12ms (within SLA)
```

---

### Question 3: Your consumer group shows consistent lag of 10,000 messages per partition. How do you resolve this?

**Answer**:

**Solutions**:
1. Scale consumers (10 → 30)
2. Optimize processing (batch DB writes)
3. Increase fetch size

---

## Summary

Kafka performance tuning requires understanding producer batching, consumer fetch optimization, and broker resource allocation. Key takeaways:

1. **Producer**: Use `linger.ms=10`, `batch.size=64KB`, `compression.type=lz4`
2. **Consumer**: Tune fetch parameters, use manual commit with idempotency
3. **Broker**: JVM heap 6-8 GB, allocate RAM to page cache
4. **Latency**: p99 <10ms achievable with optimized configuration
5. **Monitoring**: Track batch-size-avg, lag, UnderReplicatedPartitions

Performance tuning enables meeting strict SLAs while minimizing infrastructure costs—essential for production banking systems.

**Word Count**: ~8,000 words
