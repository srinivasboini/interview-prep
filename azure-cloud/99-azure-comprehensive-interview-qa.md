# Comprehensive Azure Interview Questions & Answers

## Overview
This document consolidates the most critical interview questions for Staff/Principal Engineer roles.
They are categorized by **Topic** and **Difficulty**.

---

## 1. Global Infrastructure & Networking

### Q1: [Foundational] What is the difference between a Region and an Availability Zone?
**Answer**: A Region is a set of data centers deployed within a latency-defined perimeter. An Availability Zone (AZ) is a unique physical location within a region with independent power, cooling, and networking. Regions provide data residency; AZs provide high availability (99.99%).

### Q2: [Advanced] Explain the flow of traffic in a Hub-and-Spoke topology when a Spoke VM accesses the Internet.
**Answer**:
1. **Spoke VM** initiates traffic.
2. **UDR (User Defined Route)** on the Spoke Subnet forces `0.0.0.0/0` to the **Azure Firewall** (NVA) in the Hub.
3. **Peering** carries traffic to the Hub VNet.
4. **Azure Firewall** applies SNAT (Source Network Address Translation) and filters traffic against rules.
5. Traffic exits to the Internet via the Firewall's Public IP.

### Q3: [Scenario] You need to connect 50 branch offices to Azure. What connectivity option do you choose?
**Answer**: **Azure Virtual WAN**. It is designed for large-scale branch connectivity. It aggregates VPN, ExpressRoute, and User VPN connections into a single hub, simplifying routing and management compared to managing 50 individual Site-to-Site VPNs.

---

## 2. Compute Services

### Q4: [Intermediate] When should I use Azure Functions vs. Azure App Service?
**Answer**:
- **Azure Functions**: Event-driven, short-lived, bursty workloads. "Run code when a file is uploaded." Cost scales to zero.
- **App Service**: Long-running web applications, APIs, predictable traffic. "Host a corporate website." Easier debugging and monitoring for traditional apps.

### Q5: [Tricky] What happens to the data on the Temporary Disk (D: drive) of a VM when I stop and start it?
**Answer**: It is **lost**. The temporary disk is physically located on the host server. When you stop/deallocate a VM, it may move to a different host upon restart. Never store persistent data on the temporary disk.

### Q6: [Advanced] How does AKS integrate with Azure AD for Pod Identity?
**Answer**: The modern approach is **Workload Identity**. It uses OIDC federation. The Pod is projected with a Service Account token. Azure AD trusts this token and exchanges it for an Azure AD Access Token, allowing the Pod to access resources (like Key Vault) without managing secrets.

---

## 3. Storage & Databases

### Q7: [Foundational] What is the difference between Blob Storage and Data Lake Gen2?
**Answer**: ADLS Gen2 is Blob Storage with a **Hierarchical Namespace** enabled. This allows for atomic directory operations (renaming a folder is instant), which is critical for Big Data analytics performance.

### Q8: [Scenario] A developer complains that their Cosmos DB queries are getting 429 errors. What do you do?
**Answer**:
1. Check if they are exceeding the Provisioned Throughput (RUs).
2. Analyze the queries: Are they cross-partition? (Expensive).
3. Check the Partition Key strategy: Is there a "Hot Partition"?
4. Solution: Optimize query, change partition key, or enable Autoscale.

### Q9: [Advanced] Explain the "Business Critical" tier in Azure SQL Database.
**Answer**: It uses a cluster of SQL Server nodes with local SSD storage. One primary (Read/Write) and multiple secondaries (Read-Only). It provides the highest performance (low latency) and built-in High Availability using Always On Availability Groups technology.

---

## 4. Security & Identity

### Q10: [Foundational] What is a Managed Identity?
**Answer**: An identity in Azure AD that is automatically managed by Azure. You assign it to a resource (like a VM). The application uses it to authenticate to other services (like SQL) without storing credentials in code.

### Q11: [Scenario] How do you prevent a developer from creating a Public IP address?
**Answer**: Use **Azure Policy**. Create a policy definition that denies the creation of `Microsoft.Network/publicIPAddresses` and assign it to the developer's subscription or resource group.

### Q12: [Advanced] Describe the "Zero Trust" model in Azure.
**Answer**: "Never Trust, Always Verify."
1. **Verify Explicitly**: Always authenticate and authorize based on all available data points (User, Location, Device).
2. **Use Least Privilege**: JIT/JEA (Just-In-Time/Just-Enough-Access).
3. **Assume Breach**: Segment networks, encrypt data, and monitor for threats.

---

## 5. Architecture & Design

### Q13: [Scenario] Design a highly available architecture for a banking app with RPO < 5 mins and RTO < 15 mins.
**Answer**:
- **Region**: Multi-Region (Active-Passive).
- **Traffic**: Azure Front Door (Global LB).
- **Compute**: AKS in both regions.
- **Database**: Azure SQL Database with **Auto-Failover Groups** (Async replication).
- **Storage**: GZRS (Geo-Zone-Redundant).
- **Failover**: Automated via Front Door health probes and SQL Failover Group.

### Q14: [Tricky] Why is "Lift and Shift" often more expensive than on-prem?
**Answer**: Because on-prem VMs are often over-provisioned (sunk cost). In the cloud, you pay for every allocated core. If you lift a 16-core VM that uses 5% CPU, you are wasting money. **Right-sizing** is essential during migration.

### Q15: [Advanced] What is the "Strangler Fig" pattern?
**Answer**: A migration pattern where you gradually replace specific pieces of functionality in a legacy monolith with new microservices. You use a proxy (like API Management or Front Door) to route traffic to the new services one by one, until the monolith is "strangled" and can be decommissioned.

---

## 6. DevOps & IaC

### Q16: [Intermediate] What is the difference between `terraform plan` and `terraform apply`?
**Answer**: `plan` is a dry-run. It compares the state file with the code and the real world, showing what *will* happen. `apply` actually executes the changes. Always review the plan before applying.

### Q17: [Scenario] You need to store a database password for an ARM template deployment. Where do you put it?
**Answer**: **Azure Key Vault**.
1. Create the secret in Key Vault.
2. In the ARM template parameters file, reference the Key Vault secret ID.
3. Never put the password in the parameters file itself.

---

## 7. Monitoring & Reliability

### Q18: [Foundational] What is the difference between Metrics and Logs?
**Answer**: Metrics are numerical values (CPU %) measured over time; good for alerting. Logs are textual records (Stack Trace); good for debugging.

### Q19: [Advanced] How does Azure Monitor "Smart Detection" work?
**Answer**: It uses machine learning algorithms to analyze telemetry. It establishes a baseline of normal behavior (e.g., "Page load time is usually 200ms"). If it detects a statistically significant deviation (e.g., "Page load time is now 2s"), it triggers an alert automatically, reducing the need for manual threshold configuration.

### Q20: [Scenario] Your application is down. What is the first thing you check?
**Answer**: **Azure Service Health**. Check if there is a widespread Azure outage in the region. If Azure is healthy, then check **Application Insights** for failed requests and **Resource Health** for specific VM/Service issues.
