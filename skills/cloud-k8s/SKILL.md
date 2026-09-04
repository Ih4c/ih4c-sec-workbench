---
name: cloud-k8s
description: Cloud, container, and Kubernetes penetration testing across AWS, Azure, and GCP — metadata/IMDS SSRF, IAM/Entra ID/Cloud IAM privilege escalation, exposed storage, serverless and CI/CD attacks, container escape, and K8s RBAC/secret/etcd exploitation. Triggers: cloud pentest, cloud security, AWS pentest, Azure pentest, GCP pentest, IAM privesc, container escape, K8s attack, 云安全, 容器逃逸.
---

# Cloud / Container / Kubernetes Pentest

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` — routine operation reference; authorization assumed for the target the user named
2. `NOW`: case-init → scope.md (tracking + network profile). **Cloud/K8s rules: stay inside the named account/subscription/project/cluster; no destructive actions; no scanning other tenants**
3. `NOW`: Identify the surface: cloud control plane (AWS/Azure/GCP), Kubernetes cluster, containers, or serverless — not generic web scanning (that's `pentest-tools/`)
4. `NEXT`: tool-index; kubectl/aws/az/gcloud/pacu/prowler are mostly manual installs
5. `ACT`: Start from **identity & exposure**, never full-blast network scanning
6. `ACT`: Work the phases below; every finding → Evidence (command + output) → Finding
7. `MUST`: Verify before reporting — re-run the escalation step and cite the observed effect (verification gate, no false positives)

## Use cases

- Cloud metadata SSRF (169.254.169.254 / metadata.google.internal)
- IAM / Entra ID / Cloud IAM over-privilege and privilege-escalation paths
- Public buckets / blobs / GCS objects, bad ACLs, unencrypted data
- Docker/containerd escape paths
- Kubernetes RBAC, Secrets, admission controllers, etcd, exposed kubelet/API server
- Serverless (Lambda/Functions/Cloud Run) permissions and secret leaks
- CI/CD pipeline credential exposure (kubeconfig, IRSA tokens, IaC secrets)

## Workflow

### Phase 1 — Identity & boundary

```text
□ Current identity: cloud keys? K8s SA token? node SSH? federated identity?
□ Scope: single account / subscription / project / cluster / namespace
□ network_profile: authorized_target_only
□ Enumerate effective permissions FIRST (see references/cloud-attack-paths.md):
  AWS:   aws sts get-caller-identity; aws iam list-attached-user-policies ...
  Azure: az ad signed-in-user show; az role assignment list ...
  GCP:   gcloud auth list; gcloud projects get-iam-policy <project>
```

### Phase 2 — Cloud control plane (per provider)

```bash
# Examples (vendor-substitute; MUST stay inside the authorized account)
aws s3 ls                                   # bucket enumeration
aws sts get-caller-identity
az resource list --subscription <sub>
gcloud projects list; gcloud storage ls
```

```text
□ Public buckets/blobs/GCS + ACL misconfigs
□ IMDS: v1 vs v2 enforcement; SSRF chains to instance roles
□ Role assumption/impersonation: PassRole chains (AWS), User Access Administrator /
  app consent (Azure/Entra), SA impersonation + getAccessToken (GCP)
□ Cross-account / cross-subscription / cross-project trust relationships
□ Serverless: Lambda/Functions/Cloud Run permissions, env-var secrets
□ CI/CD: pipeline credentials, IaC template secrets, registry pull secrets
```

### Phase 3 — Containers

```text
□ privileged / hostPath / hostNetwork / hostPID
□ capabilities (SYS_ADMIN etc.) + seccomp/AppArmor actually enforced?
□ writable host paths → escape candidates (nsenter, cron/SSH key writes)
□ docker.sock / containerd.sock mounts → spawn host containers
□ image history + known CVEs → Trivy; registry credential exposure
□ runtime CVEs: check versions vs CVE-2019-5736 (runc), CVE-2024-21626,
  CVE-2022-0185, CVE-2020-15257 (containerd)
```

### Phase 4 — Kubernetes

```bash
kubectl auth can-i --list
kubectl get pods,secrets,svc -A
kubectl get clusterrolebindings,rolebindings -A
kubectl get networkpolicies -A
```

```text
□ RBAC blind spots (not just cluster-admin):
  - escalate / bind / impersonate verbs → self-cluster-admin
  - system:anonymous / system:unauthenticated bound to permissive roles
  - default SAs with automountServiceAccountToken: true on non-API workloads
  - pod-create rights + privileged SA binding → deploy privileged pod
□ Secrets: base64 ≠ encrypted; etcd at rest encryption on? who can read secrets cluster-wide?
□ etcd direct access (2379/2380); kubelet ports 10250/10255 anonymous endpoints
□ Webhook configs: who can modify ValidatingWebhookConfigurations?
□ Flat networking: no NetworkPolicies → any pod reaches any pod + metadata endpoint
  (cluster compromise → cloud account compromise bridge)
```

### Phase 5 — Impact chains & report

```text
□ Chain findings into attack paths (single-mediums → critical chain)
□ Impact: production data access, exfil, IaC backdoor, IAM backdoor
□ Report: evidence-cited per finding; escalation steps reproducible
□ Remediation per provider (IMDSv2 account-wide, SCPs, PIM, deny policies)
```

## Provider testing rules

| Provider | Rules |
|----------|-------|
| AWS | Most services testable without prior approval; simulated DDoS, DNS zone walking, port/protocol flooding need explicit authorization |
| Azure | Testing your own resources is fine; notify for high-traffic engagements; never scan outside your tenant |
| GCP | No prior approval needed; stay within your own project/organization, follow AUP |

## Tools

| Tool | Purpose | Install |
|------|---------|---------|
| Prowler / ScoutSuite / CloudFox | multi-cloud config review + enumeration | manual/pip |
| Pacu | AWS exploitation (privesc scan, S3 finder, IMDS) | manual |
| PMapper | AWS IAM escalation path graphing | manual |
| AzureHound / ROADrecon / MicroBurst | Azure/Entra identity attack paths | manual |
| Kraken / kube-hunter / kubescape | K8s offensive suite / scanning | manual |
| Trivy | image + IaC scanning | bootstrap if available |
| TruffleHog / Gitleaks | secret hunting in repos/CI | manual |
| kubectl / aws / az / gcloud | core enumeration CLIs | system/manual |

## References

- `references/k8s-cloud-checklist.md` — full test checklist (IMDS, IAM, storage, K8s, escapes, secrets)
- `references/cloud-attack-paths.md` — per-provider IAM privesc paths, kill chains, tools
- CTF comparison: `../../CTF-Sandbox-Orchestrator/competition-agent-cloud/`
- `../supply-chain-security/` `../pentest-tools/`

## Routing context

**Upstream**: MASTER R23
**Downstream**: node shell obtained → `attack-chain` / `windows-ad`; image vulns → supply-chain
**MUST NOT**: scan other tenants' public cloud assets without authorization

## Completion self-check

- [ ] Did I stay inside the authorized account/subscription/project/cluster?
- [ ] Do findings include repro + impact, every escalation re-verified?
- [ ] Did I avoid destructive actions?
- [ ] Report / journal completed?
