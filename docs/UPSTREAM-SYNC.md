# Upstream Sync Policy

Upstream: [`zhaoxuya520/reverse-skill`](https://github.com/zhaoxuya520/reverse-skill) (git remote `upstream`).
This repository is a deliberate fork, not a mirror.

## Why we diverged

- **Assume-authorized model** — `field-journal/precedent-auth.md` is the authorization source of truth; `scope-contract.md` is case tracking, not an auth gate.
- **English-first** — control docs and workflows are English; Chinese kept only where load-bearing (trigger keywords, payload libraries, routing regexes, benchmark fixtures).
- **R45 `cloud-architect`** and an upgraded `cloud-k8s` — fork-local skills with no upstream counterpart.
- Deleted Chinese mirrors (`README_zh.md`, `docs/*_zh.md`, `RULES_zh.md`, `routing_zh.md`).
- Rewritten README and project identity (`ih4c-sec-workbench`).

Every one of these would conflict with an upstream merge. **Merging upstream wholesale is forbidden.**

## Iron rules

1. **Never** `git merge upstream/main` into `main`.
2. **Cherry-pick only new files** (new skills, new playbooks, new reference docs) — they land with zero conflicts.
3. **Port shared infrastructure by hand** (`routing.json`, `RULES.md`, `skills/SKILL.md`, `scripts/`, manifests) — review the upstream change, then re-implement the idea in English under this fork's model. Never take the file wholesale.
4. After any sync, re-run the verification gates (see `docs/RELEASE-CHECKLIST.md` / `.github/workflows/ci.yml`) and keep `INDEX.md` / `tool-index.md` regenerated.

## File categories

| Category | Files | How to sync |
|---|---|---|
| Safe to take wholesale | brand-new `skills/<new>/`, new playbooks/references that don't exist here | `git checkout upstream/main -- <path>` |
| Port by hand only | `skills/config/routing.json`, `RULES.md`, `skills/SKILL.md`, `MASTER-ROUTING.md`, `skills/scripts/*`, `kali/scripts/*`, `bootstrap-manifest.json` | read the upstream diff, port the intent |
| Do not take | `README.md`, `docs/OVERVIEW.md`, field-journal, precedent files, deleted `*_zh.md` mirrors | fork-local identity and model |

## Workflow

```bash
# 1. Pull the catalog
git fetch upstream

# 2. See what changed
git diff main..upstream/main --stat
git log main..upstream/main --oneline

# 3. Take specific NEW files (no conflicts)
git checkout upstream/main -- skills/<new-skill>/ skills/pentest-tools/src-hunter/references/playbooks/<new-playbook>.md

# 4. Port shared-infra changes by hand, then verify locally
pwsh -NoProfile -ExecutionPolicy Bypass -File skills/scripts/verify-routing-coherence.ps1
pwsh -NoProfile -ExecutionPolicy Bypass -File skills/scripts/verify-doc-facts.ps1
pwsh -NoProfile -ExecutionPolicy Bypass -File skills/scripts/extract-summaries.ps1   # regenerate INDEX.md
bash skills/scripts/master-route.sh --hint "<sample task>"                           # router smoke

# 5. Commit on main and push
```

## Route-ID divergence (important)

Route IDs are fork-local and no longer aligned with upstream:

- Upstream `R45` = `binary-ninja-reverse/`; **our `R45` = `cloud-architect/`, our `R46` = `binary-ninja-reverse/`.**
- Any upstream change to `routing.json`, `MASTER-ROUTING.md`, or `routing-benchmark.json` that mentions `R45` MUST be remapped (R45 → R46) before taking.
- Never accept upstream benchmark cases or route entries without checking the ID against our `skills/config/routing.json`.

## When to skip a sync entirely

- Upstream change only touches files we rewrote, and the change doesn't fix a bug we also have.
- Upstream release notes show nothing new in skill coverage or tooling.
- The diff is pure churn against the fork's language/model (e.g., Chinese prose edits).

When in doubt: take the *idea*, leave the diff.
