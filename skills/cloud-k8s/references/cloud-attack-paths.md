# Cloud Attack Paths — Per-Provider Privilege Escalation & Kill Chains

## AWS IAM privilege escalation (test each)

```text
□ iam:CreateAccessKey on another user → assume their identity
□ iam:PassRole + lambda:CreateFunction → run code as ANY role (classic)
□ iam:PassRole + ec2:RunInstances → SSRF the instance profile
□ sts:AssumeRole broad grants → hop across accounts/roles
□ iam:PutUserPolicy / iam:AttachUserPolicy → attach admin policy to self
□ iam:UpdateAssumeRolePolicy → rewrite a role's trust to include yourself
□ CloudFormation with PassRole → deploy a stack that creates privileged resources
□ SSM:StartSession / ssm:SendCommand on instances → on-host RCE
□ S3 GetObject on sensitive buckets; SQS/SNS subscribe to data flows
□ Lambda env vars: AWS keys, DB creds; DynamoDB streams for data access
```

Tooling: `pacu` (modules for each), `pmapper` (graph full escalation paths), `aws escalate` (RhinoSecurity research).

## Azure privilege escalation (test each — Entra ID is usually the crown jewels)

```text
□ User Access Administrator → assign any RBAC role to self
□ Owner on subscription → full control (obvious but verify depth of coverage)
□ Entra ID: app consent (users can consent to malicious apps) → tenant-wide data
□ Dynamic group rule manipulation (add self to admin group via attribute)
□ Administrative unit manipulation → group membership → admin
□ Custom role definitions with wildcard actions (/*)
□ Managed identity abuse: VM/Function MI with privileged roles
□ Key Vault access paths: contributor on KV → read all secrets
□ Automation accounts + runbooks → execute as higher-privileged identity
□ Application proxy / service principal with secrets in code
```

Tooling: `AzureHound` + BloodHound (identity attack paths), `ROADrecon` (Entra recon), `MicroBurst`, `PowerZure`, `Stormspotter` (Azure graph).

## GCP privilege escalation (test each)

```text
□ iam.serviceAccounts.getAccessToken → mint tokens for any SA
□ iam.serviceAccounts.actAs → act as a higher-privileged SA (Cloud Functions/Functions deploy)
□ iam.serviceAccounts.signJwt → forge SA JWTs
□ cloudfunctions.functions.create + SA → deploy function running as that SA
□ deploymentmanager.deployments.create → deploy resources as service account
□ pubsub subscriptions / storage triggers → data interception
□ roles/owner at org/folder level → downward inheritance everywhere
□ Org policy bypass chains (iam.allowedPolicyMemberDomains gaps)
```

Tooling: `gcloud` + `gcp_escalate` research (RhinoSecurity), ScoutSuite GCP module.

## Cross-provider metadata endpoints

```text
AWS:   http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>
Azure: http://169.254.169.254/metadata/identity/oauth2/token?api-version=...&resource=...
GCP:   http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
       (Header: Metadata-Flavor: Google)
```

SSRF → metadata = the #1 cloud initial-access bridge. Always test IMDSv1 vs v2 enforcement and SSRF reachability first.

## Classic kill chains (adversarially test these, don't just scan)

```text
1. SSRF → IMDS → node IAM role → SA enumeration → Secrets read → cluster-admin kubeconfig
2. Public S3 blob with .env / Terraform state → AWS keys → sts:AssumeRole chain → prod account
3. Compromised CI runner → kubeconfig in pipeline → pod-create rights → privileged pod → node escape
4. Lambda env secrets → DB creds → RDS snapshot exfil
5. Entra consent phishing (authorized test variant): over-broad app consent → Graph API → mail/OneDrive
6. GCP: leaked SA key (git) → getAccessToken on build SA → storage: read prod buckets
```

## Logging / detection review (blue-side check)

```text
□ AWS: CloudTrail org-wide? GuardDuty on? EventBridge alerts for AssumeRole anomalies?
□ Azure: Activity Logs → Log Analytics/Sentinel? Defender for Cloud alerts on?
□ GCP: Org-level Cloud Audit Logs sinks? SCC findings enabled?
□ K8s: audit logs enabled? Falco/runtime detection present?
Finding: "critical API calls not logged" is a real finding — not filler.
```

## Verification gate (per finding)

Every escalation claim must include: the exact command(s), the observed output proving the effect, and the remediation (condition keys, deny policies, PIM, MFA). No output → mark unverified, never confirmed.
