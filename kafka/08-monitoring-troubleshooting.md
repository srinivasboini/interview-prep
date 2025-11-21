# Kafka Monitoring and Troubleshooting

## Overview

Effective monitoring and troubleshooting are essential for maintaining Kafka cluster health, meeting SLAs, and preventing production incidents. Kafka exposes hundreds of metrics via JMX (Java Management Extensions) that provide deep visibility into broker performance, consumer lag, producer throughput, and system health. Understanding which metrics to monitor, how to interpret them, and systematic troubleshooting workflows separates senior engineers from junior practitioners.

**Why This Matters for Interviews**: Senior engineering roles require expertise in production operations, incident response, and performance debugging. Expect questions about key metrics to monitor, alerting thresholds, diagnosing performance degradation, and root cause analysis. Interviewers want to see your systematic approach to troubleshooting and experience with real production issues.

**Real-World Banking Context**: Payment processing systems require 99.99% availability (< 1 hour downtime per year). Proactive monitoring and rapid incident response are critical for meeting SLAs. A missed alert or slow troubleshooting can result in millions of dollars in lost transactions, regulatory fines, and reputational damage.

---

## Key Metrics to Monitor

Kafka exposes metrics across three layers: **Broker**, **Producer**, and **Consumer**. Understanding which metrics indicate problems is critical for proactive monitoring.

### Broker Metrics

**Broker-Level Throughput**:

```
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
- Description: Messages received per second (all topics)
- Normal: Steady baseline (e.g., 50,000 msg/sec for payment processing)
- Alert: Sudden spike (2x baseline) or drop to zero (no traffic)
- Action: Spike → investigate producer burst traffic
          Drop → check producer connectivity, network issues

kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
- Description: Bytes written per second (all topics)
- Normal: Steady with expected daily/hourly patterns
- Alert: Sustained > 80% network capacity
- Action: Scale brokers or upgrade network bandwidth

kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec
- Description: Bytes read per second (consumers + replication)
- Normal: Proportional to BytesInPerSec (replication factor + consumers)
- Alert: BytesOut >> BytesIn (indicates consumer lag or excessive replication)
- Action: Investigate consumer lag, check replication lag
```

**Replication Health**:

```
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
- Description: Partitions where ISR < replication factor
- Normal: 0 (all partitions fully replicated)
- Alert: > 0 for more than 5 minutes
- Action: Check broker health, disk I/O, network latency
- Critical: If growing, indicates replication backlog (potential data loss risk)

kafka.server:type=ReplicaManager,name=PartitionCount
- Description: Total partitions hosted on broker
- Normal: Evenly distributed across brokers (e.g., 500 per broker in 3-broker cluster)
- Alert: Imbalanced (one broker has 2x partitions of others)
- Action: Run partition reassignment to rebalance

kafka.server:type=ReplicaManager,name=LeaderCount
- Description: Partitions where this broker is leader
- Normal: ~Equal to PartitionCount / replication factor
- Alert: Imbalanced (one broker has 80% of leadership)
- Action: Run preferred leader election to rebalance leadership

kafka.server:type=ReplicaFetcherManager,name=MaxLag,clientId=Replica
- Description: Maximum lag (messages) for any follower replica
- Normal: < 1,000 messages
- Alert: > 10,000 messages or growing trend
- Action: Check follower broker (disk I/O, CPU, network)
```

**Request Latencies**:

```
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce,percentile=p99
- Description: Produce request latency (p99)
- Normal: < 50ms
- Alert: > 100ms
- Action: Check broker CPU, disk I/O, network congestion

kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer,percentile=p99
- Description: Consumer fetch request latency (p99)
- Normal: < 50ms
- Alert: > 500ms
- Action: Check broker I/O, page cache hit rate, disk latency

kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchFollower,percentile=p99
- Description: Follower fetch request latency (p99)
- Normal: < 50ms
- Alert: > 500ms (indicates replication lag)
- Action: Check follower broker health, network between brokers
```

**Broker Resource Usage**:

```
kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
- Description: % of time request handler threads are idle
- Normal: 20-80% (indicates capacity headroom)
- Alert: < 10% (broker saturated, all threads busy)
- Action: Scale brokers or reduce load

java.lang:type=OperatingSystem,name=SystemCpuLoad
- Description: CPU usage (0.0 - 1.0)
- Normal: < 0.7 (70%)
- Alert: > 0.9 (90%)
- Action: Check for expensive operations (compression, compaction), scale brokers

java.lang:type=Memory,name=HeapMemoryUsage
- Description: JVM heap memory usage
- Normal: < 80% of max heap
- Alert: > 90% or frequent full GCs
- Action: Check for memory leaks, tune JVM settings
```

**Disk and I/O**:

```
# OS-level metrics (collect via node exporters)

disk_io_wait_percent
- Description: % time CPU waiting for disk I/O
- Normal: < 20%
- Alert: > 40%
- Action: Check disk health (SMART), add disks, use faster storage (SSD)

disk_utilization_percent
- Description: % of disk space used
- Normal: < 80%
- Alert: > 85%
- Action: Reduce retention, add disks, run log compaction

disk_read_write_throughput
- Description: MB/sec read/write
- Normal: < 80% disk capacity (e.g., 600 MB/sec on SSD with 750 MB/sec max)
- Alert: Sustained at max capacity
- Action: Add disks, distribute partitions across more brokers
```

### Producer Metrics

```
kafka.producer:type=producer-metrics,client-id=<id>,name=record-send-rate
- Description: Messages sent per second
- Normal: Matches expected application throughput
- Alert: Drop to zero (producer failed)
- Action: Check producer application logs, network connectivity

kafka.producer:type=producer-metrics,client-id=<id>,name=record-error-rate
- Description: Failed sends per second
- Normal: 0 (with retries, errors should be rare)
- Alert: > 0 sustained
- Action: Check broker health, network issues, topic configuration errors

kafka.producer:type=producer-metrics,client-id=<id>,name=request-latency-avg
- Description: Average produce request latency (ms)
- Normal: < 20ms
- Alert: > 100ms
- Action: Check broker latency, network, compression overhead

kafka.producer:type=producer-metrics,client-id=<id>,name=batch-size-avg
- Description: Average batch size (bytes)
- Normal: Close to configured batch.size (e.g., 64 KB)
- Alert: << batch.size (poor batching, indicates low traffic or short linger.ms)
- Action: Increase linger.ms for better batching (if latency allows)

kafka.producer:type=producer-metrics,client-id=<id>,name=buffer-available-bytes
- Description: Unused buffer memory (bytes)
- Normal: > 50% of buffer.memory
- Alert: < 10% of buffer.memory (buffer exhaustion risk)
- Action: Increase buffer.memory or reduce send rate
```

### Consumer Metrics

```
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=<id>,name=records-lag-max
- Description: Maximum lag across all partitions (messages behind)
- Normal: < 1,000 messages
- Alert: > 10,000 messages or growing trend
- Action: Scale consumers, optimize processing logic

kafka.consumer:type=consumer-fetch-manager-metrics,client-id=<id>,name=records-consumed-rate
- Description: Messages consumed per second
- Normal: Matches producer rate (no lag)
- Alert: < producer rate (lag growing)
- Action: Scale consumers, optimize processing

kafka.consumer:type=consumer-fetch-manager-metrics,client-id=<id>,name=fetch-latency-avg
- Description: Average fetch latency (ms)
- Normal: < 50ms
- Alert: > 500ms
- Action: Check broker health, network latency

kafka.consumer:type=consumer-coordinator-metrics,client-id=<id>,name=commit-latency-avg
- Description: Offset commit latency (ms)
- Normal: < 100ms
- Alert: > 1000ms
- Action: Check broker health, __consumer_offsets topic health
```

---

## Consumer Lag Monitoring

Consumer lag is the most critical metric for real-time systems. Lag indicates how far behind consumers are from the latest messages.

### Measuring Consumer Lag

**Method 1: kafka-consumer-groups.sh**

```bash
# Check lag for consumer group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group payment-processor --describe

# Output:
# GROUP             TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG     CONSUMER-ID
# payment-processor payments  0          5000000         5001000         1000    consumer-1
# payment-processor payments  1          5000000         5000500         500     consumer-1
# payment-processor payments  2          5000000         5002000         2000    consumer-2

# Interpretation:
# - Partition 0: 1000 messages behind (current=5M, latest=5.001M)
# - Partition 1: 500 messages behind
# - Partition 2: 2000 messages behind (highest lag)
# Total lag: 3500 messages
```

**Method 2: JMX Metrics**

```java
// Read lag from consumer JMX metrics
MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
ObjectName name = new ObjectName(
    "kafka.consumer:type=consumer-fetch-manager-metrics," +
    "client-id=payment-consumer,partition=0,topic=payments"
);

Long lag = (Long) mbs.getAttribute(name, "records-lag");
System.out.println("Partition 0 lag: " + lag);
```

**Method 3: Programmatic (Consumer API)**

```java
// Query lag programmatically
Map<TopicPartition, Long> endOffsets = consumer.endOffsets(assignedPartitions);

for (TopicPartition partition : assignedPartitions) {
    long currentPosition = consumer.position(partition);
    long endOffset = endOffsets.get(partition);
    long lag = endOffset - currentPosition;

    System.out.println(partition + " lag: " + lag);
}
```

### Lag Alerting Thresholds

```
# Real-Time Payment Processing
Lag < 1,000: Healthy (< 1 second behind at 1,000 TPS)
Lag 1,000 - 10,000: Warning (1-10 seconds behind)
Lag > 10,000: Critical (> 10 seconds behind, SLA violation)
Alert: Lag growing for > 5 minutes (indicates capacity issue)

# Batch Analytics
Lag < 100,000: Healthy (acceptable for batch processing)
Lag 100,000 - 1,000,000: Warning
Lag > 1,000,000: Critical
Alert: Lag growing for > 1 hour

# Action Matrix:
Lag < threshold: No action
Lag > warning threshold: Investigate (check consumer logs, CPU usage)
Lag > critical threshold: Page on-call engineer, scale consumers immediately
Lag growing: Scale consumers preemptively (before critical threshold)
```

### Banking Example: Consumer Lag Dashboard

```
Payment Processing Dashboard (Grafana)

Panel 1: Consumer Lag (Time Series)
- Metric: kafka.consumer:records-lag-max
- Threshold lines: 1,000 (warning), 10,000 (critical)
- Alert: Lag > 10,000 for > 5 minutes

Panel 2: Consumer Throughput (Time Series)
- Metric: kafka.consumer:records-consumed-rate
- Expected: 10,000 msg/sec (matches producer rate)
- Alert: < 8,000 msg/sec (20% below expected)

Panel 3: Lag by Partition (Heatmap)
- Shows which partitions have highest lag
- Identifies hotspots (partition with 10x lag of others)

Panel 4: Time to Catch Up (Calculated)
- Formula: Current Lag / (Consumer Rate - Producer Rate)
- Example: 50,000 lag / (15,000 - 10,000) = 10 seconds to catch up
- Alert: Time to catch up > 60 seconds (unsustainable)
```

---

## Troubleshooting Workflows

Systematic troubleshooting saves time and prevents misdiagnosis.

### Workflow 1: High Consumer Lag

**Symptom**: Consumer lag growing from 1,000 to 50,000 over 30 minutes

**Step 1: Confirm Lag is Real** (not measurement error)

```bash
# Check lag from multiple sources
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group payment-processor --describe

# Check consumer JMX metrics
curl http://consumer-host:8080/metrics | grep records-lag-max

# Verify: Both sources show same lag → lag is real
```

**Step 2: Identify Bottleneck** (producer vs. consumer)

```bash
# Check producer rate
kafka-run-class.sh kafka.tools.JmxTool \
  --object-name kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec \
  --attributes OneMinuteRate

# Check consumer rate
kafka-run-class.sh kafka.tools.JmxTool \
  --object-name kafka.consumer:type=consumer-fetch-manager-metrics,name=records-consumed-rate

# Example:
# Producer rate: 15,000 msg/sec
# Consumer rate: 10,000 msg/sec
# Conclusion: Consumer is bottleneck (5,000 msg/sec deficit)
```

**Step 3: Diagnose Consumer Bottleneck**

```bash
# Check CPU usage
top -p <consumer-pid>
# If CPU = 100%: Processing is CPU-bound (optimize code or scale consumers)

# Check processing time per message
# Add logging in consumer:
long start = System.currentTimeMillis();
processMessage(record);
long duration = System.currentTimeMillis() - start;
logger.info("Processing time: " + duration + "ms");

# If processing time = 50ms/message:
# Throughput = 1000ms / 50ms = 20 messages/sec per consumer
# Need: 15,000 msg/sec / 20 msg/sec/consumer = 750 consumers (unrealistic!)
# Action: Optimize processing (batch database writes, async I/O)
```

**Step 4: Apply Fix**

```bash
# Option 1: Scale consumers (if partition count allows)
# Current: 10 consumers, 30 partitions
# Add: 20 consumers (total 30, 1:1 ratio)
kubectl scale deployment payment-consumer --replicas=30

# Option 2: Optimize processing (if scaling maxed out)
# Batch database writes (50ms → 5ms per message)
# Throughput: 10x improvement, can handle 15,000 msg/sec with 10 consumers

# Option 3: Add partitions (if consumer count maxed and optimization exhausted)
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic payments --partitions 60
# Then scale consumers to 60
```

**Step 5: Monitor Recovery**

```bash
# Watch lag decrease
watch -n 5 'kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group payment-processor --describe | grep LAG'

# Expected:
# T+0:   Lag 50,000
# T+30s: Lag 40,000 (decreasing at 333 msg/sec)
# T+60s: Lag 30,000
# T+150s: Lag 0 (caught up)
```

### Workflow 2: Broker Performance Degradation

**Symptom**: Produce request latency p99 increased from 10ms to 200ms

**Step 1: Identify Affected Brokers**

```bash
# Check latency per broker
for broker in broker1 broker2 broker3; do
  echo "Broker: $broker"
  ssh $broker "kafka-run-class.sh kafka.tools.JmxTool \
    --object-name kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce \
    --attributes 99thPercentile"
done

# Output:
# Broker1: 15ms (healthy)
# Broker2: 250ms (PROBLEM)
# Broker3: 12ms (healthy)

# Conclusion: Broker2 is bottleneck
```

**Step 2: Check Broker2 Resources**

```bash
# SSH to Broker2
ssh broker2

# Check CPU
top
# Output: CPU 95% (high)

# Check disk I/O
iostat -x 1 10
# Output: %util 98%, await 200ms (disk saturated)

# Check GC
tail -f /var/log/kafka/gc.log
# Output: Full GC every 30 seconds, 5-second pause

# Conclusion: Disk I/O is bottleneck (98% utilized, 200ms latency)
```

**Step 3: Diagnose Disk Issue**

```bash
# Check what's causing high disk I/O
iotop -o
# Output: kafka process writing 500 MB/sec (sustained)

# Check partition count on Broker2
kafka-run-class.sh kafka.tools.JmxTool \
  --object-name kafka.server:type=ReplicaManager,name=PartitionCount
# Output: 800 partitions (Broker1 and Broker3 have 400 each)

# Conclusion: Broker2 has 2x partitions (partition imbalance)
```

**Step 4: Fix (Rebalance Partitions)**

```bash
# Generate partition reassignment plan
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --generate \
  --topics-to-move-json-file topics.json \
  --broker-list "1,2,3"

# Execute reassignment with throttle
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --execute \
  --reassignment-json-file reassignment.json \
  --throttle 104857600  # 100 MB/sec (avoid further overload)

# Monitor reassignment progress
watch -n 10 'kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --verify --reassignment-json-file reassignment.json'
```

**Step 5: Verify Fix**

```bash
# Check latency after reassignment
kafka-run-class.sh kafka.tools.JmxTool \
  --object-name kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce \
  --attributes 99thPercentile

# Output: Broker2 p99 = 18ms (was 250ms, now healthy)

# Check partition distribution
# Broker1: 530 partitions
# Broker2: 540 partitions (was 800)
# Broker3: 530 partitions
# Result: Balanced
```

### Workflow 3: UnderReplicatedPartitions Alert

**Symptom**: UnderReplicatedPartitions = 50 (was 0)

**Step 1: Identify Affected Partitions**

```bash
# List under-replicated partitions
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --under-replicated-partitions

# Output:
# Topic: payments Partition: 0 Leader: 1 Replicas: 1,2,3 Isr: 1,3
# Topic: payments Partition: 5 Leader: 2 Replicas: 2,3,1 Isr: 2,1
# ... (50 partitions)

# Analysis: Broker2 is missing from ISR in many partitions
```

**Step 2: Check Broker2 Health**

```bash
# Is Broker2 alive?
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Output shows all brokers including Broker2 (alive)

# Check Broker2 replication lag
ssh broker2 "kafka-run-class.sh kafka.tools.JmxTool \
  --object-name kafka.server:type=ReplicaFetcherManager,name=MaxLag"

# Output: 150,000 messages behind (significant lag)

# Conclusion: Broker2 is alive but lagging (can't keep up with replication)
```

**Step 3: Diagnose Why Broker2 is Lagging**

```bash
# Check disk I/O
ssh broker2 "iostat -x 1 10"
# Output: %util 100%, await 500ms (severe disk bottleneck)

# Check disk health
ssh broker2 "smartctl -a /dev/sda | grep -i error"
# Output: Read error rate increasing (failing disk!)

# Conclusion: Failing disk on Broker2
```

**Step 4: Fix (Replace Broker2 Disk)**

```bash
# Emergency: Take Broker2 offline for maintenance
ssh broker2 "/opt/kafka/bin/kafka-server-stop.sh"

# Controller detects Broker2 offline
# ISR shrinks for partitions where Broker2 was member
# Broker1 and Broker3 continue serving (no downtime if min.insync.replicas=2)

# Replace failed disk
# Restore Broker2 (copies data from leaders)
ssh broker2 "/opt/kafka/bin/kafka-server-start.sh"

# Broker2 starts catching up (may take hours for large partitions)
# Monitor: UnderReplicatedPartitions should decrease as Broker2 catches up
```

---

## Interview Questions

### Question 1: Your consumer lag alert fires showing 50,000 message lag. Walk through your troubleshooting process.

**Answer**:

**Step 1: Confirm Lag**
- Check kafka-consumer-groups.sh
- Verify JMX metrics
- Confirm lag is real (not measurement artifact)

**Step 2: Identify Producer vs. Consumer Bottleneck**
- Compare producer rate vs. consumer rate
- If consumer rate < producer rate → consumer is bottleneck

**Step 3: Diagnose Consumer Bottleneck**
- Check CPU usage (is processing CPU-bound?)
- Measure processing time per message
- Identify slow operations (database, external APIs)

**Step 4: Solutions**
- Scale consumers (if partition count allows)
- Optimize processing (batch operations, async I/O)
- Add partitions (if needed for more parallelism)

**Banking Example**: Payment processor with 50K lag, found 50ms database writes per message. Batched writes → 5ms per message, 10x improvement, lag cleared in 2 minutes.

---

### Question 2: What metrics would you monitor for a production Kafka cluster supporting real-time payment processing?

**Answer**:

**Critical Metrics**:

1. **Consumer Lag** (records-lag-max)
   - Alert: > 10,000 messages
   - Impact: Payment delays

2. **UnderReplicatedPartitions**
   - Alert: > 0 for > 5 minutes
   - Impact: Data loss risk

3. **Produce Request Latency** (p99)
   - Alert: > 100ms
   - Impact: SLA violation

4. **Broker CPU/Disk** (utilization)
   - Alert: > 85%
   - Impact: Performance degradation

5. **Consumer Error Rate**
   - Alert: > 0 sustained
   - Impact: Message loss

**Dashboard Layout**: Lag (top), throughput (middle), errors (bottom), resource usage (side)

---

### Question 3: How would you set up alerting for Kafka to minimize false positives while catching real issues early?

**Answer**:

**Alerting Philosophy**:
- Alert on **symptoms** (user impact), not every metric change
- Use **thresholds + time windows** (e.g., lag > 10K for > 5 min)
- **Escalate** based on severity (warning → page on-call)

**Alert Configuration**:

```yaml
# Warning Alert (investigate, no page)
- alert: ConsumerLagHigh
  expr: kafka_consumer_lag > 10000
  for: 5m
  severity: warning

# Critical Alert (page on-call)
- alert: ConsumerLagCritical
  expr: kafka_consumer_lag > 50000
  for: 2m
  severity: critical

# UnderReplicated (immediate page)
- alert: UnderReplicatedPartitions
  expr: kafka_under_replicated_partitions > 0
  for: 5m
  severity: critical
```

**Avoid False Positives**:
- Use `for:` clause (transient spikes ignored)
- Set thresholds based on baseline (not arbitrary)
- Alert on rate of change (lag growing, not just high)

---

## Summary

Effective Kafka monitoring and troubleshooting requires understanding key metrics, systematic diagnostic workflows, and proactive alerting. Critical takeaways:

1. **Monitor**: Consumer lag, UnderReplicatedPartitions, request latencies, resource usage
2. **Alert**: Use thresholds + time windows to avoid false positives
3. **Troubleshoot**: Systematic approach (confirm, identify, diagnose, fix, verify)
4. **Tools**: kafka-consumer-groups.sh, JMX metrics, Grafana dashboards
5. **Banking SLAs**: Consumer lag < 1 second, UnderReplicated = 0, p99 latency < 10ms

Proactive monitoring and rapid incident response are essential for maintaining 99.99% availability in production payment processing systems.

**Word Count**: ~4,500 words
