# Domain Coverage Map (depth-first)

> Compared with the community's "hundreds of micro skills": we cover the main battlefields with **a few deep skills + routing + ops**.  
> Date: 2026-07-18

## Domain → Entry Point in This Package

| Domain | PRIMARY / module | Notes |
|----|----------------|------|
| Mobile Android | `apk-reverse/` `mobile-reverse/` | |
| Mobile iOS | `mobile-reverse/` | |
| Deep binary analysis | `ida-reverse/` `radare2/` `ghidra-reverse/` | Ghidra = main open-source path |
| General RE / anti-debug / OLLVM | `reverse-engineering/` | |
| .NET | `dotnet-reverse/` | |
| Frontend JS / signatures | `js-reverse/` | |
| Browser extensions | `browser-extension-reverse/` | |
| DSL/risk-control VM | `reverse-engineering/dsl-vm-reverse/` | |
| Protocols / PCAP protocol analysis | `protocol-reverse/` | |
| Firmware IoT | `firmware-pentest/` | |
| Malicious samples | `malware-analysis/` | |
| Digital forensics / IR | `digital-forensics/` | |
| Threat hunting / blue team | `threat-hunting/` | |
| Pentest tooling | `pentest-tools/` (+ src-hunter) | |
| Windows / AD | `windows-ad/` | |
| Cloud / containers / K8s | `cloud-k8s/` | |
| Code audit / SAST | `code-audit/` | |
| Wi-Fi / wireless | `wifi-wireless/` | |
| OT / ICS | `ot-ics/` | Passive-first; writing registers forbidden by default |
| macOS | `macos-reverse/` | iOS still goes to mobile-reverse |
| Thick clients | `thick-client/` | |
| Go / Rust binaries | `go-rust-reverse/` | |
| Hardware debug ports | `hardware-security/` | Hand off to firmware-pentest |
| Databases | `database-security/` | |
| Email / phishing | `email-security/` | |
| Federated identity SSO | `identity-federation/` | Complements api-security JWT |
| RF / SDR | `radio-sdr/` | Receive-only by default; non-Wi-Fi |
| Multi-stage attacks | `attack-chain/` | |
| Pwn | `pwn-chain/` | |
| N-day patches | `patch-diff-exploit/` | |
| EDR research | `edr-bypass-re/` | |
| API | `api-security/` | |
| Supply-chain SBOM | `supply-chain-security/` | |
| LLM/Agent | `llm-security/` | + `ops/skill-supply-chain.md` |
| Browser automation | `browser-automation/` | |
| Reports / diagrams | `docs-generator/` `diagram-generator/` | |
| Symbol migration | `binary-diff/` | |
| Operation contracts | `ops/` | **Distinctive** |
| CTF orchestration | `CTF-Sandbox-Orchestrator/` | |
| Crypto pattern recognition | `reverse-engineering` pattern documents | Shared with reverse tasks; no standalone extension package maintained |

## Domains Explicitly Not Merged Wholesale (strategy on route miss)

| Domain | Strategy |
|----|------|
| Pure game-cheat development | Not a product direction; Unity samples may still go through `reverse-engineering` + seed-014 |
| Deep automotive/aviation certification-grade | Can be external-linked; this package only has RF/OT entry-level |
| Pure GRC/compliance long-form | Does not replace professional GRC tools; report templates may cite it |
| 800+ ATT&CK micro skills | Use this table + optional ATT&CK tags (Finding fields) |

## On MITRE ATT&CK (optional)

The Finding template permits `optional_attack: Txxxx` (see `ops/evidence-finding-path.md`); a full ATT&CK engine is **not** required.
