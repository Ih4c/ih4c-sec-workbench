---
name: ot-ics
description: Use for authorized OT/ICS security assessment covering Purdue model zoning, PLC/SCADA exposure, industrial protocol discovery, and safe passive-first evaluation.
---

# OT / ICS Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` — **OT missteps can cause physical harm**
2. `NOW`: The scope must spell out: site, network segments, whether active scanning / register writes are allowed
3. `NOW`: case-init → scope.md (tracking + network profile); default **passive-first**; no PLC writes before the case is ready
4. `NEXT`: tool-index; most OT tools need manual install and an isolated lab network
5. `ACT`: asset & zone identification → exposure → read-only verification

## Use cases

- ICS/SCADA/DCS security assessment (authorized)
- Purdue model zoning and cross-zone channels
- Modbus/DNP3/S7/EtherNet-IP protocol exposure
- Engineering stations, HMIs, historians, jump hosts
- IT/OT convergence boundary (firewall rules, data diodes)

## Safety rules (MUST)

```text
MUST NOT without explicit permission:
- write coils/registers to PLCs
- high-rate scans across production OT
- disrupt Safety Instrumented System (SIS) paths
Prefer: read-only identification, traffic mirroring, offline firmware/config analysis
```

## Workflow

### Phase 1 — Zones & assets

```text
□ Purdue L0–L5 sketch: field devices → control → supervisory → site DMZ → enterprise
□ Asset inventory: PLC/RTU/HMI/engineering stations/historians/jump hosts
□ Protocol & port baseline (authorized segments only)
```

### Phase 2 — Passive & read-only

```text
□ SPAN/mirror PCAP → protocol-reverse / Wireshark ICS dissectors
□ Offline audit of configs and engineering files (TIA/RSLogix exports etc.)
□ Default credentials & plaintext protocols (unauthenticated Modbus) → record as Findings; never write to disks or change values
```

### Phase 3 — Limited active (authorized only)

```text
□ Low-rate identification, inside maintenance windows
□ Read-only function codes first
□ Evidence for every step; on anomaly, stop immediately and report
```

### Phase 4 — Firmware / patch surface

```text
□ Controller firmware version → CVE mapping (never blind-flash firmware)
□ Join firmware-pentest for offline image analysis
```

## Toolchain

| Tool | Purpose | Notes |
|------|---------|-------|
| Wireshark ICS dissectors | passive parsing | mirrored traffic |
| Nmap NSE (restricted) | identification | rate & time windows |
| Claroty/Nozomi etc. | asset discovery | commercial/on-site |
| PLC vendor engineering software | config audit | offline first |
| binwalk / Ghidra | firmware | offline |

## References

- `references/ot-safe-assessment.md`
- `../firmware-pentest/` `../protocol-reverse/` `../network` via pentest-tools

## Routing context

**Upstream**: MASTER R28
**Downstream**: firmware deep-dive `firmware-pentest`; protocols `protocol-reverse`; IT lateral `windows-ad`/`attack-chain`
**Siblings**: never run default web-scan parameters against OT

## Completion self-check

- [ ] Did I default to passive/read-only and record the authorization boundary?
- [ ] Did I avoid writes to control loops (unless explicitly allowed)?
- [ ] Do Findings include physical/process impact statements?
- [ ] Checklist / journal?
