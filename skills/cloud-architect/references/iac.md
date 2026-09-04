# IaC Reference (Terraform-first)

## Default toolchain

**Terraform is the default.** Use alternatives only when the user asks or the ecosystem demands:

| Situation | Use |
|-----------|-----|
| Default, multi-cloud, hybrid | **Terraform** (OpenTofu acceptable) |
| Azure-only, org standard is MSFT | Bicep |
| AWS-only, TypeScript/Python dev teams | CDK (CloudFormation under the hood) |
| Simple AWS templates | CloudFormation (SAM for serverless) |
| Policy-as-code | OPA/Conftest, or Sentinel (TF Cloud) |

## Terraform conventions

```text
□ Modules for everything reusable; version-pin module sources (git tags)
□ Remote state (S3+DynamoDB / azurerm backend / GCS) — never local state
□ One environment per state file/workspace; envs via tfvars, not copy-paste
□ Variables + outputs documented; types set
□ Secrets: never in .tf/.tfvars — reference vaults (data sources for
  Secrets Manager / Key Vault / Secret Manager)
□ Lint + validate in CI: terraform fmt -check, terraform validate, tflint
□ Plan is mandatory before apply; apply only from CI (or explicit user approval)
□ Tag everything (env/owner/cost-center/provider)
□ Drift detection: terraform plan in scheduled CI; reconcile or codify
□ Lifecycle guards: create_before_destroy for zero-downtime swaps;
  prevent_destroy on stateful resources
```

## Standard module layout

```text
environments/
├── prod/  main.tf · variables.tf · terraform.tfvars
├── staging/
└── dev/
modules/
├── vpc/        (network, subnets, NAT, endpoints)
├── compute/    (ASG/VMSS/MIG + LB)
├── database/   (RDS/SQL/Cloud SQL + backups + replicas)
└── observability/ (alarms, dashboards, budgets)
```

## Bicep equivalents (when chosen)

```text
az bicep build → compile/validate
az deployment group create --what-if → plan equivalent
az deployment group create --mode Complete (dangerous; prefer Incremental)
Deployment stacks → lifecycle-managed resources
Modules + AVM; parameters with @secure; keyVaultRef for secrets
```

## CDK equivalents (when chosen)

```text
cdk synth (→ CloudFormation), cdk diff (→ plan), cdk deploy
Constructs as the module analog; snapshots in tests
cdk-nag for security linting
```

## CloudFormation equivalents

```text
aws cloudformation validate-template
aws cloudformation create-change-set → review → execute
SAM (sam build/sam deploy) for serverless apps
```

## Post-deploy verification (always)

```text
□ terraform output vs live state: terraform plan → "No changes" expected
□ Smoke-test the endpoints (curl health check)
□ Verify monitoring/alarms exist and fire (test a threshold once)
□ Verify backups actually ran (check latest recovery point)
□ Record actual vs planned (cost, latency) in the ADR
```
