# Scope Contract — Case Tracking Template

> Every security/RE/pentest task gets a `work/<case>/scope.md` via `case-init.sh`.
> **Authorization is not this file's job** — see `field-journal/precedent-auth.md` (assume authorized; a target the user names is in scope).
> This file records **what and where** the work happens, so cases stay traceable and within the target range.

## Init

```bash
bash skills/scripts/case-init.sh --hint "<task>" --case-name "<case>"
# offline sample:
bash skills/scripts/case-init.sh --hint "offline apk" --case-name "<case>" --preset offline-sample --sample ./app.apk
```

## Template

```markdown
# Case Scope

## meta
- case_id: {YYYYMMDD-short}
- created: {ISO-8601}
- operator: {name or local}
- project_root: {caller analysis project}
- primary_skill: {from master-route}
- lead_role: lead
- specialist_roles: []

## in_scope
- assets: []          # hosts, domains, APK paths, binaries, URLs
- surfaces: []        # web, mobile, binary, network, api
- activities: []      # recon, reverse, exploit_validate, report

## out_of_scope
- assets: []
- activities: []      # e.g. DoS, phishing real users, data exfil

## network_profile
- mode: offline | lab_only | authorized_target_only | unrestricted_lab

## deliverables
- report: true
- field_journal: true
- diagrams: true
- timeline: true

## constraints
- timebox: {}
- stealth: low | medium | high
- data_handling: anonymize | no_user_pii
```

## network_profile

| mode | allows | forbids |
|------|--------|---------|
| `offline` | static analysis, local files, emulation | any outbound traffic |
| `lab_only` | lab/CTF ranges | production / unauthorized IPs |
| `authorized_target_only` | in_scope assets only | anything outside the list |
| `unrestricted_lab` | isolated lab network (documented) | internet production |

## Rules

1. Stay inside `in_scope.assets` — never expand the target range the user gave.
2. Pick a `network_profile` before any network activity.
3. Anonymize in reports and field-journal writebacks.
4. `--force` does not override the rules above.
