---
name: ctf-sandbox
description: Thin PRIMARY for CTF / AWD / lab multi-type orchestration. Hands off to the sidecar CTF-Sandbox-Orchestrator. Use when the user says CTF, AWD, 靶场, or competition challenge and no more specific pwn/APK/IDA route already won.
---

# CTF sandbox entry (sidecar, not a second router)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Run `../scripts/case-init.ps1` to create scope.md (case tracking + network profile). CTF/lab targets are authorized environments; use `-NetworkProfile lab` or `offline` for competitions.
2. `NOW`: Open the package root's `../../CTF-Sandbox-Orchestrator/ctf-sandbox-orchestrator/SKILL.md` and continue with its sandbox assumptions.
3. `MUST NOT` register the 40+ `competition-*` sub-skills into `routing.json`. This entry is a single PRIMARY latch.
4. `ACT`: Let the orchestrator pick a downstream `competition-*`. When the challenge type is already explicit (pwn/ROP, APK, IDA), an earlier routing.json rule should already have won — don't fight it.

## Why a separate layer

`CTF-Sandbox-Orchestrator/` is a **GPL sidecar package**; authorization is sandbox-internal by default. The core router package stays MIT + scope.md tracking. This skill is only a keyword entry — it does not merge the competition tree into the core.

## Completion self-check (MUST pass before claiming done)

- [ ] Did I run case-init / scope for tracking, instead of treating "user said CTF" as an open internet pass?
- [ ] Did I open the sidecar orchestrator instead of treating the 40 sub-skills as PRIMARY?
- [ ] If the task is actually pwn/APK/IDA, did I let the more specific PRIMARY take over?
