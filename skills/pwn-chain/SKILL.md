---
name: pwn-chain
description: |
  End-to-end engineering methodology for going from reverse engineering to a usable Working Exploit.
  When it applies: you have the binary + the vulnerable point + the target environment, and you need to write an exploit that reliably works (not a script that only reproduces locally and crashes against the remote).
  Covers three main directions: stack overflow / heap exploitation / kernel pwn. Emphasizes the engineering gap between "works locally in CTF" and "reliably works on the real remote": libc version mismatch, heap spray timing, SMEP/SMAP/KASLR, stack alignment, remote buffering.
  Core toolchain: pwntools + GEF/pwndbg + ROPgadget/Ropper + one_gadget + libc-database + qemu-system kernel debugging.
  触发关键词（中文命中词，保留）：pwn、栈溢出、堆溢出、ROP、ret2libc、ret2csu、one_gadget、libc-database、堆利用、tcache、fastbin、unsorted bin、kernel pwn、kROP、SMEP、SMAP、KASLR、modprobe_path、pwntools、GEF、pwndbg。
  English trigger keywords: pwn, stack overflow, heap overflow, ROP, ret2libc, ret2csu, one_gadget, libc-database, heap exploitation, tcache, fastbin, unsorted bin, kernel pwn, kROP, SMEP, SMAP, KASLR, modprobe_path, pwntools, GEF, pwndbg.
---

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm that this skill's operations are authorized routine work
2. `NOW`: Confirm whether the current task falls within this skill's scope of application
3. `NEXT`: Read `../tool-index.md`, verify tool availability and real paths
4. `NEXT`: Call bootstrap when tools are missing, never guess paths
5. `ACT`: Enter step 1 of the "Workflow" and execute — do not stop at a confirmation state

# From Vulnerability Point to Working Exploit (Pwn Chain)

## Scope of Application

Use this skill when the task matches one of these scenarios:

1. **Binary in hand + known vulnerable point** — static/audit/fuzz already found the overflow/UAF/double free, and you need to go from trigger to shell
2. **CTF challenge works locally but not remotely** — remote environment differences break the script; it needs stabilization
3. **Binary exploitation of a real target** — SRC / red team scenarios where a memory-corruption vulnerability has been identified and RCE must be built
4. **ioctl bug in a Linux kernel driver** — triggered from userland, the goal is privilege escalation to root

**Prerequisite**: you already know "where it blows up". This skill is not responsible for discovering vulnerabilities (that is fuzzing / auditing); it only handles "writing the exploit from the vulnerable point".

### Division of labor with other skills

| Scenario | What to use |
|------|--------|
| Identifying custom VM / anti-debug / complex obfuscation | `reverse-engineering/` |
| Opening a binary from scratch for static analysis | `ida-reverse/` or `radare2/` |
| **Have the vulnerable point, write an exploit that pwns the remote** | **This skill** |
| Integrating the shell from pwn into a full attack chain | `attack-chain/` (downstream) |

`reverse-engineering/` focuses on "understanding what the program does" (pattern recognition, protocol recovery, solving the odd mechanisms in CTF challenges); this skill focuses on "turning an already-understood vulnerability into an executable attack". The two are often used together, but the division is clear.

## Core Workflow

```text
Step 1: Confirm the vulnerability type + protection mechanisms
   ├─ checksec ./vuln (NX / Canary / PIE / RELRO / Fortify)
   ├─ file ./vuln  + readelf -d ./vuln
   ├─ Vulnerability classification: stack overflow / format string / heap (UAF/DF/OF) / integer / race / kernel
   └─ → decide which references/ to follow

Step 2: Choose the exploitation strategy
   ├─ NX off + no ASLR → direct shellcode
   ├─ NX on + libc given → ret2libc / one_gadget
   ├─ NX on + no libc given → leak then look it up in libc-database
   ├─ Heap → technique per glibc version (tcache/fastbin/unsorted/large)
   └─ Kernel → commit_creds / modprobe_path / core_pattern

Step 3: Prepare libc + gadgets
   ├─ libc-database：./find puts 0x6f0
   ├─ ROPgadget --binary ./libc.so.6 --only "pop|ret"
   ├─ one_gadget ./libc.so.6
   └─ Compute base：leak_addr - libc.sym['puts']

Step 4: Write the pwntools template (local process)
   ├─ context.binary = ELF('./vuln')
   ├─ p = process('./vuln')  /  p = gdb.debug('./vuln','b *main+xx')
   ├─ payload = cyclic(N) + p64(ret) + ...
   └─ p.interactive()

Step 5: Make it work locally
   ├─ Repeatedly attach + watch registers + adjust offset
   ├─ Use pwndbg/GEF's vmmap / heap / bins / telescope
   └─ Once it works, switch to remote()

Step 6: Remote stabilization
   ├─ libc offsets: derive from the leak via libc-database, never guess
   ├─ Stack alignment: not 16-byte aligned → movaps crash → add a ret gadget
   ├─ Remote network latency → recvuntil with a precise anchor string, ban fuzzy sleep
   ├─ Remote buffering: sendlineafter is more reliable than sendline
   ├─ Heap spray success rate: scale up the spray count + leave padding chunks to prevent consolidation
   └─ Run it many times: write a while True loop and verify the success rate is ≥ 95%
```

## Typical Scenarios

### Scenario 1: remote 64-bit binary (NX+PIE+canary, libc given)

```text
Have: ./vuln (64-bit ELF, NX, PIE, canary) + ./libc.so.6 + nc host port
Vulnerability: read(buf, 0x200) but buf is only 0x40 bytes → stack overflow
Protections: canary blocks it, PIE randomizes .text

Strategy:
1. Leak the canary first (stack / format string / partial read)
2. Then leak one libc function address (puts@got)
3. Compute the libc base with libc.address = leaked - libc.sym['puts']
4. one_gadget ./libc.so.6 and pick a magic gadget whose constraints can be satisfied
5. payload = padding + canary + saved_rbp + (pop_rdi + bin_sh + system) or one_gadget directly
6. Add a ret gadget to fix the stack alignment (critical!)
```

Full templates live in `references/stack-pwn.md`.

### Scenario 2: Linux kernel driver ioctl out-of-bounds write → root

```text
Have: vmlinux + bzImage + initramfs.cpio.gz + a custom vuln.ko
Vulnerability: controllable copy_from_user length in ioctl(0x1337, ptr) → kernel heap overflow (kmalloc-64 slab)
Protections: SMEP, SMAP, KASLR, KPTI

Strategy:
1. Modify the init script to get a root shell (CTF) or first leak the KASLR base and continue (real-world)
2. Leak the kernel base via /proc/kallsyms (possibly restricted) or uninitialized heap spray
3. Spray tty_struct / msg_msg / pipe_buffer in the kmalloc-64 slab
4. Overwrite the vtable pointer to point to userland → no (SMEP), switch to stack pivot + kernel ROP
5. ROP chain: prepare_kernel_cred(0) → commit_creds → swapgs+iretq → userland execve("/bin/sh")
6. Or the easier way: overwrite modprobe_path to "/tmp/x", write a /tmp/x, then trigger modprobe
```

Full templates live in `references/kernel-pwn.md`.

## On-Demand Bootstrap

### Tool dependencies

| Tool | Purpose | Install method |
|------|------|---------|
| pwntools | exploit writing framework | `pip install pwntools` |
| GEF | gdb enhancement (recommended for kernel + userland) | `git clone https://github.com/bata24/gef` (actively maintained fork) |
| pwndbg | gdb enhancement (best heap debugging experience) | `git clone https://github.com/pwndbg/pwndbg && ./setup.sh` |
| ROPgadget | gadget search | `pip install ropgadget` |
| Ropper | gadget search (alternative, more architecture support) | `pip install ropper` |
| one_gadget | libc magic gadget finder | `gem install one_gadget` (needs ruby) |
| libc-database | libc fingerprint lookup | `git clone https://github.com/niklasb/libc-database && ./get` |
| qemu-system-x86_64 | kernel challenge debugging | `apt install qemu-system-x86` |
| binwalk / cpio | initramfs unpacking | `apt install binwalk cpio` |
| patchelf | switching libc versions | `apt install patchelf` |

### Bootstrap check script

```bash
# One-shot check + install of core tools
for t in pwntools ropgadget ropper; do
  pip show $t >/dev/null 2>&1 || pip install $t
done

command -v one_gadget >/dev/null || gem install one_gadget

[ -d ~/tools/libc-database ] || git clone https://github.com/niklasb/libc-database ~/tools/libc-database
[ -d ~/tools/libc-database/db ] || (cd ~/tools/libc-database && ./get ubuntu debian)

[ -d ~/tools/pwndbg ] || (git clone https://github.com/pwndbg/pwndbg ~/tools/pwndbg && cd ~/tools/pwndbg && ./setup.sh)
```

### After the same tool fails auto-install 2 times

Stop retrying and output structured manual install steps (pip source / gem source / git mirror / apt source) for the user to confirm.

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master control), `routing.md`
**Trigger condition**: have a binary + an identified vulnerable point, need to write an exploit

**Upstream skills (use them first, then come back to this skill)**:
- Still don't understand what the binary does → `reverse-engineering/`
- Need detailed static analysis → `ida-reverse/`
- Quick recon to confirm architecture/protection mechanisms → `radare2/`

**Downstream skills (after getting the shell)**:
- Integrate into a full attack chain (lateral movement, privilege escalation, persistence) → `attack-chain/`

**Sub-module navigation**:
- Stack exploitation (ret2libc / ret2csu / one_gadget / stack alignment) → `references/stack-pwn.md`
- Heap exploitation (tcache / fastbin / unsorted / large bin / FILE struct) → `references/heap-pwn.md`
- Kernel pwn (kROP / SMEP-SMAP bypass / KASLR leak / modprobe_path) → `references/kernel-pwn.md`

## Notes

- **Do not call it done just because it works locally** — the local libc / ASLR / network environment all differ from the remote; you must run it in remote mode 20+ consecutive times to verify stability
- **The libc version must be confirmed** — derive it from a leak + libc-database, never assume it is the default Ubuntu 22.04 libc
- **Stack alignment is a common 64-bit pitfall** — `movaps xmm0, [rsp]` faults when rsp is not 16-byte aligned; add an empty `ret` gadget to fix it
- **Heap exploitation is extremely sensitive to the glibc version** — tcache was introduced in 2.27, safe-linking in 2.32, hooks removed in 2.34; every version has a different exploitation path
- **Kernel pwn must confirm the cpu flags first** — whether the qemu launch parameters include +smep +smap +pku directly decides how the ROP chain is written
- **One KASLR leak is enough** — once you have one kernel address, every other address is an offset from it; do not keep leaking

## Task Completion Self-Check (MUST pass before claiming done)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the Checklist items required by RULES?
