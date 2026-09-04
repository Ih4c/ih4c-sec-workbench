# reverse-skill PRIMARY fast path

> `scripts/master-route.ps1` and `scripts/master-route.sh` must keep the same routing contract; the platform only changes the execution entry, never the routing semantics.

## Execution Contract

```text
1. Route first, act second
2. Output the PRIMARY path + a one-line rationale
3. case-init / scope.md (ops/scope-contract) — case tracking + network profile; authorization per field-journal/precedent-auth.md (assume authorized)
4. Assign lead + specialist roles (ops/role-map)
5. Immediately open the PRIMARY SKILL.md → ACTION REQUIRED
6. Tool paths are only trusted from tool-index; bootstrap when missing (manifest capabilities only)
7. Append to timeline / workitems as you go; conclusions go Evidence→Finding→Path
8. No match → read the full routing.md table or propose a new skill
```

### Windows

```powershell
powershell -File skills\scripts\master-route.ps1 -Hint "<用户任务>"
# defaults to writing the current project's work/master-route-<ts>/route-scope.md; specify the project root explicitly when calling from another directory
powershell -File skills\scripts\master-route.ps1 -Hint "<用户任务>" -ProjectRoot "C:\path\to\analysis-project"
powershell -File skills\scripts\case-init.ps1 -Hint "<用户任务>" -CaseName "my-case"
# cases default to the current project's work/<case>/; -PackageRoot kept for compatibility, -ProjectRoot takes precedence
powershell -File skills\scripts\case-init.ps1 -Hint "<用户任务>" -CaseName "my-case" -ProjectRoot "C:\path\to\analysis-project"
# one-shot readiness for ACT (authorization + target + network profile):
powershell -File skills\scripts\case-init.ps1 -Hint "<任务>" -CaseName "my-case" -AuthGranted -TargetUrl "https://target/" -NetworkProfile authorized_target_only
# local offline sample:
powershell -File skills\scripts\case-init.ps1 -Hint "offline apk" -CaseName "my-sample" -Preset offline-sample -Sample ".\app.apk"
# smoke: verify + script parsing + routing matrix (including Chinese hints)
powershell -File skills\scripts\smoke.ps1
# lightweight scope pre-check before ACT (exit 2 when not ready; -Force is a compatibility parameter and cannot bypass the network profile)
powershell -File skills\scripts\case-guard.ps1 -CaseRoot work\my-case
# Evidence append
powershell -File skills\scripts\append-evidence.ps1 -CaseRoot work\my-case -Id E-001 -Title "..." -ReproCommand "..."
python3 skills/case-review/scripts/review_case.py work/<case> --verify-hashes --strict
```

### Linux / macOS / Kali

PowerShell is not required for the core route/case flow:

```bash
bash skills/scripts/master-route.sh --hint "<用户任务>"
bash skills/scripts/master-route.sh --hint "<用户任务>" --project-root "/path/to/analysis-project"
bash skills/scripts/case-init.sh --hint "<用户任务>" --case-name "my-case"
bash skills/scripts/case-init.sh --hint "<用户任务>" --case-name "my-case" --project-root "/path/to/analysis-project"
# local offline sample:
bash skills/scripts/case-init.sh --hint "offline apk" --case-name "my-sample" --preset offline-sample --sample ./app.apk
# lightweight scope pre-check before ACT (--force is a compatibility parameter and cannot bypass the network profile):
bash skills/scripts/case-guard.sh --case-root work/my-sample
# routing parity:
bash skills/scripts/test-routing.sh
bash skills/scripts/test-bootstrap-manifest.sh
python3 skills/case-review/scripts/review_case.py work/<case> --verify-hashes --strict
```

## Operations Contract (ops)

| Document | Purpose |
|------|------|
| `ops/IDENTITY.md` | We are a routing package, not a Z3r0 platform |
| `ops/scope-contract.md` | case tracking template + network profile |
| `ops/evidence-finding-path.md` | evidence chain |
| `case-review/SKILL.md` | Evidence graph review and report handoff |
| `ops/role-map.md` | role→skill |
| `ops/timeline-workitem.md` | timeline and coverage |
| `ops/sandbox-profile.md` | tool mapping |
| `ops/skill-supply-chain.md` | security gate for installing external skills/MCP |
| `references/community-security-skills.md` | community skill ecosystem (borrow, do not merge into this library) |
| `reverse-engineering/references/re-agent-workflow.md` | RE: triage→static→dynamic→synthesis |
| `pentest-tools/references/recon-pipeline.md` | authorized recon pipeline + evidence gates |

## Priority (high → low)

> The order must match the `priority` array in `config/routing.json`. To change routing, change only the JSON, then update this table. `verify-routing-coherence.ps1` parses this table.

| ID | Condition | PRIMARY |
|----|------|---------|
| **R4** | DSL VM / fireye / custom opcode VM | `reverse-engineering/dsl-vm-reverse/` |
| **R1** | APK / smali / jadx / apktool | `apk-reverse/` |
| **R2** | IPA / iOS / Objection / MobSF / mobile | `mobile-reverse/` |
| **R3** | JS signing / frontend encryption / jshook / CDP | `js-reverse/` |
| **R30** | browser extension reverse engineering | `browser-extension-reverse/` |
| **R31** | macOS / Mach-O | `macos-reverse/` |
| **R33** | Go / Rust binaries | `go-rust-reverse/` |
| **R5** | .NET / dnSpy / de4dot / ConfuserEx | `dotnet-reverse/` |
| **R9** | malware samples / YARA / sandbox | `malware-analysis/` |
| **R21** | protocols / Protobuf / PCAP protocol | `protocol-reverse/` |
| **R22** | Ghidra / open source decompilation | `ghidra-reverse/` |
| **R6** | IDA / decompilation / deep disassembly | `ida-reverse/` |
| **R7** | radare2 / r2 | `radare2/` |
| **R8** | firmware / binwalk / IoT / EMBA | `firmware-pentest/` |
| **R34** | hardware debug ports / UART/JTAG | `hardware-security/` |
| **R28** | OT / ICS / industrial control | `ot-ics/` |
| **R17** | pwn / ROP / stack exploitation | `pwn-chain/` |
| **R16** | N-day / patch diffing | `patch-diff-exploit/` |
| **R18** | EDR / AV evasion / syscall | `edr-bypass-re/` |
| **R24** | Windows / AD / Kerberos / AD CS | `windows-ad/` |
| **R37** | federated identity SAML/OIDC | `identity-federation/` |
| **R23** | cloud / containers / K8s | `cloud-k8s/` |
| **R45** | cloud architecture / solution architecture / IaC / FinOps | `cloud-architect/` |
| **R35** | database security | `database-security/` |
| **R25** | forensics / memory dumps / timelines | `digital-forensics/` |
| **R44** | OSINT / threat intelligence / public X IOC enrichment | `threat-intelligence/` |
| **R36** | email / phishing analysis | `email-security/` |
| **R29** | Wi-Fi / wireless pentest | `wifi-wireless/` |
| **R38** | RF / SDR research | `radio-sdr/` |
| **R32** | thick client security | `thick-client/` |
| **R26** | code audit / SAST / Semgrep | `code-audit/` |
| **R27** | threat hunting / detection engineering / blue team | `threat-hunting/` |
| **R10** | attack chain / red team / lateral movement / full pentest | `attack-chain/` |
| **R11** | Nmap / Nuclei / SQLMap / SRC / pentest tools | `pentest-tools/` |
| **R12** | API / GraphQL / BOLA / JWT attacks | `api-security/` |
| **R13** | SBOM / Trivy / supply chain | `supply-chain-security/` |
| **R14** | LLM / Prompt injection / Agent security | `llm-security/` |
| **R15** | bindiff / symbol migration / PDB | `binary-diff/` |
| **R19** | browser/desktop automation | `browser-automation/` |
| **R40** | Case / Evidence graph review | `case-review/` |
| **R20** | reports / writeups | `docs-generator/` |
| **R39** | diagrams / Mermaid / Graphviz / PlantUML / architecture diagrams | `diagram-generator/` |
| **R41** | CTF / AWD / ranges (single entry, no expansion into 40 sub-skills) | `ctf-sandbox/` |
| **R0** | generic reverse / anti-debug / OLLVM / unknown binaries | `reverse-engineering/` |

No strong keyword match → PRIMARY=`R0`, and prompt the user to open `routing.md` (ambiguity appendix, not a second router).

## Boundaries

| Task | Handling |
|------|------|
| pure CTF multi-category orchestration | PRIMARY `ctf-sandbox/` → sidecar `../CTF-Sandbox-Orchestrator/` |

## Reading Order

```text
RULES.md → MASTER-ROUTING.md → PRIMARY SKILL.md
  → (optional) routing.md three axes / field-journal
  → tool-index.md → bootstrap → ACT
```
