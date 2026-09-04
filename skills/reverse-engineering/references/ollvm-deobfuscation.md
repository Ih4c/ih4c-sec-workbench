# OLLVM Deobfuscation (Obfuscator-LLVM)

> An OLLVM deobfuscation workflow for APK .so files, ELF binaries, and control-flow-flattening scenarios.
> Tool and variant information is based on a 2026 survey of active community projects, not training memory.
> Applies to: Android NDK hardening, CTF reversing, packed .so analysis, commercial obfuscator countermeasures.

---

## 0. Quick Decision: Which Tool Should I Use?

Match your environment and the obfuscation type you suspect, and pick directly:

| Your Situation | First Choice | Alternative | Notes |
|---------|---------|------|------|
| IDA Pro 7.5-7.7 + Hex-Rays, want one-click deflattening | **obpo-plugin** | d810-ng | obpo works at microcode + dataflow + concolic level, strongest results, but it is a cloud plugin (needs network; the core is closed-source) |
| IDA Pro (any recent version), want local one-stop deobfuscation | **d810-ng** | original D-810 | Local, open source, integrated Z3, supports many OLLVM/Tigress/Hodur/Approov variants |
| Binary Ninja | **ollvm-breaker** | — | Practical against hardened Android .so samples (libvdog, etc.) |
| No IDA/BN, pure script, x86/x64 target | **ollvm-unflattener** (Miasm) | angr deflat | Miasm-based symbolic execution, BFS multi-layer processing |
| No IDA/BN, pure script, x86/x64 target | **ollvm-unflattener** (Miasm) | angr deflat | Miasm-based symbolic execution, BFS multi-layer processing |
| Pure Python symbolic execution, CTF scenarios | **angr** Deobfuscator | Triton | No GUI dependency, scriptable |
| ARM64 .so target, no IDA | **deollvm** (Unicorn) | angr | Unicorn-based ARM64 deflat |
| Encountered BR obfuscation (indirect branches) | **DeObfBR** | Set the data segment read-only | Goron/Arkari-style BR obfuscation can be countered simply by making the data segment read-only |
| Encountered Tigress obfuscation | d810-ng `UnflattenerSwitchCase`/`UnflattenerTigressIndirect` | — | d810-ng has built-in Tigress-specific unflatteners |

> **Core advice:** prefer **d810-ng** (local, actively maintained, wide variant coverage). When the cloud service is usable, **obpo-plugin** gives the best results. Only if both fail should you go to **angr/Miasm** symbolic execution for customized handling.

---

## 1. The Modern OLLVM Variant Ecosystem (2026 Community Survey)

OLLVM is long past being just the original 2017 repository. Below are the currently active obfuscator branches — **before deobfuscating you must first determine which variant the target is**, because the countermeasures differ substantially between variants:

### 1.1 Obfuscator Branch Lineage

| Variant | Baseline LLVM | New features vs. original OLLVM | Countermeasure focus |
|------|----------|----------------------|---------|
| **Obfuscator** (original) | 3.3~4.0 | sub + bcf + fla (the three basic passes) | Standard tools can all handle it |
| **Hikari** | 6~8 | Anti Class Dump, Function Call Obfuscate, Function Wrapper, Indirect Branching, Split BB, String Encryption | Must first decrypt strings + fix indirect jumps |
| **Hikari-LLVM15** | 15~19 | + Anti Debugging, Anti Hook, Constant Encryption | Now closed-source; Constant Encryption increases the difficulty of static analysis |
| **goron** | 7~10 | Indirect Branch/Call/GlobalVariable | ⚠️ Goron-style indirect obfuscation can be countered simply by "setting the data segment read-only" |
| **Arkari** (komimoe/Hikari) | 14~latest | Based on goron, continuously maintained | Same as goron; setting the data segment read-only partially counters it |
| **Pluto** | 14 | MBA Obfuscation, Random CF, Split BB, **Trap Angr** (specifically targets angr) | ⚠️ the Trap Angr pass breaks angr symbolic execution; switch tools or dodge the traps |
| **Polaris** (formerly Pluto) | 16 | Alias Access, Indirect Branch/Call, String Encryption, Merge Function, Linear MBA, Dirty Bytes Insertion, Function Splitting, Junk Insertion | A blend of Hikari + Pluto, the trickiest; needs layered handling |
| **O-MVLL** | open-obfuscator | Python-driven pass manager; Anti Hooking, Arithmetic (MBA), BB Duplicate, CF Breaking, Function Outline, Indirect Branch/Call, Opaque Constants | Common in modern Android hardening; Python config is easy to customize |
| **amice** (Rust) | Rust implementation | Full suite + VM Flatten, Instruction Virtualization, Delayed Offset Loading, Parameter Aggregation | Includes VMization; needs VM handler recovery, not plain deflat |
| **VMP family** (SmallVmp/VMPilot/xVMP/VMPacker) | — | Instruction virtualization | **Outside OLLVM scope**, needs VM reversing — see VM-specific tools |

### 1.2 Key Identification Clues

- **Trap Angr** (Pluto/Polaris): if angr explodes or path-explodes while running, suspect the target uses the Trap Angr pass → switch to d810-ng or a Unicorn dynamic approach
- **Goron/Arkari indirect jumps**: if the dispatcher uses an indirect jump (BR x8 rather than switch), first try setting the relevant data segment read-only — indirect jump targets often become statically solvable
- **Constant Encryption** (Hikari-LLVM15/Polaris/O-MVLL): constants are decrypted at runtime; pure static analysis cannot see real values → you need Unicorn to dynamically execute the decryption stub
- **VM Flatten** (amice): control flow becomes a VM dispatch loop — **do not treat it as ordinary fla**, you first need to identify the VM handler table

---

## 2. OLLVM Obfuscation Type Detection

Identification signatures of OLLVM's three core passes:

### 2.1 Control Flow Flattening (`fla`)

**Signatures in the IDA view:**
- The function entry first jumps to a single dispatcher block
- The main logic is split into multiple basic blocks, each jumping back to the dispatcher at its end
- The dispatcher decides the next block to execute via a **state variable**
- A huge `switch` structure whose cases have no logical relationship to each other

```
Original:             OLLVM flattened:
  block_A               entry -> dispatcher
  block_B                 ↓
  block_C              state_machine:
                         switch(state):
                           0 → block_A
                           1 → block_B
                           2 → block_C
```

**Variant shapes (the multiple dispatcher types d810-ng recognizes):**
- O-LLVM: switch / if-chain + state variable
- Tigress: `m_jtbl` (switch-case) or `m_ijmp` (indirect jump; requires `goto_table_info` config)
- Hodur (PlugX): nested `while(1)` state machine, `jnz state, #CONST`, **no switch dispatcher**
- Approov: `while(v8 != C)`, state constants concentrated in `0xF6000–0xF6FFF`

### 2.2 Bogus Control Flow (`bcf`)

- **Unreachable fake branches** are inserted between every real branch
- The fake branches are protected by **opaque predicates** (the condition is always true/false, but static analysis cannot prove it directly)
- Large amounts of dead code inflate the function size

```c
// Classic opaque predicate: x(x+1) is always even, which the compiler cannot prove
if (x * (x + 1) % 2 == 0) {
    // Real logic
} else {
    // Unreachable junk code
}
```

### 2.3 Instruction Substitution (`sub`) → MBA

- Simple arithmetic/bitwise operations are replaced with equivalent complex expressions (MBA, Mixed Boolean-Arithmetic)

```
a + b  →  (a ^ b) + 2*(a & b)
a ^ b  →  (a | b) - (a & b)
a - b  →  a + (~b) + 1
```

### 2.4 Quick Classification Table

| Obfuscation type | IDA signature | Primary countermeasure |
|---------|---------|------------|
| fla (flattening) | giant switch + dispatcher | obpo / d810-ng / deflat |
| bcf (bogus control flow) | unreachable branches + dead code | d810-ng opaque predicate removal / symbolic execution |
| sub/MBA | complex arithmetic expressions | d810-ng MBA simplifier / SiMBA (Z3) |
| fla + bcf + sub | all of the above, extremely inflated | **layered deobfuscation (bcf first, then fla, then sub)** |

---

## 3. Mainstream Tools in Detail (Active Community Projects)

### 3.1 obpo-plugin — Strongest Results, Cloud Plugin

> [obpo-project/obpo-plugin](https://github.com/obpo-project/obpo-plugin) · 629 stars · active 2026-06

A pseudocode optimizer built on Hex-Rays **microcode**, using **data-flow tracking + program slicing + concolic execution** to rebuild flattened control flow. Community consensus ranks its results among the strongest.

**Key features:**
- Operates at the microcode level, directly optimizing decompiler output (not rewriting ASM)
- Supports IDA 7.5.0 / 7.6.0 / 7.7.0 + Hex-Rays
- Architectures: ARM, ARM64, x86, x86_64, PowerPC, PowerPC64, MIPS (7.6/7.5)
- **Cloud plugin**: the target function's binary is uploaded to the obpo-server for processing (core closed-source; plugin free and open source)
- Server maintained at the author's own expense; 600s timeout; **no multithreading/malicious calls**

**Installation and usage:**
```text
1. Download obpo_plugin.py and the obpoplugin directory
2. Copy them into the IDA plugins path
3. Restart IDA and open the target binary
4. Locate the dispatcher block in the CFG, which usually looks like this:
   [See repo assets/dispatchblock.png for a screenshot]
5. Right-click → OBPO → Mark and process function
6. Refresh the decompiler after processing completes
7. Keep marking new dispatcher blocks based on the decompiler changes (iterate over nested fla)
```

**Use cases and limitations:**
- ✅ Standard and nested fla work well
- ⚠️ Requires network; be careful with sensitive samples (unpublished internal vulnerabilities, trade secrets) — the binary gets uploaded
- ⚠️ The server may be down; depends on the author's maintenance
- ❌ Cannot solve every obfuscation (explicitly stated by the author)

### 3.2 d810-ng — The Local One-Stop First Choice

> [w00tzenheimer/d810-ng](https://github.com/w00tzenheimer/d810-ng) · 223 stars · updated 2026-06-26

The modern maintained/refactored version (Next Generation) of D-810. Runs locally, open source, integrates the **Z3 SMT** solver, widest variant coverage.

**Core capabilities (organized from the d810-ng README):**

*Instruction-level optimizations:*
| Category | Description |
|------|------|
| MBA simplification | `(a+b)-2*(a&b) => a^b`, Z3-verified DSL rules |
| Hacker's Delight | bitwise equivalences (from the Hacker's Delight book) |
| O-LLVM patterns | MBA patterns specific to Obfuscator-LLVM |
| Constant folding | 22 constant-folding rules |
| Predicate simplification | opaque predicate removal (setz/setnz/lnot/smod) |
| Z3 rules | use SMT solving when template matching fails |
| Hodur-specific | MBA patterns of the PlugX (Hodur) malware |

*Control-flow Unflatteners (grouped by target obfuscation):*
| Unflattener | Target | Description |
|------------|------|------|
| `Unflattener` | O-LLVM | standard switch/if-chain + state variable |
| `UnflattenerSwitchCase` | Tigress | Tigress switch-case dispatch (`m_jtbl`) |
| `UnflattenerTigressIndirect` | Tigress | Tigress indirect jumps (`m_ijmp`); requires `goto_table_info` config |
| `HodurUnflattener` | Hodur (PlugX) | nested `while(1)` + `jnz state, #CONST`, no switch |
| `BadWhileLoop` | Approov | `while(v8 != C)`, state constants in 0xF6000–0xF6FFF |
| `UnflattenerFakeJump` | generic | removes always-true/always-false conditional jumps |
| `SingleIterationLoopUnflattener` | leftovers | cleans up single-iteration loops where `INIT == CHECK` and `UPDATE != CHECK` |
| `UnflattenControlFlowRule` (experimental) | generic | CFG unflattener based on path emulation |

**Installation and usage:**
```text
1. clone d810-ng
2. Install dependencies (including Z3)
3. Copy to the IDA plugins directory
4. Press Ctrl-Shift-D in IDA to load the plugin
5. Tick the rule sets to apply in the GUI
6. Apply to the target function
```

**Why d810-ng over the original D-810:**
- The original D-810 is barely maintained anymore
- d810-ng has CI tests, refactored code, and new Tigress/Hodur/Approov-specific unflatteners
- Z3 integrated — falls back to SMT solving when template matching fails, giving a higher success rate

### 3.3 ollvm-unflattener — Miasm Symbolic Execution, Pure Script

> [cdong1012/ollvm-unflattener](https://github.com/cdong1012/ollvm-unflattener) · 265 stars · active 2026-06

Built on the **Miasm** symbolic-execution engine, with no IDA/BN dependency — a pure Python command-line tool.

**Features:**
- Uses Miasm symbolic execution to recover the original control flow (unlike MODeflattener's pure-static method)
- **BFS multi-layer processing**: automatically follows the target function's callees and deobfuscates recursively
- Supports Windows/Linux x86/x64
- Outputs a deobfuscated binary

**Installation and usage:**
```bash
git clone https://github.com/cdong1012/ollvm-unflattener.git
cd ollvm-unflattener
pip install -r requirements.txt   # miasm, graphviz, keystone-engine

# Basic usage
python unflattener -i <input.bin> -o <output.bin> -t <function_addr> -a
# -a: automatically follow callees for multi-layer processing
```

**Best for:** no IDA, x86/x64 targets, batch scripted processing.

### 3.4 ollvm-breaker — Binary Ninja in Practice

> [amimo/ollvm-breaker](https://github.com/amimo/ollvm-breaker) · 441 stars

Uses **Binary Ninja** for deflattening; the repo ships the hardened Android sample `libvdog.so` as a test case and has fixed functions such as JNI_OnLoad, crazy::GetPackageName, and prevent_attach_one.

**Best for:** Binary Ninja users, hands-on Android .so work.

### 3.5 deollvm — ARM64 Unicorn

> [GeT1t/deollvm](https://github.com/GeT1t/deollvm) · 34 stars · 2026-04

A **Unicorn**-based ARM64 OLLVM deflat. An alternative for processing ARM64 .so when IDA is unavailable.

### 3.6 DeObfBR — Dedicated to BR Obfuscation

> [Mrack/DeObfBR](https://github.com/Mrack/DeObfBR) · 96 stars · 2026-06-25

Specializes in removing **BR obfuscation** (indirect-branch obfuscation, Goron/Arkari style).

**⚠️ Easy countermeasure (from awesome-ollvm):** Goron/Arkari-style indirect-related obfuscation can be countered simply by **making the data segment read-only** — indirect jump targets often rely on a runtime-writable data segment; once it is read-only they become statically solvable.

### 3.7 angr — The General Symbolic-Execution Framework

```python
import angr

proj = angr.Project("target.so", auto_load_libs=False)
cfg = proj.analyses.CFGFast()
func = proj.kb.functions[0x12345]

# Built-in Deobfuscator
deob = proj.analyses.Deobfuscator(func=func)
deob.normalize()
```

**⚠️ Pluto/Polaris's Trap Angr pass:** these two variants wrote dedicated traps to break angr symbolic execution. If angr path-explodes or errors out, suspect the target uses Trap Angr → switch to d810-ng or a Unicorn dynamic approach.

---

## 4. Complete Deobfuscation Workflow (by Scenario)

### 4.1 General Decision Tree

```
Target binary
  ↓
1. Identify the OLLVM variant (see the clues in Section 1.2)
  ├── Original OLLVM / Hikari / O-MVLL  → standard fla/bcf/sub
  ├── Pluto / Polaris                   → watch for Trap Angr; avoid angr
  ├── Goron / Arkari                    → first try data-segment read-only, then handle BR
  ├── Tigress                           → d810-ng Tigress unflattener
  ├── Hodur (PlugX)                     → d810-ng HodurUnflattener
  └── amice (contains VM)               → not plain fla; needs VM handler recovery
  ↓
2. Choose a tool (see the Section 0 decision table)
  ├── IDA + network available + non-sensitive sample → obpo-plugin
  ├── IDA + local only              → d810-ng
  ├── Binary Ninja                 → ollvm-breaker
  ├── no GUI + x86/x64             → ollvm-unflattener (Miasm)
  ├── no GUI + ARM64               → deollvm (Unicorn) / angr
  └── pure symbolic execution / CTF → angr
  ↓
3. Layered deobfuscation (order matters)
  a) first remove opaque predicates (bcf)   → d810-ng opaque predicate removal
  b) then remove control flow flattening (fla) → unflattener
  c) finally simplify MBA (sub)             → d810-ng MBA simplifier / SiMBA
  ↓
4. Verify
  ├── Did the function size shrink significantly?
  ├── Did the CFG change from star/radial to chain/tree shape?
  └── Do Frida hooks of key functions confirm the logic is correct?
```

### 4.2 Android NDK .so Deobfuscation

Android NDK .so hardened with OLLVM is the most common APK reverse-engineering scenario.

**Step 1 — extract the .so:**
```bash
adb pull /data/app/~~/lib/arm64/libnative.so
# Or unzip it straight from the APK: unzip target.apk -d out/ ; find out -name "*.so"
```

**Step 2 — identify OLLVM and the variant:**
```bash
readelf -a libnative.so | grep -E "Size|text"   # .text abnormally large but few functions → likely OLLVM
# Open in IDA and check the function characteristics:
#   giant switch → fla
#   unreachable branches → bcf
#   complex arithmetic → sub/MBA
#   indirect jump BR x8 → Goron/Arkari, try data-segment read-only
#   while(1) + jnz state → Hodur, use the d810-ng HodurUnflattener
```

**Step 3 — deobfuscate (layered):**
```
a) bcf: d810-ng opaque predicate removal  (or obpo handles it automatically)
b) fla: d810-ng Unflattener / obpo-plugin / deollvm(ARM64)
c) sub: d810-ng MBA simplifier
```

**Step 4 — dynamic verification with Frida:**
```javascript
// Trace the OLLVM state variable; helps deflat locate the state variable address
const target = Module.findBaseAddress("libnative.so");
console.log("[+] libnative.so @", target);

// Hook the dispatcher entry to observe the sequence of state changes
Interceptor.attach(target.add(0x1234), {  // dispatcher offset
    onEnter(args) {
        // Read the state variable (register/stack location comes from decompilation)
        console.log("[state]", this.context.x8);  // assuming state is in x8
    }
});
```

### 4.3 Quick CTF Deobfuscation

CTF is usually time-pressed, so take the fastest route:

```python
#!/usr/bin/env python3
"""CTF OLLVM quick deflat with angr"""
import angr

proj = angr.Project("challenge", auto_load_libs=False)
cfg = proj.analyses.CFGFast()

# Take the largest few functions (the most likely obfuscated ones)
funcs = sorted(cfg.functions.values(), key=lambda f: f.size, reverse=True)[:5]
for func in funcs:
    print(f"[*] {func.name} @ {hex(func.addr)} size={hex(func.size)}")
    try:
        deob = proj.analyses.Deobfuscator(func=func)
        deob.normalize()
        print(f"    [+] deobfuscated")
    except Exception as e:
        print(f"    [-] failed: {e}")
        # angr failed → suspect Trap Angr → switch to d810-ng / Unicorn
```

---

## 5. MBA Expression Simplification

### 5.1 Common OLLVM MBA Patterns

```python
# These identities are the simplification targets for expressions generated by the OLLVM sub pass
"(a | b) + (a & b)"        # → a + b
"(a | b) - (a & b)"        # → a ^ b
"(a ^ b) + 2*(a & b)"      # → a + b
"(a | b) & ~(a & b)"       # → a ^ b
"~(~a & ~b)"               # → a | b (De Morgan)
```

### 5.2 Tool Selection

| Tool | Method | Best for |
|------|------|------|
| **d810-ng MBA simplifier** | in-IDA batch, Z3-verified | First choice; integrated into the decompilation flow |
| **SiMBA** (`pip install simba-simplifier`) | CLI/library | pure expression simplification, batch processing |
| **Arybo** | symbolic bit-vectors | large volumes of MBA expressions |
| **Z3 direct solving** | SMT | the most general; when template matching all fails |

```python
# SiMBA example
from simba import simplify_mba
exprs = ["(a | b) + (a & b)", "(a ^ b) + 2*(a & b)"]
for e in exprs:
    print(f"{e}  →  {simplify_mba(e)}")
```

---

## 6. Complete Deobfuscation Pipeline Script

```bash
#!/bin/bash
# OLLVM deobfuscation pipeline (2026 community tools)
# For ELF/.so hardened with standard OLLVM / Hikari / O-MVLL

BINARY=$1

echo "[*] Stage 0: basic analysis and variant identification"
file $BINARY
readelf -h $BINARY 2>/dev/null | head -5
echo "    → confirm the variant in IDA (see Section 1)"

echo "[*] Stage 1: local deobfuscation with d810-ng (first choice)"
echo "    IDA → Ctrl-Shift-D to load d810-ng"
echo "    tick: MBA + Opaque predicate + Unflattener"
echo "    Apply to target functions"
echo "    save the IDB"

echo "[*] Stage 2: obpo-plugin (if d810-ng is not enough and network is available)"
echo "    IDA → right-click the dispatcher → OBPO → Mark and process"
echo "    ⚠️ do not use on sensitive samples (the binary is uploaded to a cloud service)"

echo "[*] Stage 3: no-IDA alternative (x86/x64)"
echo "    python unflattener -i $BINARY -o deobf.bin -t <func_addr> -a"

echo "[*] Stage 4: no-IDA alternative for ARM64 .so"
echo "    deollvm (Unicorn) or the angr Deobfuscator"

echo "[+] Done. Re-analyze and verify in IDA."
```

---

## 7. Common Pitfalls (Community Field Notes)

| Problem | Cause | Solution |
|------|------|---------|
| angr path explosion / abnormal exit | Pluto/Polaris's **Trap Angr** pass | switch to d810-ng or a Unicorn dynamic approach |
| Cannot connect to obpo-plugin | the server is maintained at the author's own expense and may be down | fall back to local d810-ng; can file an issue in the obpo repo |
| Goron/Arkari indirect-jump deflat fails | the dispatcher uses BR x8 rather than switch | set the data segment read-only first, then use DeObfBR |
| Function still messy after d810-ng | OLLVM customized pass parameters/seed | symbolic-execute the opaque predicates away first, then unflatten |
| Nested fla (multi-layer flattening) not fully cleaned in one pass | obpo/d810-ng clear only one layer per pass | **iterate**: mark each newly appearing dispatcher |
| deflat errors out on ARM64 .so | old deflat scripts support only x86 | use d810-ng / obpo (ARM64-capable) / deollvm |
| Hikari strings are invisible | String Encryption pass | emulate the decryption stub with Unicorn and dump the decrypted strings |
| deflat completely ineffective on amice targets | contains VM Flatten / Instruction Virtualization | **not OLLVM fla**; needs VM handler recovery (see VM reversing) |
| Hodur (PlugX) samples have no switch dispatcher | nested while(1) + jnz state | use the d810-ng **HodurUnflattener**, not the generic Unflattener |
| Approov state constants show no pattern | constants concentrated in 0xF6000–0xF6FFF | use the d810-ng **BadWhileLoop** unflattener |
| obpo used on a sensitive sample by mistake | the binary is uploaded to a cloud service | classified / unpublished-vulnerability samples: **local tools only** (d810-ng/angr) |
| Frida hook on an OLLVM function hangs | state variable modified, causing an infinite loop | put a conditional breakpoint at the dispatcher entry to limit the execution count |

---

## 8. Tool Quick-Reference Table (2026 Community Activity)

| Tool | Platform | Method | Stars/Price | Last update | Open source | Notes |
|------|------|------|---------|---------|------|------|
| **obpo-plugin** | IDA | microcode+concolic (cloud) | 629 | 2026-06 | plugin open/core closed | strongest results; needs network |
| **ollvm-breaker** | Binary Ninja | BN API | 441 | 2026-06 | ✅ | hands-on Android .so |
| **ollvm-unflattener** | CLI | Miasm symbolic execution | 265 | 2026-06 | ✅ | x86/x64, BFS multi-layer |
| **d810-ng** | IDA | microcode+Z3 | 223 | 2026-06 | ✅ | **local first choice**, wide variant coverage |
| **DeObfBR** | — | BR-obfuscation specialist | 96 | 2026-06 | ✅ | Goron/Arkari indirect branches |
| **IDA_Ollvm-unflattener** | IDA | Miasm plugin version | 90 | 2026-04 | ✅ | IDA plugin wrapper of ollvm-unflattener |
| **deollvm** | CLI | Unicorn | 34 | 2026-04 | ✅ | ARM64 specialist |
| **angr** | CLI | symbolic execution | — | active | ✅ | general-purpose; countered by Trap Angr |
| **SiMBA** | CLI/library | MBA simplification | — | — | ✅ | expression simplification |
| **Triton** | CLI | symbolic execution + taint | — | active | ✅ | dynamic symbolic execution |

---

## 9. Reference Links

**Obfuscators (to understand the countermeasure target):**
- [obfuscator-llvm/obfuscator](https://github.com/obfuscator-llvm/obfuscator) — the original OLLVM
- [HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari) — Hikari
- [komimoe/Hikari](https://github.com/komimoe/Hikari) — Arkari (based on goron, LLVM 14+)
- [amimo/goron](https://github.com/amimo/goron) — goron
- [bluesadi/Pluto](https://github.com/bluesadi/Pluto) — Pluto
- [za233/Polaris-Obfuscator](https://github.com/za233/Polaris-Obfuscator) — Polaris (formerly Pluto)
- [open-obfuscator/o-mvll](https://github.com/open-obfuscator/o-mvll) — O-MVLL
- [fuqiuluo/amice](https://github.com/fuqiuluo/amice) — OLLVM passes implemented in Rust
- [lich4/awesome-ollvm](https://github.com/lich4/awesome-ollvm) — **variant ecosystem overview (strongly recommended to read first)**

**Deobfuscation tools:**
- [obpo-project/obpo-plugin](https://github.com/obpo-project/obpo-plugin) — the strongest cloud plugin
- [w00tzenheimer/d810-ng](https://github.com/w00tzenheimer/d810-ng) — the local first choice
- [cdong1012/ollvm-unflattener](https://github.com/cdong1012/ollvm-unflattener) — Miasm pure script
- [amimo/ollvm-breaker](https://github.com/amimo/ollvm-breaker) — Binary Ninja
- [GeT1t/deollvm](https://github.com/GeT1t/deollvm) — ARM64 Unicorn
- [Mrack/DeObfBR](https://github.com/Mrack/DeObfBR) — BR-obfuscation specialist
- [maskelihileci/IDA_Ollvm-unflattener](https://github.com/maskelihileci/IDA_Ollvm-unflattener) — IDA plugin version
- [angr](https://angr.io/) — symbolic-execution framework
- [SiMBA](https://github.com/tech-srl/simba) — MBA simplification

**Academic/blog:**
- [Quarkslab: Deobfuscation: Recovering an OLLVM-protected program](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html) — the classic deflat theory
- [MODeflattener](https://github.com/mrT4ntr4/MODeflattener) — static deflat (the comparison baseline for ollvm-unflattener)

> Related documents: [[anti-analysis.md]] (anti-debug/anti-analysis master table), [[tools-advanced.md]] (advanced toolset), [[elf-analysis.md]] (ELF file analysis), [[ai-assisted-re.md]] (AI-assisted reverse engineering)
