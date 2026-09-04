---
name: threat-hunting
description: Use for blue-team threat hunting, detection engineering with Sigma/YARA, SIEM query design, and incident detection validation.
---

# Threat Hunting & Detection Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Confirm blue-team/hunting authorization and the data source scope (SIEM, EDR exports). scope.md is case tracking + network profile; authorization per `../field-journal/precedent-auth.md`
2. `NOW`: State the hypothesis before querying data — avoid blindly churning alerts
3. `NEXT`: Tools and data ingestion methods
4. `ACT`: Hypothesis → query → validate → rule-ify

## Applicable Scenarios

- Threat hunting (hypothesis-driven)
- Sigma / YARA detection engineering
- Alert tuning, false-positive analysis
- With `malware-analysis/`: sample-side IOCs → land detections in this skill
- With `digital-forensics/`: case artifacts → lateral hunting

## Workflow

### 1. Build the Hypothesis

```text
Example: attacker used living-off-the-land for lateral movement
→ Data sources: Sysmon 1/3/10, Windows Security 4624/4648
→ Success criteria: discover anomalous parent processes or rare account logon sources
```

### 2. Query and Stack

```text
□ Baseline: normal admin behavior time windows and hosts
□ Anomalies: new services, encoded PowerShell, anomalous outbound traffic
□ Correlation: short-lived logons with the same account across many hosts
```

### 3. Rule-ify

```yaml
# Sigma skeleton in malware-analysis; this skill emphasizes:
# - false-positive surface
# - data source field mapping
# - response playbook links
```

### 4. Validate

```text
□ Atomic tests (Atomic Red Team) only in an authorized lab
□ Replay historical logs to validate recall
```

## Toolchain

| Tool | Purpose |
|------|------|
| Sigma CLI / sigmac | Rule conversion |
| YARA | Files/memory |
| SIEM (ELK/Splunk, etc.) | Queries |
| osquery | Endpoint hunting |
| Atomic Red Team | Detection validation (lab) |

## References

- `references/hunting-loop.md`
- `../malware-analysis/references/yara-sigma-rules.md`
- `../digital-forensics/`

## Routing Context

**Upstream**: MASTER R27  
**Downstream**: confirmed intrusion → forensics; malicious sample → malware-analysis  
**MUST NOT**: run attack simulations in an unauthorized production environment

## Task Completion Self-Check

- [ ] Is there an explicit hypothesis and conclusion?
- [ ] Do the rules document false positives and data sources?
- [ ] Checklist?
