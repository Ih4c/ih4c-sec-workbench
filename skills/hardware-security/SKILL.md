---
name: hardware-security
description: Use for authorized hardware and embedded interface security research including UART/JTAG discovery, debug pad triage, secure boot overview, and offline firmware extraction support.
---

# Hardware / Embedded Interface Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Confirm **physical contact authorization** and device ownership. scope.md is case tracking + network profile; authorization is per `../field-journal/precedent-auth.md`.
2. `NOW`: ESD / power safety; read-only probing by default
3. `NEXT`: Coordinate with firmware-pentest for image analysis
4. `ACT`: Shell and debug interface identification → consoles → extraction

## Applicable Scenarios

- UART / JTAG / SWD debug port discovery
- Boot logs, root shells, boot interruption
- Coordinating with teardown for Flash extraction
- Feasibility assessment of secure boot / encrypted Flash (non-destructive first)

## Workflow

```text
□ Tear down the authorized device; photograph and mark test points
□ Multimeter to find GND/VCC/TX/RX; logic levels 1.8/3.3/5V
□ USB-TTL read-only logging; record the baud rate
□ JTAG: enumerate IDCODE; assess whether locked
□ Extract images → hand off to firmware-pentest / ghidra
```

## Toolchain

| Tool | Purpose |
|------|------|
| USB-TTL / logic analyzer | UART |
| J-Link / CMSIS-DAP | Debugging |
| bus pirate / flipper (lab) | Multi-protocol |
| binwalk / flashrom | Extraction |

## References

- `references/debug-interface-triage.md`
- `../firmware-pentest/` `../ot-ics/`

## Routing Context

**Upstream**: MASTER R34
**MUST NOT**: Unauthorized teardown / damaging others' devices

## Task Completion Self-Check

- [ ] Interface levels and pinout recorded?
- [ ] Images hashed for integrity?
- [ ] Checklist?
