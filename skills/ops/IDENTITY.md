# reverse-skill Identity Manifest (relative to Z3r0)

> This file fixes **who we are**. We absorb Z3r0's Evidence/scope/role/timeline ideas, but do **not** become a Z3r0 platform.

## What We Are

| Dimension | reverse-skill |
|------|----------------|
| Form | **Skill routing package** — methodology + tool bootstrapping for any AI client (Claude/Cursor/Codex…) |
| Entry | `RULES.md` → `MASTER-ROUTING` / `master-route.ps1` → sub-skills |
| Tool truth | `tool-index.md` + `bootstrap-manifest.json` (local machine paths, no guessing) |
| Evolution | `field-journal/` sanitized experience write-backs |
| Output | Markdown reports + `work/<case>/` local case directories (gitignored) |
| Deployment | `git clone` only; no mandatory PG/UI/Docker pool |

## What We Are Not

| Z3r0 has | reverse-skill **deliberately does not** |
|---------|---------------------------|
| React operations console | ❌ |
| FastAPI control plane + WebSocket sessions | ❌ |
| PostgreSQL evidence store | ❌ |
| LightRAG service | ❌ |
| Docker host pool / noVNC control proxy | ❌ (may **document** recommended optional sandbox profiles) |
| Multi-agent process runtime | ❌ (only **role→skill mapping + handoff protocol**) |

## What We Learn From Z3r0 (downscaled implementation)

| Idea | reverse-skill form |
|------|-------------------|
| Authorization and project boundaries | `ops/scope-contract.md` → per-case `scope.md` |
| Evidence→Finding→Path | `ops/evidence-finding-path.md` + report templates |
| Expert division of labor | `ops/role-map.md` (Lead/cie/cpe/cre…→ skill) |
| Replayability | append-only `work/<case>/timeline.md` |
| WorkItem/coverage | `workitems.md` + coverage checkboxes |
| Sandbox tool readiness | `ops/sandbox-profile.md` vs bootstrap-manifest |
| Egress control | `network_profile` field (offline/lab/authorized) |

## Distinctive Features (must keep)

1. **Three-axis routing + PRIMARY fast path** (target type / intent / toolchain)  
2. **Bootstrap installs tools on demand**, across Windows/Kali/Linux/macOS  
3. **MCP-friendly** (IDA/Burp/jshook/anything-analyzer)  
4. **field-journal sanitized evolution**  
5. **Obedience engineering**: ACTION REQUIRED / completion self-checks / fake pauses forbidden  

## Healthy Relationship with Z3r0

```text
Z3r0 = red team operating system / team collaboration platform
reverse-skill = Agent security operation router + manual

Optional future: mount this package's skills inside Z3r0 sandbox-local skills
Current: zero-dependency on Z3r0 installation for fully functional operation
```

## Relationship with the "800+ Community Micro-Skills"

- Do **not** submodule the giant skill libraries (poisoning surface and maintenance cost, see `skill-supply-chain.md`)  
- Do **maintain** `references/community-security-skills.md` as an index and borrowing-rules reference  
- Do **use** `domain-coverage-map.md` to prove: deep skills + routing > fragmented skill stacks  
- External skill installation: AST10 mindset + only curated sources (e.g. Trail of Bits curated)  
