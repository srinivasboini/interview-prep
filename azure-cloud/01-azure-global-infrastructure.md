# Azure Global Infrastructure

## Overview
Azure's global infrastructure is the physical backbone of the cloud platform. For a Staff/Principal Engineer, understanding this is not just about knowing where data centers are, but about **latency**, **data sovereignty**, **compliance**, and **disaster recovery**.

Interviewers ask about this to test your ability to design for:
- **High Availability (HA)**: Can your application survive a data center failure? A region failure?
- **Latency**: Are you placing compute close to users?
- **Compliance**: Is data staying within legal boundaries (e.g., GDPR in Europe)?

## Foundational Concepts

### Geography vs. Region vs. Availability Zone
- **Geography**: A discrete market, typically containing two or more regions, that preserves data residency and compliance boundaries (e.g., "United States", "Europe").
- **Region**: A set of data centers deployed within a latency-defined perimeter and connected through a dedicated regional low-latency network.
- **Availability Zone (AZ)**: Unique physical locations within a region. Each zone is made up of one or more data centers equipped with independent power, cooling, and networking.

## Technical Deep Dive

### 1. Regions and Availability Zones
Not all regions support Availability Zones. "Hero" regions (e.g., East US, West Europe) always do.
- **Zonal Services**: You pin the resource to a specific zone (e.g., VM in Zone 1).
- **Zone-Redundant Services**: The platform automatically replicates across zones (e.g., Zone-redundant SQL Database).
- **Regional Services**: No zone awareness (e.g., Azure AD, Traffic Manager).

### 2. Region Pairs
Each Azure region is paired with another region within the same geography (except Brazil South) to form a **Region Pair**.
- **Physical Isolation**: At least 300 miles separation to reduce the likelihood of simultaneous natural disasters.
- **Platform Replication**: Some services (like GRS Storage) automatically replicate to the paired region.
- **Sequential Updates**: Planned Azure updates are rolled out to paired regions sequentially (not simultaneously) to minimize downtime.

> [!IMPORTANT]
> In a disaster scenario, one region in every pair is prioritized for recovery.

### 3. Sovereign Clouds
Specialized clouds physically and logically isolated from the main Azure commercial cloud.
- **Azure Government (US)**: For US federal, state, local, and tribal governments. Physically isolated regions (e.g., US Gov Virginia).
- **Azure China**: Operated by 21Vianet. Physically separated from global Azure. Data stays in China.
- **Azure Germany**: (Legacy/Retired) Now served by standard German regions with data residency guarantees.

### 4. Azure Edge Zones
- **Azure Edge Zones**: Extensions of Azure placed in population centers.
- **Azure Private Edge Zones**: Private 5G/LTE networks combined with Azure Stack Edge on-premises.

## Visual Representations

### Azure Region & Availability Zone Architecture
```mermaid
graph TB
    subgraph "Geography (e.g., Europe)"
        subgraph "Region Pair"
            subgraph "Region A (e.g., North Europe)"
                AZ1[("Availability Zone 1<br/>(Independent Power/Cooling)")]
                AZ2[("Availability Zone 2<br/>(Independent Power/Cooling)")]
                AZ3[("Availability Zone 3<br/>(Independent Power/Cooling)")]
                
                AZ1 <-->|Low Latency (<2ms)| AZ2
                AZ2 <-->|Low Latency (<2ms)| AZ3
                AZ1 <-->|Low Latency (<2ms)| AZ3
            end
            
            subgraph "Region B (e.g., West Europe)"
                AZ4[("Availability Zone 1")]
                AZ5[("Availability Zone 2")]
                AZ6[("Availability Zone 3")]
            end
            
            RegionA <==>|Asynchronous Replication<br/>(>300 miles)| RegionB
        end
    end
```

### High Availability Architecture Patterns
```mermaid
graph TD
    subgraph "SLA: 99.9%"
        VM_Single[Single VM<br/>(Premium SSD)]
    end
    
    subgraph "SLA: 99.95%"
        AVSet[Availability Set]
        VM1[VM 1]
        VM2[VM 2]
        AVSet --> VM1
        AVSet --> VM2
    end
    
    subgraph "SLA: 99.99%"
        AZs[Availability Zones]
        Zone1[Zone 1]
        Zone2[Zone 2]
        AZs --> Zone1
        AZs --> Zone2
        Zone1 --> VM_Z1[VM]
        Zone2 --> VM_Z2[VM]
    end
    
    subgraph "SLA: 99.99% + DR"
        Region1[Region 1 (Active)]
        Region2[Region 2 (Passive)]
        Region1 -.->|Geo-Replication| Region2
    end
```

## Real-World Enterprise Scenarios

### Scenario 1: Banking Data Residency
**Requirement**: A Swiss bank needs to host customer data. Swiss regulations require data to never leave Switzerland.
**Solution**: Use the **Switzerland North** and **Switzerland West** regions. These are a region pair contained entirely within Switzerland borders.
**Constraint**: You cannot use GRS (Geo-Redundant Storage) if it replicates to a region outside the legal boundary.

### Scenario 2: Low-Latency Trading App
**Requirement**: A high-frequency trading application needs sub-millisecond latency between app and database.
**Solution**:
1. Deploy App and DB in the **same region**.
2. Use **Proximity Placement Groups (PPG)** to physically locate the VMs close to each other within a single data center.
3. Enable **Accelerated Networking**.

## Interview Questions & Model Answers

### Q1: Explain the difference between an Availability Set and an Availability Zone. When would you use one over the other?
**Answer**:
- **Availability Set**: Logical grouping of VMs within a *single* data center. Protects against hardware failures (Fault Domains) and patching reboots (Update Domains). SLA: 99.95%.
- **Availability Zone**: Physical separation of VMs across *different* data centers within a region. Protects against entire data center failures (power, cooling, fire). SLA: 99.99%.
- **Decision**: Always prefer Availability Zones for higher SLA and better resilience. Use Availability Sets only in regions where AZs are not yet supported or for legacy applications requiring Layer 2 adjacency.

### Q2: What is a Region Pair and why does it matter for Disaster Recovery?
**Answer**:
A Region Pair consists of two regions within the same geography (typically >300 miles apart). It matters because:
1. **Platform Replication**: Services like GRS storage automatically replicate data to the paired region.
2. **Safe Deployment**: Azure updates are rolled out sequentially to pairs, ensuring one region is always stable.
3. **Prioritized Recovery**: In a global outage, Microsoft prioritizes recovering one region in every pair.

### Q3: How do you design for data sovereignty in a multi-national bank?
**Answer**:
1. **Region Selection**: Choose regions strictly within the legal boundaries (e.g., Germany West Central for German data).
2. **Policy Enforcement**: Use **Azure Policy** to deny resource creation in non-compliant regions.
3. **Replication Control**: Configure storage accounts with LRS (Locally Redundant Storage) or ZRS (Zone Redundant Storage) to prevent data from automatically replicating to a paired region if that pair is outside the sovereign boundary.

## Key Takeaways
- **SLA Hierarchy**: Single VM (99.9%) < AvSet (99.95%) < AvZone (99.99%) < Multi-Region.
- **Latency**: Speed of light matters. Test latency between regions using tools like `azurespeed.com`.
- **Compliance**: Always verify data residency requirements before selecting regions.
- **Cost**: Data transfer between zones and regions is **not free**.

## Further Reading
- [Azure Regions and Availability Zones](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-service-support)
- [Cross-region replication in Azure](https://learn.microsoft.com/en-us/azure/reliability/cross-region-replication-azure)
- [Azure Geographies](https://azure.microsoft.com/en-us/explore/global-infrastructure/geographies/)
