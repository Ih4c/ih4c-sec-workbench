# RE Agent Workflow Gates (Static ↔ Dynamic)

> Inspiration: binary-re phase divisions, community RE skills (Frida/r2/Ghidra/IDA loops), the Cerberus three-head ring (static/dynamic/instrumentation)  
> Issue #65 additions: the IAT repair iron rule, six-phase mapping, .NET/DLL·SYS equivalent paths; user-instruction feasibility gate; bypass patches 6–10; the anti-debug/obfuscation cookbook A–T; the non-PE multi-format cookbook U–AV (2026-08-12)  
> Applies to: `reverse-engineering/`, `ida-reverse/`, `radare2/`, `malware-analysis/`, and handoffs with the cre role

## 0. Startup

```text
□ scope.md: offline sample path or authorized device/target
□ tool-index: actual paths for file/strings/r2/ida/frida, etc.
□ Role: cre (ops/role-map)
```

## 0.1 Transition handoff (decision delta)

Between phases, do not re-inject the full case context. `scope.md` / `workitems.md` / Evidence remain authoritative; `timeline.md` carries only the transition delta:

1. At the end of a stage/turn, write only the `decision_delta` that actually changes later actions; write `[]` when nothing changed.
2. Unchanged route/auth/scope/network profile/tool state/hypothesis go only in `carry_forward_refs`; the consumer reads them by reference, without re-serializing / re-emitting.
3. `decision_delta` is not full state; the consumer must first inherit the refs, then apply the delta.
4. Stop at a next-step menu only when two or more evidence-supported branches lead to different next actions; deterministic gates advance directly.

Example: when Triage is complete and the only legal next step is Static, the transition only needs `decision_delta: [phase=triage->static]` + `carry_forward_refs: [scope.md, evidence/E-triage.md]`.

## 0.5 User-Instruction Feasibility Gate (Issue #65)

**Principle**: obey the user's **goal**, not blindly the user's **step order**. Before skipping a step you must state the prerequisite and ask for confirmation; forced steps after confirmation MUST be done, with Evidence quality honestly annotated.

| Situation | Agent MUST |
|------|------------|
| User wants X, and the current state can yield **valid** Evidence | Do X, update the Evidence |
| User wants X, but a **known blocking prerequisite** exists (e.g., the sample is already determined packed and the static IAT is unreadable) | **Forbidden** to pretend a meaningful IAT check was done; ① state the blocker in one sentence; ② give the recommended order (unpack/repair the IAT first, or go straight to dynamic API capture); ③ **ask the user to confirm** "still force the current garbage table" or "follow the recommended order" |
| User explicitly **forces** the current step (e.g., read the IAT even without unpacking) | Execute it and record Evidence, MUST annotate `quality=unreadable` / `packed` (or equivalent); **forbidden** to draw conclusions like "no network capability" from that |
| User accepts the recommended order | Do the prerequisite step first; then do X automatically or on request; **forbidden** to pass the prerequisite step (e.g., unpacking) **off as** "import table check completed" |

**Relationship with "redo X"**: redoing X still means redoing the named step (or its confirmed, negotiated legal prerequisite), and it is forbidden to swap in an unrelated step. Unpacking is a **prerequisite** of the import table, not a **substitute** for it.

Typical conflict: on a packed sample the user says "don't unpack yet, look at the import table first" → the packer usually corrupts the import directory / encrypts descriptors, so the static table is garbage and meaningless → follow this table's "blocking prerequisite" row; you may neither silently unpack and pass it off nor silently hand over the garbage table as done.

## 1. Triage (5–15 minutes · mandatory start point)

```text
□ Compute the sample Hash (MD5/SHA256) → unique ID
□ Identify the file type: EXE / DLL / SYS / ELF / Mach-O / .NET / script(bat/ps1/vba) / JS / APK, etc.
□ Non-PE/script/APK/driver special cases: see §3.4 and `references/nonpe-format-cookbook.md` (U–AV)
□ file / DIE / entropy / packer signatures (PEiD / DIE / Exeinfo, etc.)
□ Architecture: x86 / x64 / ARM; compiler-language clues (VC++ / Delphi / .NET / Go / Rust)
□ Packer-type clues: UPX / ASPack / VMProtect / Themida / unknown obfuscation
□ strings / rabin2 -z sweep
□ MUST have the import/export anchors (see "Import Table Hard Gate and Equivalent Paths" below); if the user jumps the gun and the sample is packed → go through §0.5 first
□ Output: E-triage (MUST contain the imports or an equivalent-anchor classification summary, with quality annotations where applicable) + a hypothesis list
```

**Phase gate (Triage → Static/Dynamic)**: before E-triage records the imports **or** a legal equivalent-anchor summary, MUST NOT enter Dynamic (unless an IAT repair failure is already recorded and the dynamic bypass is chosen, see §1.2), and MUST NOT claim "basic triage complete". On parse failure you MUST still write the failure output into Evidence — no skipping. When the user asks to "redo the import table check", MUST redo the imports/equivalent step itself (or first complete the prerequisite negotiated per §0.5) — swapping in other analysis steps as a substitute is forbidden.

### 1.1 Import Table Hard Gate and Equivalent Paths

| Sample type | MUST anchor (Evidence) | Notes |
|----------|----------------------|------|
| Native PE/ELF/Mach-O (readable IAT) | `E-imports` / `E-triage-imports`: import classification summary | `rabin2 -i` / IDA imports / equivalent |
| DLL / SYS / shared libraries | **Both** `E-imports` + `E-exports` (`rabin2 -i` + `rabin2 -E`) | The export table has the same priority as the import table (external entry surface) |
| .NET managed (no traditional IAT) | **Equivalent path**: dnSpy/IL/metadata/assembly references and sensitive-API summary → still written into the `E-imports` or `E-triage-imports` semantic slot | **Forbidden** to slip through the hard gate by saying "no IAT"; dnSpy inspection = the native "checking the import table" |
| Import table parse failure / empty / packed garbage table | Still record the failure or the garbage-table output as Evidence, and mark `quality` | No silent skipping; a garbage table must not support capability-negative conclusions |

**Overly-clean import table warning (MUST flag)**: if the import table is "too clean" (only kernel32/ntdll and other base DLLs, almost no business APIs), strongly suspect dynamic loading via `LoadLibrary` + `GetProcAddress` → note the suspicion in Evidence and **SHOULD** move to Dynamic to capture in-memory APIs; do not claim "no network/file capability" from the static IAT alone.

**High-risk API combinations (Patch 8 · SHOULD)**: when the import table is very long, prefer outputting **malicious-combination clusters** over raw listings, filtering pure system-base calls. Examples (non-exhaustive):

- High-risk cluster: `FindWindowA/W` + `WriteProcessMemory` + `CreateRemoteThread` (injection)
- High-risk cluster: `CryptEncrypt` / `CryptAcquireContext` + lots of `FindFirstFile` / `DeleteFile` (ransomware tendency)
- High-risk cluster: `InternetOpen` / `WinHttp` / `URLDownloadToFile` + persistence APIs (`RegSetValue` / `CreateService`)
- Standalone `CreateFile` / `ReadFile` are usually benign noise, unless co-occurring with the clusters above

### 1.2 Unpacking and IAT Handling (High-Risk Fork · Issue #65)

```text
Branch A: no packer / .NET managed
  → go straight to §2 Static (.NET via the equivalent anchor)

Branch B: packed / heavily obfuscated
  Step 1: attempt unpacking (automatic unpacker / manual OEP location) — must be in an authorized and isolated environment
  Step 2: attempt IAT repair
    Tools: x86 → ImportREC (or equivalent); x64 → Scylla (or equivalent). Do not grind 64-bit samples with ImportREC.
    Case B1: repair succeeded and is parseable → record E-imports (post-repair) → §2 Static
    Case B2: ImportREC/Scylla errors out, the sample cannot run after repair, or the IAT is entirely garbled (VMP/encrypted packer)
      → 【IAT Repair Iron Rule】immediately stop further static IAT repair
      → MUST record E-iat-repair-fail (command, tool, failure symptom, decision to go dynamic)
      → go straight to §3 Dynamic: API breakpoints / hardware execution breakpoints / memory search to capture imports
      → this is not "skipping the import table": the import-table path was attempted and recorded in Evidence
    Case B3 (Patch 6): after unpacking and IAT repair the sample crashes on double-click / BSODs (suspected file CRC/size self-check)
      → stop patching the file statically; record E-self-check-crash or fold it into E-iat-repair-fail
      → move to §3 Dynamic: breakpoint CreateFile / GetFileSize / hash-related APIs to locate the check-bypass point
```

**IAT repair iron rule (MUST)**: prefer automatic/semi-automatic repair first; once the repair tool errors out or the sample cannot run after repair, **immediately stop** grinding on the static import table, switch to dynamic debugging, and capture the imported functions at runtime with API breakpoints (e.g., `bp CreateFile` / key network APIs).

## 2. Static (Basic Static Anchors → Deep Dive)

| Tool | When |
|------|------|
| radare2 / rabin2 | fast function/import/string work (imports were already completed in Triage MUST, or a failure bypass is recorded) |
| IDA / Ghidra (MCP or headless) | deep dive, cross-references, types; re-check the imports classification at the survey stage |
| jadx / dnSpy | Android / .NET |
| OLLVM docs | when control-flow flattening is suspected |

```text
□ Confirm E-imports / E-triage already contains the import table or equivalent-anchor Evidence (if missing, backfill first — no deferring)
□ If DLL/SYS: confirm E-exports is recorded
□ Sensitive-API grouping + high-risk combination clustering (Patch 8)
□ Hardcoded domain/IP/URL strings; whether the resource section hides a payload
□ Locate key functions (crypto/checksum/network/licensing) → addresses/symbols into Evidence
□ One path blocked → switch tools (IDA↔r2↔Ghidra)
□ Time box (Patch 9 · SHOULD default): ~15 minutes of static deep-dive without a key path → force the move to §3 Dynamic (duration may be overridden by the user/task)
```

**Without MCP**: you may export the decompiled text and analyze it offline (following the P4nda0s reverse-skills / IDA-NO-MCP idea), still writing the Evidence path.

## 3. Dynamic (Cross-Validation Loop Zone)

Core idea: **static provides clues → dynamic verifies → verification stalls → back to static review** (no fixed mandatory order).

### 3.0 Breakpoint Opening Moves (Patch 7 + 10 · MUST order)

Before launching the sample in a user-mode debugger (x64dbg, etc.), preset breakpoints in the "four-level escalation" order (names may vary slightly by architecture/tool; the order does not change):

1. **TLS callback** breakpoint (may already have run before the debugger's EP)
2. **Entry point EP** breakpoint
3. **Sensitive API** breakpoints (e.g., `CreateRemoteThread` / network / file writes)
4. **Safety net**: `ExitProcess` / process-exit-path breakpoint (Patch 10) — if the sample exits directly due to anti-debug, **do not rush to restart**; dump memory immediately and write the pre-crash image path into Evidence for string/data recovery

```text
□ Frida / x64dbg / gdb / emulator: verify static hypotheses
□ Run after presetting breakpoints per §3.0; single-step and track stack/registers (white box)
□ Behavior monitoring: sandbox / Procmon / RegShot (black box)
□ IAT-repair-failed / self-check-crash samples: hardware execution breakpoints or memory search to force-capture APIs; CreateFile/GetFileSize for CRC checks
□ Anti-debug/anti-Frida → reverse-engineering/anti-analysis
□ Android: root detection / SSL pinning bypass scripts generated on demand, **must be on an authorized device**
□ Crash logs drive the next hook round (adaptive loop)
□ Time box (Patch 9 · SHOULD default): ~200 instructions single-stepped without a malicious-behavior clue → force return to static string search / re-anchor (overridable)
```

### 3.1 Sandbox / No-Dynamic-Behavior Emergency Branch (MUST)

```text
No behavior, or immediate exit / infinite sleep
  → check for anti-debug / anti-VM routines (CPUID, high-precision timing, sandbox fingerprints, etc.)
  → try hardware breakpoint bypass, patch the detection points, or move to a physical machine / higher-fidelity environment
  → write "no behavior + suspected anti-VM" into Evidence; forbidden to write "sample is harmless" without that condition
```

### 3.2 Time-Box Strategy (Patch 9 · SHOULD)

| Phase | Default threshold (overridable by user/task) | Action |
|------|------------------------------|------|
| Static deep-dive without a key path | ~15 minutes | Move to Dynamic |
| Dynamic single-stepping without progress | ~200 instructions | Return to Static strings/cross-references for re-anchoring |
| Any path fails repeatedly | Record Evidence, then switch tools or bypass | No idling on the same failing method |

### 3.3 Anti-Debug / Obfuscation Bypass Quick Reference (Issue #65 patches A–T · high frequency)

The full index and action details are in `reverse-engineering/anti-analysis.md` "Agent Response Cookbook A–T". Only the **P0 must-checks + common transitions** are listed here. Default is the **authorized isolated lab**; patching/flipping flags is not an unauthorized production action.

| Trigger signature | Primary action (summary) | Evidence |
|----------|------------------|----------|
| `cpuid` followed by jz/jnz (A) | lab: flip the flag or patch to the real branch; record the detection-point address | `E-anti-debug-cpuid` |
| `rdtsc` + sub/cmp (B) | bp rdtsc or hook the time source; no infinite idling waiting for a sandbox timeout to pass as "harmless" | `E-anti-debug-rdtsc` |
| PEB BeingDebugged / NtGlobalFlag (K) | ScyllaHide or hand-edit the PEB; patch the conditional jump | `E-anti-debug-peb` |
| `NtQueryInformationProcess` DebugPort/Flags/Object (P) | ScyllaHide / hook the return value; record the class argument | `E-anti-debug-ntqip` |
| Very few imports but rich behavior → API hashing (N) | bp GetProcAddress; reverse-lookup the hash and feed it back into IDA | `E-api-hash` |
| Empty strings but network/file behavior → string encryption (I) | find the decode routine's xref; dump and feed the decrypted text back | `E-string-decrypt` |
| Signed but dubious source (F) | SigCheck: valid/revoked/timestamp; **invalid does not lower** the threat level | `E-sig-forge` |
| Standard strings show no IOC → try wide characters (T) | `strings -el` / UTF-16LE; Alt+A unicode | `E-wide-strings` |
| Debugger-name strings / Toolhelp scans (C) | bp the CreateToolhelp32Snapshot chain | `E-anti-debug-procscan` |
| AddVectoredExceptionHandler + deliberate exceptions (D) | bp the VEH registration; analyze the handler | `E-anti-debug-veh` |
| int3 / DR0–DR7 (M) | patch int3; software breakpoints or ScyllaHide to hide hardware BPs | `E-anti-debug-bp` |
| Multiple PE headers/overlapping sections (G) | real mapping of the section table + entropy; do not trust section names | `E-pe-anomaly` |
| File tail beyond the section total = Overlay (J) | extract the overlay; file/entropy; find the loading-offset xref | `E-overlay` |
| .rsrc abnormally large / high-entropy RT_RCDATA (Q) | extract the resource; FindResource chain + decrypt-and-dump | `E-rsrc-payload` |
| DLL loaded only at runtime (R) | check Delay Import; bp the delay-load helper | `E-delay-import` |
| while+switch star-shaped CFG (H) | **See** `ollvm-deobfuscation.md`; if plugins fail, take the dynamic path | `E-cff` |
| Always-true/always-false branches (S) | **See** ollvm / symbolic execution; dynamic results prevail | `E-opaque-pred` |
| `/proc/self/status` TracerPid (L) | **Linux/ELF**; hook or patch; not mandatory on the Windows main path | `E-anti-debug-tracerpid` |

**Constraints**: failed bypasses also record Evidence; forbidden to write "anti-debug triggered exit" as "sample is harmless". The full A–T and P2 items (E compile-time, O junk code) are in the anti-analysis cookbook section.

### 3.4 Non-PE / Multi-Format Bypass (Issue #65 patches U–AV · routing)

Full index: `reverse-engineering/references/nonpe-format-cookbook.md`. Only the **type → entry** is listed here; action details live in the cookbook / the corresponding skill.

| Type | Jump to | P0 Evidence anchor (example) |
|------|------|---------------------------|
| BAT/CMD | cookbook §1 + malware | `E-batch-deobf` |
| PowerShell | cookbook §2 + malware | `E-ps-decode-layer-N` |
| VBA macros | cookbook §3 + malware | `E-vba-pcode` |
| JS heavy obfuscation / JSVMP | **js-reverse** + cookbook §4 | `E-js-vmp` / `E-js-deobf` |
| SYS drivers | kernel-driver-reverse + cookbook §5 | `E-driver-irp-handlers` / `E-driver-ioctl` |
| DLL focus | cookbook §6 (AM ≡ A–T **R**) | `E-dll-tls-dllmain` / `E-exports` |
| Android wiper/hidden icons | **apk-reverse** + cookbook §7–8 | `E-android-wiper-*` / `E-android-hidden-icon-*` |

**Constraints**: do not start a separate "non-PE six-phase" flow; this divides work with §3.3 A–T (PE anti-debug vs multi-format). Authorized lab; wiper/BYOVD/reflective = detection-and-forensics wording.

## 4. Synthesis (IOC / Attack Chain / Report)

### Decision quality overlay (Issue #77)

Before closing Synthesis, apply [analysis-decision-framework.md](../../ops/analysis-decision-framework.md) **P0 checklist**: R41 grounded claims, R4* validated sufficiency, R1 confidence->dynamic, R2 hypothesis exit, R43 deadlock replan (under feasibility gate), R8/R23 no default malice/IOC. Multi-module -> R50; anti-analysis effort -> R51 + A-T cookbook.

Blindspots (Rust/Go/VMP/injection/OLE/PDF/agent-meta): [analysis-blindspot-cookbook.md](../../ops/analysis-blindspot-cookbook.md) R52-R81 — detection-oriented; not a parallel master flow.

```text
□ Finding: algorithm/checksum logic/exploitable points / behavior conclusions
□ Path: callflow or solve steps, anchored to E-*
□ IOC: network fingerprints + host fingerprints (table if present; if absent, n/a + reason)
□ Report docs-generator (malware/apt/null/vuln overlays chosen by task) + optional diagrams
□ Optional: YARA / Snort·Suricata rule standardization
□ field-journal desensitization
```

## 5. Six-Phase Practical Mapping (Issue #65 mind map → this file)

| Practical phase | Section in this file | Hard gate / iron rule |
|----------|------------|-------------|
| 1 Initial quick assessment | §0–§1 Triage | Hash, architecture, file type, packer check; imports/equivalent anchors; §0.5 instruction gate |
| 2 Unpacking and IAT | §1.2 | IAT iron rule; failure/self-check crash → Evidence → Dynamic |
| 3 Basic static anchors | §2 Static | high-risk API combinations; time box SHOULD |
| 4 Deep cross-validation | §3 Dynamic | four-level breakpoint escalation; no-behavior emergency; time boxes; §3.3 A–T; §3.4 U–AV type routing |
| 5 Extract IoCs and attack chain | §4 Synthesis | IOC + Kill Chain / Path |
| 6 Archive and standardize | §4 + docs-generator / YARA | structured report; rules optional |

## 6. Differences from "Stacked RE Skill Plugins"

- This package uses **phase gates + tool-index**, and does not default to enabling Hex-Rays-style "unsafe full-auto execution" plugins
- Dynamic instrumentation defaults to the **offline/lab** network_profile
- IAT/import tables: **attempt + record** takes priority over "infinite static grinding" or "silent skipping"
- User instructions: **goal-first + prerequisite negotiation**; forbids impersonating the named step with an unrelated step
