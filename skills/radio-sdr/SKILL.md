---
name: radio-sdr
description: Use for authorized RF/SDR security research including signal identification, replay feasibility study in shielded labs, and wireless protocol analysis outside classic Wi-Fi.
---

# RF / SDR Security Research

## ACTION REQUIRED (Execute immediately after reading)

1. `NOW`: **Spectrum use and transmission are strictly regulated by law**; only authorized bands / shielded rooms / lab targets
2. `NOW`: scope must state the equipment, frequency band, and whether transmission is permitted (receive-only by default)
3. `ACT`: Receive-only identification → demodulation analysis → lab replay feasibility assessment

## Scope

- Wireless remote controls / sensors and other non-Wi-Fi RF (authorized)
- Protocol research such as ADS-B / remote controls (legal reception)
- Division of labor with wifi-wireless: this skill is biased toward **general-purpose RF via SDR**; Wi-Fi attacks and defense go to R29

## Workflow

```text
□ Regulatory and licensing confirmation
□ Receive-only: identify center frequency and modulation
□ GNU Radio / URH analysis
□ Replay only in a shielded room with written permission
□ Conclusions focus on: whether unauthorized control is possible / hardening recommendations
```

## Toolchain

| Tool | Purpose |
|------|------|
| RTL-SDR / HackRF (compliant) | Transmit/receive hardware |
| URH / GNU Radio | Analysis |
| Inspectrum | Signal |

## References

- `references/sdr-lab-rules.md`
- `../wifi-wireless/` `../ot-ics/` `../hardware-security/`

## Routing Context

**Upstream**: MASTER R38  
**MUST NOT**: interfere with public communications, unauthorized transmission

## Task Completion Self-Check

- [ ] Receive-only by default and legal boundaries recorded?
- [ ] Checklist?
