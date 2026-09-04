# Cloud / K8s Test Checklist (full)

## IMDS / metadata

- [ ] SSRF reachable to 169.254.169.254 / metadata.google.internal from web surface?
- [ ] IMDSv2 enforced account-wide (AWS)? hop-limit set to 1?
- [ ] Azure IMDS: managed identity endpoints reachable from SSRF? api-version filters?
- [ ] GCP: Metadata-Flavor header requirement bypassed anywhere?
- [ ] Instance/node role permission surface (what does the leaked role actually grant?)

## IAM / identity

- [ ] Least-privilege? unused roles/credentials/key age/rotation?
- [ ] Privilege-escalation paths tested per provider (see `cloud-attack-paths.md`)
- [ ] Cross-account/subscription/project trust restricted with conditions?
- [ ] No long-lived access keys; MFA on humans; SCPs/org policies/deny assignments
- [ ] Federated identity: SAML/OIDC trust policies over-permissive?

## Storage

- [ ] Public S3 buckets / Blob containers / GCS objects? (block-public-access actually on?)
- [ ] ACL vs policy conflicts; pre-signed URL exposure windows
- [ ] Encryption at rest + in transit; SSE-KMS/CMEK where compliance demands
- [ ] Object access logging (CloudTrail S3 data events / blob diagnostics / GCS logs)?

## Network / perimeter

- [ ] Security groups / NSGs / firewall rules open to 0.0.0.0/0 (SSH/RDP/admin consoles)?
- [ ] VPC/VNet config: exposed management interfaces, default VPCs in use?
- [ ] East-west traffic encrypted/authenticated? (or flat and open)
- [ ] WAF on public endpoints? private endpoints used for PaaS?

## Containers

- [ ] Running as root? readOnlyRootFilesystem? drop ALL capabilities default?
- [ ] privileged / hostPID / hostNetwork / hostPath mounts
- [ ] Writable /var/run/docker.sock, /run/containerd/containerd.sock
- [ ] CAP_SYS_ADMIN + unconfined AppArmor → cgroup release_agent escape
- [ ] seccomp/AppArmor/gVisor/Kata actually enforced (not Unconfined)?
- [ ] Runtime CVEs: CVE-2019-5736 (runc), CVE-2024-21626 (runc), CVE-2022-0185,
      CVE-2024-1086 (kernel), CVE-2020-15257 (containerd) — version-check
- [ ] Image scanning (Trivy) + registry credential/pull-secret exposure

## Kubernetes

- [ ] cluster-admin / admin bindings over-broad (who is bound?)
- [ ] escalate / bind / impersonate verbs available to any non-admin subject
- [ ] system:anonymous / system:unauthenticated bound to permissive roles; anonymous auth on
- [ ] Default SAs: automountServiceAccountToken true on non-API workloads
- [ ] Pod-create rights + privileged SA combination → privileged pod path
- [ ] Secrets: base64-only (no etcd encryption at rest); cluster-wide secret read possible?
- [ ] etcd direct access (2379/2380) — unauthenticated v2 API?
- [ ] Kubelet 10250 / read-only 10255 anonymous endpoints (/pods, /spec)
- [ ] Dashboard / API server exposed without auth
- [ ] ValidatingWebhookConfiguration mutation rights (intercept every API request)
- [ ] NetworkPolicies: default deny? flat networking lets any pod reach any pod + metadata
- [ ] CronJobs / DaemonSets as persistence (backdoor containers, shadow API servers)
- [ ] Admission controllers: PSP/PSS enforcement? mutable webhook configs?

## Serverless

- [ ] Lambda/Functions/Cloud Run env vars with secrets (DB creds, keys)
- [ ] Function roles over-privileged (one over-permissive Lambda role → whole account)
- [ ] API Gateway/APIM auth gaps (no auth on internal endpoints)
- [ ] Cold-start config, VPC attachment, public exposure
- [ ] CloudFormation/ARM/Deployment Manager templates with embedded secrets

## Secrets management / CI-CD

- [ ] Hardcoded creds in: repos (TruffleHog/Gitleaks), user data, CI logs, IaC state files
- [ ] Secrets Manager / Key Vault / Secret Manager access scoping (who can read all?)
- [ ] CI/CD: kubeconfig in pipelines, IRSA/workload identity vs static keys
- [ ] Terraform state files public (S3/blob/GCS) with secrets?

## Logging & detection

- [ ] CloudTrail / Activity Logs / Cloud Audit Logs: critical API calls logged, tamper-proof?
- [ ] Alerts firing for: AssumeRole anomalies, new admin assignments, public bucket creation?
- [ ] K8s audit logs on? runtime detection (Falco) present?

## Tools quick map

```text
Multi-cloud: Prowler, ScoutSuite, CloudFox
AWS:  pacu, pmapper, RhinoSecurity escalate
Azure: AzureHound, ROADrecon, MicroBurst, Stormspotter
GCP:  gcloud + gcp_escalate, ScoutSuite GCP
K8s:  kube-hunter, kubescape, Kraken, rbac-tool, kubectl-who-can
Secrets: TruffleHog, Gitleaks
```

## Reminder (blind-spot protection)

Scanners find single misconfigurations. The pentest job is **chains**: test the multi-step kill chains in `cloud-attack-paths.md`, verify each escalation by re-running it, and report the chain — not just the single findings.
