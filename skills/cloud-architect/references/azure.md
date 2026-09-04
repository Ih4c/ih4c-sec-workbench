# Azure Architecture Reference

## Service selection (quick decision map)

| Need | Preferred | Alternatives / when |
|------|-----------|---------------------|
| Compute — VMs | Azure VM Scale Sets | Spot VMs for stateless/batch |
| Compute — containers | AKS (full K8s) | Container Apps (serverless containers), ACI (single) |
| Compute — serverless | Azure Functions | Container Apps for longer/gpu/heavy runtimes |
| Orchestration — workflows | Logic Apps | Durable Functions for code-first workflows |
| Storage — object | Blob Storage (lifecycle tiers) | Archive tier |
| Storage — block | Managed Disks | Ultra/SSD v2 for extreme IOPS |
| Storage — file | Azure Files | NetApp Files for enterprise |
| Database — relational | Azure SQL / SQL Managed Instance | PostgreSQL/MySQL Flexible Server |
| Database — NoSQL | Cosmos DB | Table Storage legacy only |
| Database — warehouse | Synapse / Fabric | Databricks for Spark-heavy |
| Cache | Azure Cache for Redis | Front Door cache for edge |
| Messaging | Service Bus (enterprise) / Event Grid (routing) | Event Hubs (Kafka-style streams) |
| API layer | API Management | Front Door for global routing + WAF |
| CDN/edge | Azure Front Door | CDN classic for pure caching |
| Identity | Microsoft Entra ID + managed identities | Entra B2C for app users |
| Secrets | Key Vault (+ RBAC model) | App Configuration for feature flags |
| Observability | Azure Monitor + Log Analytics + App Insights | Sentinel for SIEM |
| CI/CD | Azure DevOps / GitHub Actions | Bicep + deployment stacks |

## Well-Architected Framework — 5-pillar checklist (compressed)

### 1. Reliability
- [ ] Availability Zones for stateful; zones + region pairs for DR
- [ ] Availability Sets or Zonal VMSS for VMs
- [ ] SQL: geo-replication or failover groups; point-in-time restore
- [ ] Backups (Recovery Services vault) + tested restores
- [ ] RTO/RPO defined; Site Recovery for DR region
- [ ] Retries with exponential backoff; dead-lettering in Service Bus/Event Grid
- [ ] Health probes on every endpoint; autoscale rules on load

### 2. Security
- [ ] Managed identities everywhere (no service principals with secrets)
- [ ] Zero hardcoded secrets — Key Vault with RBAC, keyVaultRef in Bicep
- [ ] Private Endpoints for PaaS (SQL, Storage, Key Vault) — no public exposure
- [ ] NSGs + least-privilege; WAF on public endpoints (Front Door/App GW)
- [ ] TLS 1.2+ everywhere; encryption at rest (default) + customer keys where required
- [ ] Defender for Cloud on; diagnostics → Log Analytics/Sentinel
- [ ] Entra: PIM for privileged roles, Conditional Access policies

### 3. Cost Optimization
- [ ] Reservations / Savings Plans for steady VM/SQL spend
- [ ] Blob lifecycle tiers (Hot → Cool → Archive); delete stale snapshots
- [ ] Auto-shutdown for dev/test VMs; serverless tiers where spiky
- [ ] Budgets + alerts on every subscription
- [ ] Rightsize with Advisor; orphan cleanup (disks, IPs, NICs)
- [ ] Tagging policy (env/owner/cost-center) enforced by Azure Policy

### 4. Operational Excellence
- [ ] Everything is IaC (Bicep-first, or Terraform); no portal-only changes
- [ ] CI/CD: validate (az bicep build + what-if) before deploy
- [ ] Azure Monitor alerts on key metrics; centralized Log Analytics
- [ ] Azure Policy guardrails (naming, SKUs, locations)
- [ ] Runbooks + recovery drills documented

### 5. Performance Efficiency
- [ ] Right VM SKUs (B-series dev, D/E-series general, M memory)
- [ ] Autoscale on metrics; caching with Redis/CDN/Front Door
- [ ] SQL: proper indexes, elastic pools for multi-tenant, Hyperscale for big
- [ ] Review periodically (Advisor performance recommendations)

## Core patterns

```text
# Hub-spoke (the standard enterprise pattern)
Hub VNet:    firewalls, VPN/ER gateways, shared services, DNS
Spoke VNets: per workload/env — peered to hub, no spoke-to-spoke
```

- Landing zone: subscriptions per env/workload under management groups, policy inheritance
- 3-tier: Front Door/App GW → VMSS/App tier → SQL Private Endpoint
- Serverless API: API Management → Functions → Cosmos DB
- Event-driven: Event Grid → Service Bus queues → Functions
- Data lake: Blob ADLS Gen2 → Synapse/Fabric → Power BI

## Bicep conventions (when Bicep is chosen over Terraform)

```text
□ Prefer Bicep over ARM JSON; use modules for reusable patterns
□ @allowed/@minLength/@secure decorators on parameters
□ targetScope = 'subscription' for policy/RBAC assignments
□ Use Azure Verified Modules (AVM) where they fit
□ keyVaultRef for secrets; user-assigned managed identities preferred
□ Validate: az bicep build + az deployment what-if before apply
□ Deployment stacks for lifecycle-managed resources
```
