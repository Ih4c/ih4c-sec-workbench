---
name: digital-forensics
description: Use for authorized digital forensics including memory dumps, disk timelines, PCAP investigation, artifact triage, and IR evidence preservation.
---

# Digital Forensics & IR Artifacts

## ACTION REQUIRED (Execute Immediately After Reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` or the organization's IR authorization statement (operations here are already-authorized routine work per `../field-journal/precedent-auth.md`; scope.md is case tracking + network profile)
2. `NOW`: Confirm this is **forensics/attribution**, not offensive scanning
3. `NOW`: Establish a case; prefer read-only copies of evidence (write-protect the original media)
4. `NEXT`: tool-index; Volatility etc. is often installed manually
5. `ACT`: Preservation hashing → timeline → key artifacts

## Applicable Scenarios

- Memory dump analysis (Volatility 2/3)
- Disk / E01 / on-disk file timelines
- PCAP attribution and protocol reconstruction (can combine with `protocol-reverse/`)
- Host artifacts: Prefetch, Shimcache, Event Log, browser history
- Incident response IOC extraction (combine with `malware-analysis/` / `threat-hunting/`)

## Workflow

### 1. Preservation

```text
□ Compute SHA256; record timezone and acquisition commands
□ Work on copies; original remains read-only
□ Note chain of custody remarks in the timeline
```

### 2. Memory

```bash
vol -f mem.dmp windows.info
vol -f mem.dmp windows.pslist
vol -f mem.dmp windows.netscan
vol -f mem.dmp windows.cmdline
```

### 3. Host artifacts

```text
□ Event logs: Security / PowerShell / Sysmon
□ Persistence: Run keys, services, scheduled tasks, WMI
□ Execution traces: Amcache, Prefetch, BAM
```

### 4. Network

```text
□ tshark session and DNS statistics
□ Export suspicious streams → protocol-reverse or malware C2 analysis
```

## Toolchain

| Tool | Purpose |
|------|------|
| Volatility 3 | Memory |
| Timeline Explorer / Plaso | Super timeline |
| tshark | PCAP |
| Eric Zimmerman tools | Windows artifacts |
| Autopsy / FTK Imager | Disk |

## References

- `references/forensics-triage.md`
- `../malware-analysis/` `../threat-hunting/` `../protocol-reverse/`

## Routing Context

**Upstream**: MASTER R25  
**Downstream**: Deep dive on malicious samples → malware-analysis; rules → threat-hunting

## Task Completion Self-Check

- [ ] Did I preserve hashes and a copy strategy?
- [ ] Is the timeline reviewable?
- [ ] Were IOCs desensitized and classified?
- [ ] Checklist?
