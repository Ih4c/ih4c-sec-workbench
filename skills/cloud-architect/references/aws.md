# AWS Architecture Reference

## Service selection (quick decision map)

| Need | Preferred | Alternatives / when |
|------|-----------|---------------------|
| Compute — VMs | EC2 (ASG + ALB) | EC2 Spot for stateless/batch |
| Compute — containers | ECS Fargate (simple) / EKS (full K8s) | ECS EC2 for cost on steady load |
| Compute — serverless | Lambda | Fargate if >15min, GPU, or heavy runtime |
| Orchestration — workflows | Step Functions | SWF legacy only |
| Storage — object | S3 (lifecycle policies) | Glacier for archive |
| Storage — block | EBS gp3 | io2 for extreme IOPS, st1/sc1 cold |
| Storage — file | EFS | FSx for Windows/Lustre |
| Database — relational | Aurora PostgreSQL/MySQL | RDS if engine-specific needs |
| Database — NoSQL | DynamoDB | ElastiCache in front, DAX for cache |
| Database — warehouse | Redshift (RA3) | Athena for ad-hoc on S3 |
| Cache | ElastiCache Redis | MemoryDB for Redis persistence |
| Messaging | SQS (queue) / SNS (fanout) / EventBridge (routing) | Kinesis for ordered streams |
| API layer | API Gateway (REST/HTTP) | AppSync for GraphQL |
| CDN/edge | CloudFront + WAF | Global Accelerator for TCP/UDP |
| Identity | IAM + Organizations + SSO | Cognito for app users |
| Secrets | Secrets Manager | SSM Parameter Store for non-rotation |
| Observability | CloudWatch + X-Ray | Managed Grafana/Prometheus for OSS |
| CI/CD | CodePipeline/CodeBuild | GitHub Actions / GitLab |

## Well-Architected — 6-pillar checklist (compressed)

### 1. Operational Excellence
- [ ] Everything is IaC (CDK/CloudFormation/Terraform), no console-only changes
- [ ] CI/CD pipelines; deployments are reversible (blue/green or canary)
- [ ] CloudWatch alarms on key metrics; centralized logging
- [ ] Runbooks exist for known failure modes; game days practiced

### 2. Security
- [ ] IAM least-privilege; no long-lived access keys (use roles/OIDC)
- [ ] MFA on root + humans; SCPs at org level
- [ ] VPC: private subnets for workloads; security groups as the only per-instance firewall
- [ ] Encryption: KMS everywhere (EBS, RDS, S3 SSE-KMS); TLS in transit
- [ ] Secrets in Secrets Manager with rotation
- [ ] GuardDuty + Security Hub + CloudTrail (org-wide) on
- [ ] Public S3 buckets blocked (account-level block public access)

### 3. Reliability
- [ ] Multi-AZ for everything stateful (RDS/Aurora, ElastiCache, ALB)
- [ ] ASG with min 2 across AZs for stateless compute
- [ ] Backups: automated + tested restores; RDS point-in-time recovery
- [ ] RTO/RPO defined per workload; DR region + failover plan for critical
- [ ] Retry with exponential backoff + jitter; DLQs for async
- [ ] Circuit breakers / bulkheads between services

### 4. Performance Efficiency
- [ ] Right instance families (Graviton first); right storage class
- [ ] Autoscaling on load metrics (target tracking), not guesses
- [ ] Cache at every layer (CloudFront, ElastiCache, DAX)
- [ ] DB: proper indexes, read replicas, connection pooling
- [ ] Review periodically against newer instance types

### 5. Cost Optimization
- [ ] Savings Plans / Reserved Instances for steady state; Spot for flexible
- [ ] S3 Intelligent-Tiering; lifecycle to Glacier
- [ ] Auto-shutdown for dev/test (Instance Scheduler)
- [ ] Cost allocation tags on everything; Budgets + alerts
- [ ] Rightsize with Compute Optimizer
- [ ] Serverless-first for spiky/unpredictable workloads

### 6. Sustainability
- [ ] Graviton/ARM wherever possible
- [ ] Scale-to-zero (Lambda) vs idle EC2
- [ ] Right-size storage; delete snapshots/AMIs past retention

## Core patterns

```text
# 3-tier VPC (the standard)
VPC 10.0.0.0/16
├── public:    ALB, NAT GW, bastion
├── private:   ASG app tier
└── database:  RDS/Aurora, ElastiCache
```

- Landing zone: Organizations + Control Tower, OUs per environment/business unit, shared network account
- Serverless API: API GW → Lambda → DynamoDB (single-table design)
- Event-driven: EventBridge bus → SQS → Lambda consumers
- Data lake: S3 raw → Glue catalog → Athena/EMR/Redshift
