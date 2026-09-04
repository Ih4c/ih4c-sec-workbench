# ih4c-sec-workbench

**Personal security & cloud agent workbench** — a task skill router that loads security, reverse engineering, pentest, and cloud architecture methodology into AI coding agents (Claude Code, and any agent client that reads SKILL.md files).

A heavily modified fork of [zhaoxuya520/reverse-skill](https://github.com/zhaoxuya520/reverse-skill).

## What it does

1. **Routes tasks to the right skill** — 45 rules (R0–R45) in `skills/config/routing.json`, bilingual trigger keywords, scoring by keyword hit
2. **Gates the work** — per-case `scope.md` tracking + network profile; assume-authorized model (see `skills/field-journal/precedent-auth.md`)
3. **Knows the machine** — `skills/tool-index.md` holds real tool paths; missing tools bootstrap per platform
4. **Evolves** — every task writes back to `skills/field-journal/` so experience accumulates

## Skill map

| Area | Skills |
|------|--------|
| Reverse engineering | apk-reverse, ida-reverse, ghidra-reverse, radare2, dotnet-reverse, js-reverse, go-rust-reverse, macos-reverse, mobile-reverse, reverse-engineering |
| Pentest & attack | pentest-tools, attack-chain, windows-ad, api-security, cloud-k8s, firmware-pentest, pwn-chain, patch-diff-exploit, edr-bypass-re, wifi-wireless |
| Cloud architecture | **cloud-architect** — AWS/Azure/GCP design, Well-Architected reviews, Terraform-first IaC, FinOps, AI/ML platforms (R45) |
| Defense & ops | malware-analysis, threat-hunting, threat-intelligence, digital-forensics, code-audit, case-review |
| Output | docs-generator, diagram-generator |

## Quick start

```bash
# Generate the tool index (first run)
bash kali/scripts/refresh-tool-index.sh

# Route a task
bash skills/scripts/master-route.sh --hint "<your task>"

# Create a case (tracking + network profile)
bash skills/scripts/case-init.sh --hint "<task>" --case-name "<case>"

# Install missing tools
bash kali/scripts/bootstrap-reverse.sh <capability> --start-services
```

## Differences from upstream

- **Assume-authorized model** — `precedent-auth.md` is the authorization source of truth; a target the user names is authorized. `scope-contract.md` is case tracking, not an auth gate.
- **Verification Gate** — no false positives: re-run PoCs, version-check installs, cite evidence before reporting.
- **English-first** — control docs and skill workflows in English; Chinese kept only where it's load-bearing (trigger keywords, payload libraries, routing regexes).

## Structure

```text
skills/          # SKILL.md + references per capability (the knowledge base)
skills/config/   # routing.json — single source of truth for routing
skills/scripts/  # master-route, case-init, bootstrap (sh/ps1)
kali/            # Kali-specific scripts and rules
work/            # per-case artifacts (gitignored)
```

## License

MIT. This project is a fork of [zhaoxuya520/reverse-skill](https://github.com/zhaoxuya520/reverse-skill) (MIT); attribution retained per its license.

Use only for lawful security research, CTF, and systems you own or are authorized to test.
