# GCP Architecture Reference

## Service selection (quick decision map)

| Need | Preferred | Alternatives / when |
|------|-----------|---------------------|
| Compute — VMs | Compute Engine (MIGs) | Spot VMs for stateless/batch |
| Compute — containers | GKE (Autopilot default) | Cloud Run for serverless containers |
| Compute — serverless | Cloud Run (functions or containers) | Cloud Functions (Gen2 = Cloud Run) |
| Orchestration — workflows | Workflows | Cloud Composer for complex DAGs |
| Storage — object | Cloud Storage (lifecycle policies) | Archive class |
| Storage — block | Persistent Disk | Hyperdisk for extreme IOPS |
| Storage — file | Filestore | NetApp Volumes for enterprise |
| Database — relational | Cloud SQL | AlloyDB for PostgreSQL at scale |
| Database — NoSQL | Firestore (app dev) / Bigtable (analytics/ops) | Memorystore in front for cache |
| Database — warehouse | BigQuery | Looker for BI on top |
| Cache | Memorystore Redis | Cloud CDN for edge cache |
| Messaging | Pub/Sub | Kafka (managed) for strict ordering |
| API layer | API Gateway / Cloud Endpoints | Apigee for enterprise API mgmt |
| CDN/edge | Cloud CDN + Cloud Armor | Global external LB for TCP/UDP |
| Identity | Cloud IAM + Identity Platform | Workload Identity Federation for on-prem |
| Secrets | Secret Manager | KMS for keys |
| Observability | Cloud Monitoring/Logging + Trace | Managed Prometheus for GKE |
| CI/CD | Cloud Build + Deploy | GitHub Actions / Cloud Deploy (progressive) |

## Architecture Framework — 4-pillar checklist (compressed)

### 1. Operational Excellence
- [ ] Everything is IaC (Terraform-first); no console-only changes
- [ ] CI/CD with Cloud Build/Deploy; canary via Cloud Deploy
- [ ] Cloud Monitoring alerts + centralized Logging (org sinks)
- [ ] SRE practices: SLOs defined, error budgets, runbooks

### 2. Security, Privacy, Compliance
- [ ] Org/Folder/Project hierarchy; IAM least-privilege at folder level
- [ ] Workload Identity (GKE SA → IAM SA), no service account keys
- [ ] VPC Service Controls for data exfil protection (BigQuery, GCS)
- [ ] Private Google Access; no public IPs on VMs
- [ ] CMEK (customer-managed keys) where compliance demands; default encryption on
- [ ] Secrets in Secret Manager with rotation
- [ ] Cloud Armor + reCAPTCHA on public endpoints; Security Command Center on

### 3. Reliability
- [ ] Regional MIGs (multi-zone) for compute; regional GKE clusters
- [ ] Cloud SQL HA (regional replicas) + automated backups + PITR
- [ ] Cloud Storage: dual-region/multi-region for critical data
- [ ] Global load balancing (anycast) — active-active multi-region where needed
- [ ] RTO/RPO defined; DR drills executed
- [ ] Pub/Sub with DLQ topics; retries with backoff
- [ ] Autoscaling on load metrics for MIGs/GKE/Cloud Run

### 4. Cost Optimization
- [ ] Committed Use Discounts (CUDs) for steady state; Spot for flexible
- [ ] Cloud Storage lifecycle → Archive; delete old snapshots
- [ ] Rightsizing recommendations (active assist)
- [ ] Budgets + alerts per project/folder
- [ ] Autopilot GKE / Cloud Run scale-to-zero for spiky workloads
- [ ] BigQuery: partitioned/clustered tables, slot reservations only when steady

## Core patterns

```text
# Shared VPC (the standard enterprise pattern)
Host project:    VPC networks, firewalls, interconnects
Service projects: workloads attached via shared subnets
```

- Landing zone: org → folders (env/business unit) → projects; bootstrap with Terraform
- 3-tier: Global external LB → GKE/Cloud Run → Cloud SQL Private Service Connect
- Serverless API: API Gateway/Cloud Run → Firestore
- Event-driven: Pub/Sub → Cloud Run/Workflows consumers
- Data lake: GCS → BigQuery (external tables) → Looker
