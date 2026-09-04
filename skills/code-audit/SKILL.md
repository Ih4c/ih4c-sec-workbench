---
name: code-audit
description: Use for authorized source-code security review and SAST workflows including Semgrep, CodeQL patterns, dangerous API hunting, and fix verification.
---

# Source Code Security Audit

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` or the code audit authorization
2. `NOW`: Confirm there is **source/repository access** (binary without source → switch to an RE skill)
3. `NOW`: Clarify the language stack and scope (directory/service/PR diff)
4. `NEXT`: tool-index; semgrep etc.
5. `ACT`: Threat model sketch → automated scanning → manual verification

## Applicable Scenarios

- White-box audits, PR/diff security reviews
- SAST with Semgrep / CodeQL / Bandit / gosec etc.
- Dangerous APIs, injection points, missing auth, crypto misuse
- Division of labor with `supply-chain-security/`: this skill focuses on **the code's own logic**; supply-chain covers dependencies and pipelines

## Workflow

### 1. Scope and Threat Model

```text
□ Trust boundaries: user input, files, deserialization, SSRF, auth middleware
□ High-value assets: auth, payments, admin panels, key handling
```

### 2. Automated Scanning

```bash
semgrep --config auto .
# or project rulesets
semgrep --config p/owasp-top-ten .
```

### 3. Manual Verification (MUST)

```text
□ For each SAST hit: reachable? exploitable? false positive?
□ Auth: IDOR/privilege escalation (越权), missing checks, broken multi-tenant isolation
□ Injection: SQL/command/template/LDAP
□ Crypto: hardcoded keys, ECB, custom crypto
```

### 4. Deliverables

```text
Finding: location + data flow + PoC + fix recommendation
Optional ATT&CK / CWE identifiers
```

## Toolchain

| Tool | Language/Scenario |
|------|-----------|
| Semgrep | Multi-language fast rules |
| CodeQL | Deep data flow (GitHub) |
| Bandit | Python |
| gosec / staticcheck | Go |
| SpotBugs / FindSecBugs | Java |

## References

- `references/sast-review-checklist.md`
- `../supply-chain-security/` `../api-security/` `../llm-security/` (Agent code)

## Routing Context

**Upstream**: MASTER R26
**Role**: `ops/role-map.md` cae
**Downstream**: dependency vulnerabilities → supply-chain; runtime verification → pentest-tools

## Task Completion Self-Check

- [ ] Was manual verification done rather than just pasting scanner output?
- [ ] Do the findings include fix recommendations?
- [ ] Was the work kept within the authorized repository scope?
- [ ] Checklist?
