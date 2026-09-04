# EDR Hook Survey Quick Reference

> Authorized red teams / adversary emulation / own-product testing only; using this against unauthorized targets is forbidden.

This document summarizes the userland and kernel monitoring points of mainstream EDR/AV products, so the red-team reconnaissance phase can quickly determine "what needs to be handled".

## 1. Mainstream EDR Fingerprints and Hook Patterns at a Glance

| Vendor / Product | Userland components | Kernel drivers | Main monitoring surface |
|------------|-----------|---------|-----------|
| CrowdStrike Falcon | `CSFalconService.exe`, `CSAgent.sys` injected into target processes | `CSAgent.sys`, `CSBoot.sys` | heavy kernel callbacks + ETW-TI; fewer userland hooks (cloud detections) |
| Microsoft Defender for Endpoint (MDE) | `MsMpEng.exe`, `MpClient.dll` | `WdFilter.sys`, `WdBoot.sys`, `WdNisDrv.sys` | comprehensive: AMSI + ETW-TI + ntdll inline hooks + kernel callbacks |
| SentinelOne | `SentinelAgent.exe`, `SentinelHelperService.exe` | `SentinelMonitor.sys`, `SentinelDeviceControl.sys` | heavy ntdll userland hooks + kernel callbacks + its own ETW provider |
| Elastic Defend (formerly Endpoint Security) | `elastic-endpoint.exe` | `elastic-endpoint-driver.sys` | mainly ETW + a few ntdll hooks, uploading via the Elastic Agent |
| ESET | `ekrn.exe`, `eamsi.dll` | `eamonm.sys`, `epfwwfp.sys` | very many userland hooks (NtCreateFile / NtOpenProcess, etc.) |
| Sophos Intercept X | `SophosFileScanner.exe`, `SophosNtpService.exe` | `SophosED.sys`, `hmpalert.sys` | ntdll hooks + HMPA memory protection + kernel callbacks |
| Kaspersky | `avp.exe`, `klif.sys` | `klif.sys`, `klhk.sys` | heavy userland hooks + KLIF proprietary minifilter + network filter driver |
| Trend Micro Apex One | `TmListen.exe`, `TmCCSF.dll` | `tmcomm.sys`, `tmactmon.sys` | userland hooks + behavior monitoring driver |
| Carbon Black | `RepMgr.exe`, `RepWAV.exe` | `ParityDriver.sys` | leans kernel callbacks + ETW |

### Quick fingerprint script

```powershell
$edrSigs = @{
    'CSAgent'           = 'CrowdStrike Falcon'
    'SentinelAgent'     = 'SentinelOne'
    'elastic-endpoint'  = 'Elastic Defend'
    'ekrn'              = 'ESET'
    'MsMpEng'           = 'Microsoft Defender'
    'SophosFileScanner' = 'Sophos Intercept X'
    'avp'               = 'Kaspersky'
    'TmListen'          = 'Trend Micro Apex One'
    'cb'                = 'Carbon Black'
}

Get-Process | ForEach-Object {
    foreach ($k in $edrSigs.Keys) {
        if ($_.ProcessName -match $k) {
            "[+] $($edrSigs[$k]) detected: $($_.ProcessName) (PID $($_.Id))"
        }
    }
}

Get-ChildItem 'C:\Windows\System32\drivers\*.sys' |
    Where-Object { $_.Name -match 'CSAgent|Sentinel|elastic|eam|WdFilter|Sophos|klif|tmcomm|Parity' } |
    Select-Object Name, VersionInfo
```

## 2. Key Userland ntdll Hook Functions

`ntdll.dll` exports that an EDR is almost certain to hook (grouped by ATT&CK behavior):

| Function | Behavior monitored | ATT&CK |
|------|-----------|--------|
| `NtCreateThreadEx` | remote thread injection, QueueUserAPC injection | T1055.002 / T1055.004 |
| `NtAllocateVirtualMemory` | shellcode allocating RWX memory | T1055 |
| `NtAllocateVirtualMemoryEx` | cross-process memory allocation (Win10+ new API) | T1055 |
| `NtProtectVirtualMemory` | changing page permissions RW→RX | T1055 |
| `NtWriteVirtualMemory` | writing shellcode into another process | T1055.012 |
| `NtMapViewOfSection` | section-based injection (Process Doppelganging / Ghosting) | T1055.013 |
| `NtCreateSection` | paired with MapViewOfSection | T1055.013 |
| `NtOpenProcess` | opening the target process to obtain a handle | T1057 |
| `NtQueueApcThread` / `NtQueueApcThreadEx` | APC injection | T1055.004 |
| `NtCreateProcess` / `NtCreateProcessEx` / `NtCreateUserProcess` | creating child processes (including PPID spoofing) | T1106 |
| `NtSetContextThread` | changing the thread context (thread hijacking injection) | T1055.003 |
| `NtResumeThread` | resuming the thread after injection | T1055 |
| `NtQuerySystemInformation` | enumerating processes / drivers / handles | T1057 / T1082 |
| `NtAdjustPrivilegesToken` | privilege escalation to SeDebugPrivilege etc. | T1134 |
| `NtLoadDriver` | loading a kernel driver (BYOVD) | T1543.003 |

### Verifying whether hooks exist

```powershell
# Simple: disassembly-diff the on-disk ntdll against the current process's ntdll
# 1. grab the on-disk ntdll
copy C:\Windows\System32\ntdll.dll C:\temp\ntdll_clean.dll

# 2. In windbg, attach to any process and export the .text section of the live ntdll
# .writemem c:\temp\ntdll_live.bin ntdll!.text L?<size>

# 3. Disassemble NtAllocateVirtualMemory in IDA / radare2; the normal form is:
#    mov r10, rcx
#    mov eax, <SSN>
#    test byte ptr [...]
#    jne ...
#    syscall
#    ret
# If the first instruction is instead jmp <some address>, it is hooked
```

## 3. Kernel Callback Monitor Points

Common kernel callbacks EDR products register (all can be unregistered via the BYOVD route in `attack-chain`, but at high cost):

| API | Callback timing registered | Defensive purpose |
|-----|--------------|-----------|
| `PsSetCreateProcessNotifyRoutineEx` | process creation / exit | intercepting suspicious child processes |
| `PsSetCreateThreadNotifyRoutine` | thread creation / exit | detecting remote thread injection |
| `PsSetLoadImageNotifyRoutine` | DLL / EXE loaded into any process | module integrity / unsigned-blocking |
| `CmRegisterCallback` / `CmRegisterCallbackEx` | registry operations | persistence detection |
| `ObRegisterCallbacks` | `OpenProcess` / `OpenThread` handle requests | preventing LSASS handle acquisition (T1003.001) |
| `MmRegisterPhysicalMemoryCallback` | physical memory mapping | anti-DMA / memory forensics |
| `IoRegisterFsRegistrationChange` | filesystem registration | minifilter coordination |
| `KeRegisterNmiCallback` | NMI (few EDRs use it) | anomaly monitoring |
| `EtwRegister` (kernel side) | kernel ETW reporting | symbiotic with ETW-TI |

### Enumerating registered callbacks with windbg

```text
0: kd> dx -r1 nt!PspCreateProcessNotifyRoutine
0: kd> dx -r1 nt!PspCreateThreadNotifyRoutine
0: kd> dx -r1 nt!PspLoadImageNotifyRoutine

0: kd> !object \Callback
0: kd> !object \Callback\ProcessObject
```

Or use tools such as PChunter / DRVHV — normal users can view the callback list through a GUI.

## 4. Statically Dumping the Hook Table (IDA + windbg flow)

### Workflow A: single-process comparison

```text
1. Find a process that the EDR has injected its userland component into (any live process)
2. windbg attach (-pn target.exe)
3. lm m ntdll  → get the module base
4. .writemem c:\temp\ntdll_live.bin ntdll+0x0 L?<image size>
5. Copy C:\Windows\System32\ntdll.dll to c:\temp\ntdll_disk.dll
6. Load both files in IDA and jump to NtAllocateVirtualMemory:
     - disk: standard prologue
     - live: first instruction is jmp <0x7FFE000000xx>
7. Follow the jmp target address → that is the EDR's trampoline; dump it
8. Look inside the trampoline to see which DLL it finally lands in, and confirm the EDR module name
```

### Workflow B: batch hook-table generation

Use `HookHunter` or a self-written script:

```powershell
# pseudo workflow; see the scripts referenced in the references section
$disk = Get-Content C:\Windows\System32\ntdll.dll -Encoding Byte
$live = # obtain via OpenProcess + ReadProcessMemory
# compare the first 16 bytes of every export in the .text section
```

## 5. pe-sieve Automatic Detection

`pe-sieve` is the first choice for reconnoitering EDR hooks and implant self-checks:

```powershell
# Basic scan
pe-sieve64.exe /pid 1234

# Recommended combination (includes shellcode and hook detection)
pe-sieve64.exe /pid 1234 /shellc 3 /modules 3 /imp 3 /data 3 /dir hooks_dump

# Key parameters:
#   /shellc N    shellcode scan level (0-3)
#   /modules N   module integrity checks (0-3)
#   /imp N       IAT hook checks
#   /data N      data-section scans
#   /dir <path>  dump output directory
```

The output produces `*.tag` files under `hooks_dump/<pid>.<name>/`, listing hook addresses:

```text
modified_modules.tag example:
71f10000;ntdll.dll
71f1a3b0;hook;jmp_far
71f1c020;hook;jmp_near
```

These can be fed straight into IDA to jump to the corresponding RVA for further analysis.

### Embedding pe-sieve in an implant (self-check)

In real engagements the `pe-sieve` is often compiled as a library (`libpe-sieve`) so the implant self-checks at startup: if ntdll is hooked, it triggers the unhook flow; conversely, finding yourself hooked can be a warning sign — you may be inside a sandbox.

## 6. API Monitor v2 Dynamic Observation

API Monitor v2 (Rohitab) is suitable for observing in the lab when and where the EDR inserts hooks:

```text
1. Start API Monitor v2 (as administrator)
2. In API Filter, check:
     - NT Native API → Memory Management
     - NT Native API → Process and Thread
     - Windows Defender / AMSI (if visible)
3. Monitor New Process → select the implant test sample
4. Observe:
     - the NtAllocateVirtualMemory call order
     - whether the call is relayed through an EDR DLL
5. In the Modules tab, see which EDR DLLs were injected via LoadLibrary
```

## 7. Common EDR DLLs (userland) at a Glance

| DLL | Vendor | Notes |
|-----|------|------|
| `umppc*.dll` | Microsoft Defender | MpClient userland |
| `mpoav.dll` | Microsoft Defender | AMSI provider |
| `aswAMSI.dll` | Avast | AMSI provider |
| `eamsi.dll` | ESET | AMSI provider |
| `IDPMServiceClient.dll` | Sophos | HMPA injection |
| `klsihk64.dll` | Kaspersky | injected into target processes |
| `CrowdStrike.Sensor.dll` | CrowdStrike | older versions; new versions rely mainly on the kernel |
| `SentinelInjection64.dll` | SentinelOne | userland injection |
| `TmUmEvt64.dll` | Trend Micro | behavior monitoring |

After confirming the target EDR, decide which DLL to reverse to extract the hook table.

## Reference Links

- pe-sieve: <https://github.com/hasherezade/pe-sieve>
- HollowsHunter: <https://github.com/hasherezade/hollows_hunter>
- API Monitor v2: <http://www.rohitab.com/apimonitor>
- MITRE ATT&CK T1562: <https://attack.mitre.org/techniques/T1562/>
- MITRE ATT&CK T1055: <https://attack.mitre.org/techniques/T1055/>
- ired.team EDR notes: <https://www.ired.team/offensive-security/defense-evasion>

## Routing Callback

After finishing the hook survey, return to `SKILL.md` Step 3 to choose the bypass technique combination, then execute per `references/unhook-techniques.md` and `references/telemetry-blinding.md`.
