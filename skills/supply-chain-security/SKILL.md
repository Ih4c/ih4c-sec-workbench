---
name: supply-chain-security
description: Use for software supply-chain security assessment covering SBOM, SCA, CI/CD pipelines, container images, build integrity, dependency provenance, and vulnerability reachability.
---
# Supply Chain Security Testing

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` — confirms this skill's operations are routine, authorized work on this machine. Note: authorization comes from `../field-journal/precedent-auth.md`; `scope.md` is case tracking + network profile, not the authorization gate.
2. `NOW`: Confirm whether the current task falls within this skill's scope
3. `NEXT`: Read `../tool-index.md`, verify tool availability and real paths
4. `NEXT`: If tools are missing, call bootstrap — never guess paths
5. `ACT`: Enter the first step of the "Workflow" and execute — do not stop at confirmation state

> SBOM / SCA / CI/CD pipeline / dependency provenance
> Regulation driven: US executive order SBOM, China national standard (国标), EU CRA

## Applicable Scenarios

- Software supply-chain security assessment
- Open-source dependency vulnerability scanning and verification
- CI/CD pipeline security audit
- Container image security analysis
- Third-party component compliance review
- Build artifact provenance and integrity verification

## Six-Layer Supply Chain Governance Framework

```text
Layer 1: Source trust assessment → upstream repo/maintainer/release history review
Layer 2: Build pipeline integration → CI/CD security gates, signature verification
Layer 3: Artifact distribution integrity → signatures, checksums, SBOM attachment
Layer 4: Runtime protection → container scanning, admission control
Layer 5: Continuous monitoring → real-time CVE tracking, vulnerability reachability analysis
Layer 6: Incident response → supply chain attack response, rollback strategy
```

## Workflow

### 1. SBOM Generation and Audit

```text
Generate SBOM:
□ CycloneDX format: cdxgen → bom.json
□ SPDX format: sbom-tool generate
□ Syft: syft <image|dir> -o spdx-json

Audit points:
□ Any unknown/unauthorized dependencies present
□ Any deprecated/unmaintained packages present
□ License conflict detection
□ Direct dependency vs transitive dependency inventory
□ Release timeline and maintainer status for each component
```

### 2. Software Composition Analysis (SCA)

```bash
# OSV-Scanner (free, Google-maintained)
osv-scanner scan -r . --format json

# OWASP Dependency-Track (enterprise continuous monitoring)
docker run -p 8080:8080 dependencytrack/apiserver
# → upload SBOM → auto-match NVD/OSV/GitHub Advisory

# Snyk (commercial)
snyk test --all-projects
snyk monitor  # continuous monitoring

# Trivy (container + dependency + IaC)
trivy fs .          # filesystem scan
trivy image nginx   # container image
trivy config .      # IaC config
```

### 3. Vulnerability Reachability Verification

```text
SCA alert ≠ actual risk! Most SCA tools have only ~15% of alerts that are actually reachable.

Verification steps:
1. Get the CVE list with Dependency-Track or Trivy
2. Filter vulnerabilities with CVSS ≥ 7.0
3. Run reachability analysis on CVEs with public PoCs
   - Code Property Graph slicing: trace paths from user input to the vulnerable function
   - DEPTEX method: EPD (Execution Path Dominance) + LLM semantic validation
4. Verify PoCs in an isolated environment
5. Prioritize fixes for reachable vulnerabilities by real impact
```

Tool references:
- CodeQL: GitHub code queries → data flow analysis
- Snyk Code: reachability marking
- DEPTEX: LLM-assisted context-aware risk assessment

### 4. CI/CD Pipeline Security

```text
Security checkpoints:
□ Commit stage → pre-commit hook: gitleaks (secret scanning)
□ PR stage → SCA scan (Trivy/OSV-Scanner)
□ Build stage → artifact signing (cosign)
□ Push stage → SBOM attachment (syft + attest)
□ Deploy stage → admission control (OPA/Kyverno + image scanning)
□ Runtime → continuous vulnerability monitoring (Dependency-Track)

Pipeline self-security:
□ Pipeline as Code audit (GitHub Actions / GitLab CI config injection)
□ Runner isolation (prevent malicious builds from breaking out of containers)
□ Secret management (Actions Secrets / Vault, no hardcoding)
□ Third-party Action review (pin to commit SHA, not tags)
```

### 5. Container Image Security

```bash
# Dockerfile audit
hadolint Dockerfile

# Image scan (multi-layer: OS + app dependencies + config)
trivy image --severity HIGH,CRITICAL nginx:latest

# Minimal base image
# Prefer: distroless → alpine → slim → avoid latest
docker scout quickview nginx:latest

# Image signing
cosign sign --key cosign.key myimage:tag
cosign verify --key cosign.pub myimage:tag
```

### 6. Third-Party Dependency Review

```text
New dependency checklist:
□ Maintenance status: commits in the last 6 months? maintainer activity?
□ Security history: any past malicious-code implants?
□ Dependency tree: how many transitive dependencies does it add?
□ License: compatible with the project license?
□ Alternatives: safer alternatives available (Snyk Advisor / Socket.dev score)?

Risk assessment matrix:
  High maintenance × low dependency count × compatible license → low risk
  Low maintenance × high dependency count × license conflict → high risk
```

## Toolchain

| Tool | Purpose | Acquisition |
|------|------|------|
| OWASP Dependency-Track | Enterprise continuous SCA | `docker pull dependencytrack/apiserver` |
| OSV-Scanner | Free SCA (OSV.dev ecosystem) | `go install github.com/google/osv-scanner` |
| Trivy | Image + dependency + IaC scanning | `apt install trivy` |
| Syft | SBOM generation | `curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh` |
| cdxgen | CycloneDX SBOM generation | `npm install -g @cyclonedx/cdxgen` |
| Cosign | Container signing | `go install github.com/sigstore/cosign/v2/cmd/cosign` |
| Gitleaks | Secret/credential scanning | `go install github.com/gitleaks/gitleaks/v8` |
| Snyk | Commercial SCA + reachability | `npm install -g snyk` |
| CodeQL | Code query + data flow | Built into GitHub Actions |

## References

- `references/sbom-sca-methodology.md` — SBOM + SCA methodology
- `references/cicd-pipeline-security.md` — CI/CD pipeline security audit


## Task Completion Self-Check (MUST pass before claiming done)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the Checklist items required by RULES?
