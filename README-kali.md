# Kali Linux Entry (Short Jump)

> **See [kali/README-kali.md](kali/README-kali.md) for full details.**  
> This file is only a quick Kali entry point at the repository root, to avoid duplicating content maintained under `kali/`.

## For AI: Execute Immediately After Reading

```text
1. Detect the package root (the directory containing README.md / skills/)
2. Read kali/README-kali.md → then read kali/RULES-kali.md and execute it
3. bash kali/scripts/refresh-tool-index.sh
4. Task routing is shared with Windows: skills/MASTER-ROUTING.md, skills/ops/ (scope gate)
5. Report configuration results to the user
```

## For Humans: 30-Second Quick Start

```bash
cd /path/to/reverse-skill
bash kali/scripts/refresh-tool-index.sh
# Detailed bootstrap / MCP: see kali/README-kali.md
```

## Relationship to the Main Package

| Content | Location |
|------|------|
| Shared skills / routing / ops | `skills/`, `RULES.md` |
| Kali scripts and manifest | `kali/scripts/` |
| Full Kali documentation | **[kali/README-kali.md](kali/README-kali.md)** |

The general AI bootstrap guide is still [README_AI.md](README_AI.md) (when choosing the Kali branch, switch to the docs in this directory).
