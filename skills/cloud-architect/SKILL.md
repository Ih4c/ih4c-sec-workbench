---
name: cloud-architect
description: Solutions architecture across AWS, Azure, GCP (and hybrid/on-prem). Designs, reviews, and generates IaC for full-spectrum workloads — VMs, containers/K8s, serverless, data platforms, AI/ML platforms — following Well-Architected (AWS), Well-Architected Framework (Azure), and Architecture Framework (GCP). Terraform-first, Bicep/CDK/CloudFormation/Pulumi when needed. Triggers: cloud architecture, solutions architect, AWS architecture, Azure architecture, GCP architecture, landing zone, Well-Architected review, architecture review, IaC, Terraform, Bicep, cloud cost optimization, FinOps, DR design, hybrid cloud, AI/ML platform architecture, MLOps, RAG infrastructure.
---

# Cloud Architect Skill

Full-spectrum cloud solutions architecture: design, review, cost, security, reliability, and IaC for AWS + Azure + GCP (+ hybrid/on-prem).

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Confirm what the user wants: **design** (new), **review** (existing), or **migrate**.
2. `NOW`: Identify the vendor(s): AWS / Azure / GCP / hybrid. Unknown → ask; never assume.
3. `NEXT`: Load the matching reference(s): `references/aws.md` · `references/azure.md` · `references/gcp.md` · `references/multi-cloud.md`. Design in 3 layers (foundation → platform → workload).
4. `NEXT`: Load `references/iac.md` (Terraform-first), `references/cost.md`, `references/review-workflow.md`, `references/ai-ml-platforms.md` as the task demands.
5. `ACT`: Produce deliverables per `references/deliverables.md`.
6. `MUST`: Pass the **approval gate** before generating any IaC code.

## Workflow (8 phases)

```text
1. Discovery      — requirements, constraints, compliance (SOC2/HIPAA/PCI/GDPR), current state
2. Design         — vendor-neutral capability model FIRST, then vendor service mapping
                    (3 layers: foundation/landing zone → shared platform → workload)
3. Security       — zero-trust, least-privilege identity, encryption everywhere,
                    private endpoints, secrets in vaults
4. Cost Model     — right-sizing, reserved/spot/savings plans, tagging, budgets + alerts
5. Reliability/DR — 99.9%+ availability, no SPOFs, RTO/RPO defined, multi-region for critical
6. IaC            — Terraform-first; modules, remote state, plan-review-apply; drift detection
7. APPROVAL GATE  — present: architecture + diagram + cost + security before writing code
8. Validate/Ops   — health checks, monitoring + alarms, runbooks, continuous optimization
```

## Validation checkpoints

- **After Design**: every component has a redundancy strategy; no single points of failure
- **Before IaC**: approval gate passed (user confirmed the plan)
- **After deployment**: verify health/routing (ALB target groups, LB backend health, GCP health checks)
- **After DR test**: RTO/RPO actually met — document real recovery times, don't assume

## MUST

- Design for 99.9%+ availability; multi-region for critical workloads
- Zero-trust security: least-privilege IAM/Entra ID/Cloud IAM, no wildcard permissions
- Terraform-first IaC (Bicep/CDK/CloudFormation/Pulumi only when the user asks or the ecosystem demands)
- Cost tags on every resource; monitoring + alarms on every deployment
- DR with defined RTO/RPO, tested — not just documented
- Prefer managed services over self-managed unless there's a reason
- Document architecture decisions (ADR format)
- Verification gate: cite provider docs/specifics before claiming a service behaves a certain way

## MUST NOT

- No credentials in code/repos (use vaults: Secrets Manager / Key Vault / Secret Manager)
- No skipped encryption (at rest + in transit)
- No single points of failure
- No monitoring-less deployments
- No over-engineered architectures (simplest thing that meets the requirements)
- No ignored compliance requirements
- No skipped DR testing
- No IaC code before the approval gate

## References (load by task)

| File | Use when |
|------|----------|
| `references/aws.md` | AWS design/review: WA 6 pillars, service selection, VPC/IAM patterns |
| `references/azure.md` | Azure design/review: WAF 5 pillars, Bicep conventions, Entra ID, hub-spoke |
| `references/gcp.md` | GCP design/review: ARC 4 pillars, org/folder/project, VPC SC |
| `references/multi-cloud.md` | hybrid/on-prem, landing zones, provider selection matrix, multi-cloud DR |
| `references/iac.md` | Terraform conventions + Bicep/CDK/CloudFormation/Pulumi equivalents |
| `references/cost.md` | FinOps: right-sizing, commitments, budgets, cross-provider TCO |
| `references/review-workflow.md` | pillar-by-pillar architecture review + drift detection |
| `references/ai-ml-platforms.md` | SageMaker/Bedrock, Azure AI/Foundry, Vertex AI, MLOps, RAG infra |
| `references/deliverables.md` | diagram, ADR, cost estimate, security design, deployment/rollback plan |

## Completion self-check (MUST pass before claiming done)

- [ ] Did I produce an architecture diagram with data flows?
- [ ] Service selection rationale documented (compute/storage/database/networking)?
- [ ] Security architecture reviewed (identity, network, encryption, secrets)?
- [ ] Cost estimate + optimization strategy provided?
- [ ] IaC delivered only after the approval gate? Drift detection commands included?
- [ ] ADR written for every significant decision?
