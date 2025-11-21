# Cost Optimization & Governance

## Overview
"The cloud is just someone else's computer, and they charge by the second."
Staff Engineers are responsible for the **Unit Economics** of their architecture.
Interviewers want to see that you can innovate *within budget* and enforce standards *without* being a bottleneck.

## Foundational Concepts

### CapEx vs. OpEx
- **CapEx (Capital Expenditure)**: Upfront cost (buying servers).
- **OpEx (Operational Expenditure)**: Pay-as-you-go (renting VMs). Azure is primarily OpEx.

### Governance Hierarchy
1. **Management Group**: Policy & Compliance across subscriptions.
2. **Subscription**: Billing boundary.
3. **Resource Group**: Lifecycle boundary.
4. **Resource**: The actual service.

## Technical Deep Dive

### 1. Cost Optimization Strategies
- **Reservations (RIs)**: Commit to 1 or 3 years for ~72% discount. Use for steady-state workloads.
- **Savings Plans**: Commit to $X/hour spend. More flexible than RIs (applies across regions/families).
- **Azure Hybrid Benefit (AHB)**: Bring your on-prem Windows/SQL licenses to Azure. Saves ~40%.
- **Spot Instances**: Use for interruptible workloads (Batch, Dev/Test).

### 2. Azure Policy
"Guardrails, not Gates."
- **Definitions**: The rule (e.g., "Allowed Locations = East US").
- **Initiatives**: A collection of definitions (e.g., "NIST Compliance").
- **Effects**:
  - **Deny**: Block the deployment.
  - **Audit**: Allow but log it.
  - **DeployIfNotExists (DINE)**: Automatically fix it (e.g., "Enable Backup on this VM").

### 3. Azure Blueprints
Orchestrates the deployment of:
- Resource Groups
- ARM Templates
- Role Assignments
- Policies
*Used to stamp out compliant environments (e.g., "FedRAMP Ready Subscription").*

## Visual Representations

### Governance Hierarchy & Scope
```mermaid
graph TD
    Root[Tenant Root Group]
    
    subgraph "Management Groups"
        ProdMG[Production MG]
        DevMG[Development MG]
    end
    
    subgraph "Subscriptions"
        SubA[Sub A (Banking App)]
        SubB[Sub B (Trading App)]
        SubC[Sub C (Sandbox)]
    end
    
    subgraph "Policies"
        P1[Policy: Deny Public IP]
        P2[Policy: Require Tags]
    end
    
    Root --> ProdMG
    Root --> DevMG
    
    ProdMG --> SubA
    ProdMG --> SubB
    DevMG --> SubC
    
    ProdMG -.->|Inherits| P1
    ProdMG -.->|Inherits| P2
```

## Configuration Examples

### Azure Policy: Enforce Tagging (JSON)
```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "field": "tags['CostCenter']",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

### Azure CLI: Create a Budget
```bash
az consumption budget create \
  --amount 1000 \
  --name MonthlyBudget \
  --category cost \
  --time-grain monthly \
  --start-date 2023-01-01 \
  --end-date 2023-12-31 \
  --resource-group MyRG
```

## Real-World Enterprise Scenarios

### Scenario: "The Bill Shock"
**Requirement**: A developer left a GPU cluster running over the weekend. The bill is $5,000.
**Solution**: **Budgets & Alerts**.
1. Set a **Budget** on the Dev Subscription.
2. Configure **Alerts** at 50%, 80%, and 100% of budget.
3. **Action Group**: Trigger an Azure Function to *shut down* VMs when budget hits 100%.

### Scenario: Compliance Enforcement
**Requirement**: The bank must ensure NO data is ever stored in a region outside the US.
**Solution**: **Azure Policy**.
1. Assign the built-in policy "Allowed locations" to the Root Management Group.
2. Parameter: `["eastus", "westus", "centralus"]`.
3. **Effect**: Deny.
4. Result: Even the CIO cannot create a VM in Europe.

## Interview Questions & Model Answers

### Q1: What is the difference between Azure Policy and RBAC?
**Answer**:
- **RBAC (Who)**: Controls *user access*. "Can John create a VM?"
- **Policy (What)**: Controls *resource properties*. "Can John create a *G5* VM?" (No, too expensive).
- **Together**: RBAC lets John in the door; Policy stops him from breaking the furniture.

### Q2: How do you allocate costs in a shared subscription?
**Answer**:
**Tagging**.
- Enforce a `CostCenter` tag on every resource group.
- Use **Azure Cost Management** to group costs by Tag.
- For shared resources (like an ExpressRoute circuit), use a "Shared" tag and split the cost manually or via a chargeback model.

### Q3: Explain the "DeployIfNotExists" policy effect.
**Answer**:
It is a remediation tool.
- If I create a VM and forget to enable the Monitoring Agent...
- The Policy engine sees the non-compliance.
- It triggers a deployment (ARM template) to *install* the agent automatically.
- Essential for maintaining a baseline security posture without blocking developers.

## Key Takeaways
- **Tags** are not optional; they are the metadata of your business.
- **Reservations** are free money if you have stable usage.
- **Management Groups** are the key to governing scale (100+ subscriptions).

## Further Reading
- [Azure Cost Management and Billing](https://learn.microsoft.com/en-us/azure/cost-management-billing/)
- [Azure Policy documentation](https://learn.microsoft.com/en-us/azure/governance/policy/)
- [Organize your resources with management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
