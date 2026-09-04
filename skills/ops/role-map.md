# Expert Role → Skill Mapping (no multi-agent server)

> Role codes are inspired by the Z3r0 expert team; the **implementation** is the reverse-skill routing and handoff protocol, not process orchestration.

## Role Table

| Code | Name (localizable) | Responsibilities | PRIMARY / tool skill |
|------|------------------|------|----------------------|
| **lead** | Lead / overall commander | split tasks, set scope, phase gates, aggregate the report | `attack-chain/` or the current PRIMARY hub; at the end → `docs-generator/` |
| **cie** | intelligence collection | asset discovery, exposure, relationships | `pentest-tools/` (recon); browser → `browser-automation/`; cloud surface → `cloud-k8s/` |
| **cpe** | penetration validation | scanning, exploit validation, impact confirmation | `pentest-tools/`; API → `api-security/`; AD → `windows-ad/`; wireless → `wifi-wireless/`; databases → `database-security/`; SSO → `identity-federation/`; OT → `ot-ics/` |
| **cre** | reverse analysis | binary/firmware/mobile/frontend logic | `ida-reverse/` `ghidra-reverse/` `radare2/` `apk-reverse/` `mobile-reverse/` `macos-reverse/` `js-reverse/` `browser-extension-reverse/` `dotnet-reverse/` `go-rust-reverse/` `firmware-pentest/` `hardware-security/` `malware-analysis/` `protocol-reverse/` `thick-client/` `reverse-engineering/` |
| **cae** | code audit | source code/dependencies/supply chain | `code-audit/` + `supply-chain-security/` |
| **cbe** | blue team/forensics | hunting, detection, IR artifacts | `threat-hunting/` `digital-forensics/` |
| **cce** | cryptography | algorithms/protocols/key misuse | `reverse-engineering` pattern documents |
| **llm** | AI security | Prompt/Agent | `llm-security/` |
| **doc** | documentation officer | reports/writeups/diagrams | `docs-generator/` + `diagram-generator/` |

## Mandatory Lead Protocol

```text
1. Output PRIMARY (master-route) + lead_role=lead
2. Write scope.md (ops/scope-contract)
3. Assign specialist_roles[] and handoff conditions
4. At the end of each phase: update timeline + workitems; decide to continue / switch roles / produce a report
5. Forbidden to skip scope and go straight to cpe scanning production
```

## Handoff Rules

| From → To | Trigger | Deliverable |
|---------|------|--------|
| lead → cie | asset surface needed | scope + known domains/IPs |
| cie → cpe | live surface/services exist | assets list + ports/URLs |
| cpe → cre | reverse verification / client logic needed | sample paths + suspicious points |
| cre → cpe | protocol/key/check reconstructed | algorithm description + repro commands |
| any → doc | phase or task complete | Evidence/Finding/Path drafts |
| any → lead | blocked/out of scope/path change | timeline note + blocked reason |

## How a Single Agent Uses This (flavor)

No need to actually spawn 6 agents:

```text
Within the same session:
  [lead] plan
  [cie] run the recon skill
  [cpe] switch to pentest-tools
  …
Prefix output with the role tag for easy timeline retrieval:
  [cpe] nuclei high findings → E-003
```

## Relationship with master-route

- `master-route` determines the **PRIMARY skill**  
- `role-map` determines **who is responsible in the current phase** (can be written into scope.md)  
- For multi-phase tasks the PRIMARY is usually `attack-chain/`, then redistributed by the lead  

## MUST NOT

- Do not assume a Z3r0 session API exists  
- Do not launch extra scans against unauthorized targets in any role  
