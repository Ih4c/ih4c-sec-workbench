# Architecture Review Workflow

Pillar-by-pillar review of an existing or proposed architecture. Merges AWS Well-Architected (6 pillars), Azure WAF (5 pillars), GCP Architecture Framework (4 pillars) into one neutral 6-pillar pass. Works against IaC repos, live resources, or design docs.

## Review modes

```text
quick        — 15 min: worst offenders only (security + cost)
pillar-scoped — one pillar deep-dive
full         — all 6 pillars, evidence-cited
drift-check  — compare IaC vs live resources
```

## Steps

```text
1. Scope & inventory
   □ What's in scope: subscription/project/account, workloads, repos
   □ Scan IaC files (Terraform *.tf, Bicep *.bicep, CloudFormation, CDK)
   □ Inventory live resources:
     AWS:   aws resourcegroupstaggingapi / per-service list calls
     Azure: az resource list --subscription X
     GCP:   gcloud asset search-all-resources
   □ Drift: compare IaC vs live → flag portal/console-created resources,
     undeployed definitions, config mismatches
2. Diagram  — generate Mermaid architecture diagram from the inventory
3. Pillar pass (evidence per finding: resource ID + why it fails)
4. Report + prioritized remediation roadmap
```

## The 6-pillar checklist (merged, essentials)

### 1. Security — highest priority, always
```text
□ Identity: least-privilege? no long-lived keys? MFA? (IAM/Entra/Cloud IAM)
□ Secrets: all in vaults, rotated?
□ Network: private endpoints for PaaS? NSG/SG least-privilege? WAF on public?
□ Encryption: at rest + TLS in transit everywhere?
□ Detection: GuardDuty/Defender/SCC + audit trails on?
□ Public exposure: any public buckets/blob/objects? default-open ports?
```

### 2. Reliability
```text
□ Every stateful service multi-AZ/zone? backups + tested restore?
□ Autoscaling on load? health checks on every endpoint?
□ RTO/RPO defined and tested? DR region/plan exists?
□ SPOFs: single LB, single region, single DB instance?
□ Retry/backoff/DLQ in async paths?
```

### 3. Cost
```text
□ Commitments in place for steady state? spot for flexible?
□ Orphans: unattached disks/IPs/NICs/snapshots?
□ Storage tiers + lifecycle rules active?
□ Budgets + alerts configured?
□ Rightsizing: any chronically <10% CPU instances?
```

### 4. Operational Excellence
```text
□ Everything in IaC? any console-only resources?
□ CI/CD with plan/review before apply?
□ Centralized logging + alerting on key metrics?
□ Runbooks for known failures? recent game day/drill?
□ Tagging consistent and enforced?
```

### 5. Performance Efficiency
```text
□ Right compute families/SKUs? last rightsize review when?
□ Caching at the right layers (CDN, in-memory, DB)?
□ DB: indexes, pooling, replicas where needed?
□ Latency: services in same region? cross-AZ/zone chatter?
```

### 6. Sustainability (skip for quick mode)
```text
□ ARM/Graviton where possible? scale-to-zero for idle?
□ Oversized storage/instance waste? retention policies sane?
```

## Report format

```text
# Architecture Review — <system> (<date>)
## Summary: N findings (X critical / Y high / Z medium)
## Per pillar: finding table
| ID | Severity | Finding | Evidence (resource ID/file:line) | Recommendation |
## Drift report (if run)
## Remediation roadmap: quick wins → 30/60/90 days
```

## Rules

- Every finding MUST cite evidence (resource ID, file:line, or command output) — no evidence, no finding
- Scanner/surface-level observation ≠ confirmed finding; verify before reporting
- Severity honestly: call it "unverified" if you couldn't confirm
- Remediation first, blame never
