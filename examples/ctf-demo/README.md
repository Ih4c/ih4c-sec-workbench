# examples/ctf-demo — Complete Workflow Example

> This directory demonstrates the standard reverse-skill workflow: **routing → authorization gate → timeline → evidence chain → report**.
> The content is a fictional example (CTF lab), shown only to illustrate how the workflow works.

## Workflow Demonstration

```text
1. User task: "analyze this CTF pwn challenge, stack overflow gets"
2. Routing: master-route.ps1 -Hint "CTF pwn stack overflow" → PRIMARY R17 (pwn-chain)
3. Authorization: case-init.ps1 -Hint ... -CaseName ctf-demo -AuthGranted → scope.md
4. Execution: timeline append + evidence E-001/E-002 + workitems updates
5. Output: report (docs-generator) + anonymized field-journal write-back
```

## Files

| File | Description |
|------|------|
| `scope.md` | Case scope (auth granted / targets / network_profile) |
| `timeline.md` | Append-only timeline |
| `workitems.md` | Work items and coverage |
| `evidence/` | Example evidence records (E-001 repro command, E-002 crash output) |
| `report/` | Example final report structure |

## Real-World Usage

```powershell
# Initialize a real case (authorized target)
powershell -NoProfile -ExecutionPolicy Bypass -File skills/scripts/case-init.ps1 `
  -Hint "your task" -CaseName my-case -AuthGranted -TargetUrl "https://target/" `
  -NetworkProfile authorized_target_only

# Append evidence
powershell -File skills/scripts/append-evidence.ps1 -CaseRoot work\my-case `
  -Id E-001 -Title "..." -ReproCommand "..."
```

> Note: real cases should live under `work/<case>/` (gitignored, to prevent leaks); this example directory is kept in git for reference.
