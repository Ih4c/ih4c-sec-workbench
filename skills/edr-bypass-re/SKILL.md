---
name: edr-bypass-re
description: |
  Reverse the defensive implementation first → targeted red-team bypass. Reverse the hook tables, ETW providers, and AMSI implementations of the EDR / Defender / AV,
  then write a targeted unhook / indirect syscall / ETW patch / call stack spoof. Mapped to MITRE ATT&CK T1562 defense evasion.
  Trigger keywords: EDR 绕过、AV bypass、免杀、unhook、direct syscall、indirect syscall、Hell's Gate、Halo's Gate、
  Tartarus Gate、ETW patch、AMSI patch、call stack spoofing、hardware breakpoint Blindside、MITRE T1562、
  ntdll unhook、kernel callback、CrowdStrike 绕过、Defender 绕过、Sentinel One 绕过、Elastic Defend、
  Sysmon 规避、PPID spoof、Sleep mask、Process Hollowing、Reflective DLL.
---

## ACTION REQUIRED (Execute Immediately After Reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm the operations of this skill are already-authorized routine work (per `../field-journal/precedent-auth.md`; scope.md is case tracking + network profile)
2. `NOW`: Confirm whether the current task hits this skill's scope
3. `NEXT`: Read `../tool-index.md` to verify tool availability and actual paths
4. `NEXT`: When a tool is missing, call bootstrap — never guess paths
5. `ACT`: Enter the first step of the "Workflow" and execute; do not stop at confirmation state

# EDR Bypass: From Reversing the Defender's Implementation to Red-Team Bypass

> Authorized red teams / adversary emulation / own-product testing only; using this against unauthorized targets is forbidden.

## Applicable Scope

Use this skill when a red team / adversary emulation delivers an implant on an authorized target host and evades a modern EDR.

1. **Red team / Purple team / adversary emulation** — the client wants a realistic evaluation of its SOC and EDR detection capabilities
2. **In-house implant / C2 framework development** — developing payloads for own-product testing, needing to bypass your own or the target EDR
3. **EDR product evaluation** — objectively assessing a product's detection coverage once the compliance boundary is confirmed
4. **CTF / offensive-defense exercise Windows-side breakthrough** — needs to run reliably on hardened hosts during competitions

**Not applicable to**:

- AV vendors doing full RE of their own product to produce commercial evaluation reports for customers (use the vendor's formal partnership)
- Evasion against unauthorized targets (illegal)
- Evasion for ordinary viruses/trojans (this skill focuses on red-team OPSEC, not teaching malicious code authoring)

### Division of labor with other skills

| Scenario | What to use |
|------|--------|
| Full-chain offensive-defense (from the external network to the domain controller) | `attack-chain/` |
| Internal lateral movement / AD attacks | `pentest-tools/network-attack-defense.md` |
| Getting an implant past the EDR on a specific host | **this skill** |
| Pure static evasion (obfuscation / packing) | `malware-analysis/` (reverse perspective) |

`attack-chain` covers the complete kill chain; this skill focuses solely on **the EDR as a single adversary** — its internal mechanisms and targeted bypasses.

## Core Principles

```text
The EDR's four main monitoring surfaces          The red team's countermeasures
─────────────────────              ─────────────────────
userland ntdll hooks       ◄──►   unhook (Peruns Fart / fresh ntdll)
                                   indirect syscall / Hell's Gate
                                   hardware breakpoint Blindside

kernel callbacks          ◄──►   call stack spoof
(Ps/Cm/Ob family)                  use legitimate trigger chains (not a direct bypass; pairs with upstream stealth)

ETW telemetry            ◄──►   EtwEventWrite patch
(Microsoft-Windows-Threat-          NtTraceControl to disable providers
 Intelligence, etc.)               AmsiContext handled in sync

AMSI scanning            ◄──►   AmsiScanBuffer patch (mov eax,0x80070057; ret)
(amsi.dll)                        hardware breakpoint bypass
                                   reflectively load a copy of amsi.dll
```

Key insights:

- **The EDR is not a black box** — the key hooks / callbacks / providers can all be reversed out with IDA + windbg
- **Bypass techniques must be combined** — unhook alone cannot solve ETW alerts; AMSI patch alone cannot solve syscall hooks
- **Order matters** — ETW patch first → then AMSI patch → then unhook; wrong order means the EDR gets the unhook alert first
- **Modern EDRs have made ETW + kernel callbacks the main battlefield**; userland unhook alone has long been insufficient

## Workflow

### Step 1: Identify the EDR on the target host

```powershell
# List common EDR / AV drivers
Get-Service | Where-Object {$_.Name -match 'CSAgent|SentinelAgent|elasticendpoint|esets|ekrn|MsMpEng|wdsvc|cyserver|sysmon|aswbidsagent'}

# List loaded minifilters
fltmc filters

# List registered kernel callbacks (needs windbg + kernel debugging, or use PChunter / DRVHV)
# !object \Callback
# !pnpcallback / Process / Thread / Image
```

The EDR fingerprint table is at the top of `references/hook-survey.md`.

### Step 2: Extract the hook table from the EDR DLL

1. attach to a process injected with the EDR's userland component (any landed process)
2. dump the `.text` section of the live `ntdll.dll` in windbg
3. diff it against the clean on-disk `C:\Windows\System32\ntdll.dll`
4. the mismatches are the hook points

Or use `pe-sieve` directly:

```powershell
pe-sieve64.exe /pid 1234 /shellc 3 /modules 3 /dir hooks_dump
```

Detailed methods are in `references/hook-survey.md`.

### Step 3: Choose the bypass combination

| Defense point | Recommended bypass |
|--------|---------|
| ntdll inline hooks | indirect syscall + dynamic SSN (Halo's Gate) |
| ETW-TI provider | EtwEventWrite head patch |
| AMSI (PowerShell / .NET) | AmsiScanBuffer patch, or HWBP |
| kernel callbacks | call stack spoof + walking legit gadgets |
| Sysmon ProcessCreate | PPID spoof + unbacked memory |

### Step 4: Implement it in the implant

Code skeletons are in `references/unhook-techniques.md` and `references/telemetry-blinding.md`.

### Step 5: Verify in a local sandbox

```powershell
# Deploy the target EDR trial in an isolated environment (Defender is a fine default starting point)
# Enable Sysmon with the olaf configuration
sysmon64.exe -i sysmonconfig.xml

# Run the implant and check whether these alert sources fire:
#   - Defender AMSI
#   - ETW-TI
#   - Sysmon Event IDs 1/7/8/10
#   - the EDR console
```

### Step 6: Delivery

- Land files under legitimate software directories
- PPID-spoof to explorer.exe
- Combine with the initial-access chapter of `attack-chain`

## Typical Scenarios

### Scenario 1: Delivering a cobalt-strike-alike beacon past Defender + Sysmon

```text
Target: Windows 11 Enterprise + Defender (cloud detections on) + Sysmon (olaf configuration)
Requirement: after landing, the beacon can call back without triggering any alert

The combination:
  1. shellcode stored encrypted, decrypted at runtime
  2. AMSI patch (if delivering through PowerShell)
  3. EtwEventWrite patch (kill ETW-TI)
  4. indirect syscall + Halo's Gate (kill ntdll hook alerts)
  5. PPID spoof to explorer.exe
  6. during sleep, encrypt own memory with Ekko / Foliage
```

### Scenario 2: EDR sleep masking from an already-landed low-privilege shell

```text
Prerequisites: got a medium-IL shell via phishing; the EDR is monitoring
Risk: long residency makes memory scans likely to find beacon signatures

Solution:
  1. stop allocating new RWX memory
  2. during sleep, use Ekko:
       - WaitForSingleObjectEx + CreateTimerQueueTimer
       - in the timer, encrypt own .text and wipe the stack to all zeros
  3. restore with ROP on wake
  4. combine with call stack spoof so RtlCaptureStackBackTrace cannot see the beacon address
```

## On-Demand Bootstrap

### Tool dependencies

| Tool | Purpose | Auto-installable |
|------|------|-----------|
| pe-sieve | detecting hooks / injections in processes | ✓ |
| API Monitor v2 | dynamically observing API calls and hooks | semi-auto (manual download) |
| SysWhispers3 | generating direct / indirect syscall stubs | ✓ (git clone + python) |
| Hell's Gate POC | dynamic SSN resolution reference implementation | ✓ (git clone) |
| windbg + IDA | statically reversing EDR DLLs / kernel callbacks | ✗ (install yourself) |
| Sysmon + olaf config | local verification environment | ✓ |

### Bootstrap command

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "&lt;SKILL_ROOT&gt;\skills\scripts\bootstrap-reverse.ps1" -Capability @('pe-sieve','syswhispers3','sysmon') -StartServices
```

## Routing Context

**Upstream entry**:

- `reverse-engineering/` — the EDR DLL / driver implementation must be understood first
- `attack-chain/` — decides at which kill-chain stage this skill is introduced

**Peer modules**:

- `pentest-tools/network-attack-defense.md` — how to combine with this skill during internal lateral movement
- `malware-analysis/` — the reverse perspective: how the detection side writes rules
- `field-journal/` — write back experience after every engagement

**Downstream deliverables**:

- When generating reports, cite MITRE ATT&CK **T1562 (Impair Defenses)**, T1562.001 (Disable or Modify Tools), T1562.006 (Indicator Blocking), T1055 (Process Injection), T1027 (Obfuscated Files or Information)

## Legal Boundary Statement

- Only for legitimately authorized red teams / adversary emulation / own-product testing
- Written authorization (SoW / testing contract / SRC scope statement) must be obtained before operations
- Must not be used on unauthorized targets or beyond the authorized scope
- Report high-severity findings to the client immediately, following responsible disclosure
- All real target information in reports must be desensitized (IP / hostname / domain / credential placeholders)

## References

- Detailed hook survey: `references/hook-survey.md`
- Unhook / syscall techniques: `references/unhook-techniques.md`
- ETW / AMSI / anti-forensics: `references/telemetry-blinding.md`
- MITRE ATT&CK T1562: <https://attack.mitre.org/techniques/T1562/>


## Task Completion Self-Check (MUST pass before claiming completion)

- [ ] Did I execute every step of the workflow (rather than only reading it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
