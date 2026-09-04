---
name: wifi-wireless
description: Use for authorized wireless security assessment including Wi-Fi capture, WPA handshake analysis, rogue AP detection research, and lab-only deauth testing.
---

# Wi-Fi / Wireless Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read precedent-pentest; **wireless attacks carry high legal risk** — written authorization and a physical scope are mandatory
2. `NOW`: scope.md records the target SSID/BSSID/site; scanning neighbor networks is forbidden
3. `NEXT`: Confirm the adapter's monitor-mode capability
4. `ACT`: Recon → capture → analyze (lab first)

## Applicable Scenarios

- Authorized Wi-Fi security assessment
- WPA/WPA2 handshake capture and offline assessment
- Rogue AP / rogue hotspot detection research
- Enterprise wireless isolation and captive portal security

## Workflow

```text
□ iwconfig / airmon-ng into monitor mode (legal environment)
□ airodump-ng to lock onto the target BSSID channel
□ Handshake or PMKID capture (target only)
□ hashcat/aircrack offline passphrase policy assessment
□ Report: encryption type, isolation, portal bypass, recommendations
```

## Toolchain

| Tool | Purpose |
|------|------|
| aircrack-ng suite | Capture/assessment |
| hcxdumptool / hcxtools | PMKID |
| hashcat | Passphrase assessment |
| Wireshark | Management frame analysis |

## References

- `references/wireless-lab-rules.md`
- `../pentest-tools/` `../attack-chain/` (proximity section)

## Routing Context

**Upstream**: MASTER R29  
**MUST NOT**: unauthorized deauth, operations against non-target client networks

## Task Completion Self-Check

- [ ] Was the target BSSID strictly locked?
- [ ] Does the report include hardening recommendations?
- [ ] Checklist?
