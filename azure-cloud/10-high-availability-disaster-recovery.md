# High Availability (HA) & Disaster Recovery (DR)

## Overview
"Everything fails, all the time." — Werner Vogels.
Staff Engineers design systems that embrace failure.
Interviewers will ask: "What is your RPO and RTO?" If you don't know, you fail.

## Foundational Concepts

### HA vs. DR
- **High Availability (HA)**: Keeping the system running *during* a failure (e.g., one server dies, another takes over). Goal: Zero downtime.
- **Disaster Recovery (DR)**: Restoring the system *after* a catastrophic failure (e.g., entire region burns down). Goal: Minimize downtime and data loss.

### RPO & RTO
- **RPO (Recovery Point Objective)**: How much data can you afford to lose? (Time).
  - RPO = 0: Zero data loss (Synchronous replication). Expensive.
  - RPO = 1 hour: Restore from backup taken 1 hour ago. Cheap.
- **RTO (Recovery Time Objective)**: How long can you be down? (Time).
  - RTO = 0: Instant failover (Active-Active). Expensive.
  - RTO = 4 hours: Time to spin up new VMs and restore data.

## Technical Deep Dive

### 1. HA Patterns
- **Availability Zones (AZ)**: 99.99% SLA. Protects against data center failure.
- **Load Balancing**:
  - **Layer 4**: Azure Load Balancer (Zone Redundant).
  - **Layer 7**: App Gateway (Zone Redundant).
- **Stateless Compute**: VMs/Pods should be disposable. State lives in DB/Storage.

### 2. DR Strategies
- **Backup & Restore**: Cheapest. High RTO/RPO.
- **Pilot Light**: Database is replicated (Active-Passive). Compute is off. Spin up compute on disaster.
- **Warm Standby**: Scaled-down version of prod running in secondary region. Scale up on disaster.
- **Active-Active**: Both regions process traffic. Zero RTO. Complex data synchronization.

### 3. Azure Site Recovery (ASR)
Disaster Recovery as a Service (DRaaS).
- Replicates VMs (Hyper-V, VMware, Physical, Azure) to a secondary region.
- **Orchestrated Recovery**: One-click failover plans (Recovery Plans).
- **RPO**: Seconds to minutes.

### 4. Resilience Patterns (Code Level)
- **Retry Pattern**: Transient failures (network blip) should be retried with exponential backoff.
- **Circuit Breaker**: If a service is down, stop calling it. Fail fast.
- **Bulkhead**: Isolate resources so one failure doesn't cascade (e.g., separate thread pools).

## Visual Representations

### Active-Passive DR Architecture (Warm Standby)
```mermaid
graph TB
    subgraph "Traffic Routing"
        TM[Traffic Manager / Front Door]
    end

    subgraph "Primary Region (Active)"
        LB1[Load Balancer]
        VM1[VM Scale Set (10 instances)]
        SQL1[SQL Primary]
    end

    subgraph "Secondary Region (Passive)"
        LB2[Load Balancer]
        VM2[VM Scale Set (1 instance)]
        SQL2[SQL Secondary]
    end

    TM -->|100% Traffic| LB1
    TM -.->|Failover Only| LB2
    
    LB1 --> VM1
    VM1 --> SQL1
    
    LB2 --> VM2
    VM2 --> SQL2
    
    SQL1 -->|Async Replication| SQL2
```

### Circuit Breaker Pattern
```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Failure Threshold Reached
    Open --> HalfOpen: Timeout Expired
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
    
    state Closed {
        Note: Normal Operation
    }
    state Open {
        Note: Fail Fast (No calls)
    }
    state HalfOpen {
        Note: Test if service is back
    }
```

## Configuration Examples

### Azure Site Recovery (ASR) Setup (Conceptual)
1. **Source**: Select VM in East US.
2. **Target**: Select West US.
3. **Replication Policy**: RPO threshold (e.g., 15 mins), Recovery Point retention (24 hours).
4. **Enable Replication**: Azure installs mobility agent, starts initial sync.

### Resilience in C# (Polly)
```csharp
// Retry with Exponential Backoff
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => 
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

await retryPolicy.ExecuteAsync(async () => 
{
    await httpClient.GetAsync("https://api.example.com");
});
```

## Real-World Enterprise Scenarios

### Scenario: Ransomware Attack
**Requirement**: A bank gets hit by ransomware. All data in the primary region is encrypted.
**Solution**: **Azure Backup (Immutable Vault)**.
1. **Backup Vault**: Configured with "Soft Delete" and "Immutability" (WORM).
2. **Attack**: Ransomware tries to delete backups. Fails because of Immutability policy.
3. **Recovery**: Restore VMs from the immutable backup point to a clean VNet (Isolated Recovery Environment).

### Scenario: Region Outage
**Requirement**: East US goes down completely.
**Solution**: **Geo-Redundancy**.
1. **Front Door**: Detects East US is unhealthy (probes fail).
2. **Failover**: Automatically routes traffic to West US.
3. **Database**: Auto-Failover Group triggers (if configured) or manual failover initiated.
4. **RPO**: Potential data loss of ~5 seconds (Async replication).

## Interview Questions & Model Answers

### Q1: What is the difference between Synchronous and Asynchronous replication?
**Answer**:
- **Synchronous**: Write is not committed until it is saved in *both* locations. RPO=0. Latency penalty (speed of light). Used within a region (AZs).
- **Asynchronous**: Write is committed in Primary, then sent to Secondary. RPO>0. No latency penalty. Used across regions (Geo).

### Q2: How do you test your DR plan?
**Answer**:
"A DR plan that isn't tested is just a hope."
- **ASR Test Failover**: Spins up the VMs in the secondary region in an isolated VNet. Does *not* impact production replication.
- **Game Days**: Scheduled days where we intentionally break things (Chaos Engineering) to verify the team's response.

### Q3: Why not just use Active-Active everywhere?
**Answer**:
**Complexity and Cost**.
- **Data Sync**: Bi-directional database replication is extremely hard (conflict resolution).
- **Cost**: You are paying for double the infrastructure.
- **Decision**: Use Active-Active for stateless web tiers. Use Active-Passive for stateful data tiers (unless using Cosmos DB).

## Key Takeaways
- **SLA Math**: 99.9% (App) + 99.9% (DB) = 99.8% (Serial dependency reduces availability).
- **Backup != DR**: Backup is for data corruption/deletion. DR is for infrastructure loss.
- **Chaos Studio**: Use it to inject faults and prove your resilience.

## Further Reading
- [Azure Well-Architected Framework - Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/)
- [Business continuity and disaster recovery (BCDR)](https://learn.microsoft.com/en-us/azure/architecture/framework/resiliency/backup-and-recovery)
