# Multi-Cloud & Hybrid Reference

## Design method (vendor-neutral first)

Always express requirements as **vendor-neutral capabilities** before picking services:

```text
1. Capability model:  "auto-scaling container platform", "managed Postgres",
                      "serverless event consumers", "central identity"
2. Vendor mapping:    project each capability onto the chosen vendor's services
                      with a fidelity note (exact / partial / workaround / gap)
3. Implement:         Terraform per vendor, one module set per provider
```

This prevents lock-in thinking and makes the *why* of every service choice explicit (it also feeds the ADR).

## Provider selection matrix

| Driver | Prefer | Notes |
|--------|--------|-------|
| Enterprise Microsoft estate / Entra ID | Azure | native identity + licensing |
| Deep AWS skills / largest service breadth | AWS | most mature partner ecosystem |
| Data/analytics (BigQuery), K8s (GKE) | GCP | best-in-class data + K8s |
| Cost-sensitive / simplicity | GCP or AWS | evaluate per workload |
| Existing on-prem VMware | Azure (AVS) or AWS (VMC) | hybrid lift |
| Avoiding lock-in / regulatory spread | multi-cloud | costs ops complexity — charge for it |

Rule: **multi-cloud only when there's a real driver** (DR, data gravity, regulation, acquisition). Single-cloud + well-designed landing zone beats opportunistic multi-cloud.

## Landing zones (per provider)

| Layer | AWS | Azure | GCP |
|-------|-----|-------|-----|
| Org / identity | Organizations + Control Tower + SSO | Management groups + Entra ID | Org → folders → projects + Cloud Identity |
| Network | Shared VPC / transit gateway | Hub-spoke VNets | Shared VPC |
| Security baseline | SCPs, GuardDuty, Security Hub | Azure Policy, Defender | Org policies, SCC |
| Logging | CloudTrail org trail | Log Analytics/Sentinel | Org-level log sinks |
| Billing | Cost categories + budgets | Budgets + cost management | Budgets per folder/project |

## Hybrid / on-prem connectivity

```text
AWS:  Direct Connect (private) or Site-to-Site VPN (cheap)
Azure: ExpressRoute or VPN Gateway
GCP:  Dedicated/Partner Interconnect or HA VPN
```

- DNS: Route 53 Resolver / Azure DNS Private Resolver / Cloud DNS forwarding zones
- Identity sync: SSO federation (SAML/OIDC) into each cloud — one Entra ID/Okta as source of truth
- Workload Identity Federation (GCP) / IAM roles anywhere (AWS) / workload identity federation (Azure) — no static keys

## Multi-cloud DR pattern (most common real-world use)

```text
Primary:  AWS us-east-1 (active)
Secondary: Azure (warm) or GCP (pilot-light)
- Data: async replication (DMS/Azure Data Box... no — managed services:
  AWS DMS, Azure SQL geo-replication, GCP Spanner multi-region)
- DNS: global failover (Route 53 / Traffic Manager / Cloud DNS routing policies)
- Terraform: one module per provider; DR stack tested every quarter
- RPO: replication lag; RTO: time to flip DNS + scale up warm pool
```

## Multi-cloud MUST / MUST NOT

MUST:
- One identity plane (SSO federation), one observability story (or an aggregator)
- One IaC toolchain (Terraform) across providers
- Per-provider landing zone before any workload
- Explicit inter-cloud networking + encryption in transit (IPsec/private interconnect)

MUST NOT:
- Stretch clusters or synchronous replication across clouds (latency/consistency trap)
- Lift-and-shift to a second cloud "just in case"
- Duplicate PaaS SaaS-style services across clouds without a data-gravity reason
- No static access keys for cross-cloud automation (federation only)
