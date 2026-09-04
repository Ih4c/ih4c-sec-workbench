---
name: binary-diff
description: |
  Cross-version symbol migration and binary diffing (跨版本符号迁移与二进制差分). Use when you have symbols/reversing results for an old version and need to migrate them quickly to a new version.
  Scenarios: kernel without PDB — derive symbols from an old version (内核缺 PDB 用旧版符号推导), batch-migrate function names after a program update (程序更新后批量迁移函数名), quickly locate new offsets after an app update (应用更新后快速定位新偏移).
  Core method: structured LLM diffing with programmatic input/output at very low cost (200 functions ~1 yuan).
  Trigger keywords (触发关键词): 符号迁移、bindiff、跨版本、PDB 缺失、函数偏移迁移、symbol migration、binary diff、版本对比.
---

# Cross-Version Symbol Migration (Binary Diff)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: Confirm whether the current task falls within this skill's scope of application
3. `NEXT`: Read `../tool-index.md`, verify tool availability and actual paths
4. `NEXT`: If a tool is missing, invoke bootstrap; never guess paths
5. `ACT`: Enter step one of the "Workflow" below and execute — do not stop at an acknowledgment state

## Scope of Application

Use this skill when the task matches one of the following scenarios:

1. **Kernel/driver missing PDB** — you have symbols for an old ntoskrnl.exe and Microsoft has pulled the PDB for the new version; derive the addresses of the new version's non-exported functions using the old symbols
2. **Symbol migration after a program update** — you reversed a program once, the program was updated, and you do not want to re-reverse everything; batch-migrate using the old results
3. **Protection mechanism updates** — the old version has complete reversing results and you need to quickly locate the new offset of the same function in the new version
4. **Any "old version has symbols + new version has none" binary comparison scenario**

### Division of Labor with Other Skills

| Scenario | What to use |
|------|--------|
| Reversing a binary from scratch | `ida-reverse/` or `radare2/` |
| Have old-version results, migrating to the new version | **This skill** |
| Comparing two completely different binaries | BinDiff / Diaphora (traditional tools) |

### Core Advantages

Compared with traditional approaches:

| Approach | Cost for 200 functions | Time | Accuracy |
|------|--------------|------|--------|
| Manually comparing in two IDA windows | Free but drains your life | Hours | High |
| BinDiff automatic matching | Free | Fast | Medium (fails when structure changed heavily) |
| Handing everything to an Agent (CC/Codex) | 50-100 yuan | Slow | High |
| **This skill (LLM batch diffing)** | **~1 yuan** | **~10 sec/function** | **High** |

## Core Principle

```text
Old-version function (with symbols)      Same function in new version (no symbols)
    ↓                              ↓
Export disassembly + pseudocode    Export disassembly + pseudocode
    ↓                              ↓
    └──────── LLM structured comparison ────────┘
                    ↓
         Output YAML (symbol mapping table)
                    ↓
         Programmatic parse → batch apply to new IDB
```

Key points:
- The prompt is a fixed template, filled programmatically
- Input/output formats are deterministic, parsed programmatically
- The LLM only handles "look at two code snippets and find the correspondence"
- Time cost and token cost are extremely low

## Prompt Template

### Standard Comparison Prompt

```text
I have disassembly outputs and procedure code of the same function.

This is the function for reference:

**Disassembly for Reference**
```c
{disasm_for_reference}
```

**Procedure code for Reference**
```c
{procedure_for_reference}
```

This is the function you need to reverse-engineering:

**Disassembly to reverse-engineering**
```c
{disasm_code}
```

**Procedure code to reverse-engineering**
```c
{procedure}
```

What you need to do is to collect all references to "{symbol_name_list}" in the function you need to reverse-engineering and output those references as YAML.

Example:
```yaml
found_vcall: # This is for indirect call to virtual function or virtual function pointer fetching.
  - insn_va: '0x180777700' # Always be the instruction with displacement offset
    insn_disasm: call [rax+68h] # Always be the instruction with displacement offset
    vfunc_offset: '0x68'
    func_name: ILoopMode_OnLoopActivate
  - insn_va: '0x180777778' # Always be the instruction with displacement offset
    insn_disasm: mov rax, [rax+80h] # Always be the instruction with displacement offset
    vfunc_offset: '0x80'
    func_name: INetworkMessages_GetNetworkGroupCount

found_call: # This is for direct call to non-virtual regular function.
  - insn_va: '0x180888800'
    insn_disasm: call sub_180999900
    func_name: CLoopMode_RegisterEventMapInternal
  - insn_va: '0x180888880'
    insn_disasm: call sub_180555500
    func_name: CLoopMode_SetSystemState

found_funcptr: # This is for non-virtual regular function pointer.
  - insn_va: '0x180666600' # Must load/reference the function pointer target address
    insn_disasm: lea rdx, sub_15BC910 # Must load/reference the function pointer target address
    funcptr_name: CLoopMode_OnClientPollNetworking

found_gv: # This is for reference to global variable.
  - insn_va: '0x180444400'
    insn_disasm: mov rcx, cs:qword_180666600 # Must load/reference the global variable
    gv_name: g_pNetworkMessages
  - insn_va: '0x180333300'
    insn_disasm: lea rax, unk_180222200 # Must load/reference the global variable
    gv_name: s_EventManager

found_struct_offset: # This is for reference to struct offset. NOTE THAT virtual function pointer should not be here! virtual function pointer should ALWAYS be in found_vcall !
  - insn_va: '0x1801BA12A' # Always be the instruction with displacement offset
    insn_disasm: mov rcx, [r14+58h] # Always be the instruction with displacement offset
    offset: '0x58'
    size: 8
    struct_name: CResourceService
    member_name: m_pEntitySystem
```

If nothing found, output an empty YAML. DO NOT output anything other than the desired YAML. DO NOT collect unrelated symbols.
```

### Variable Guide

| Variable | Source | Description |
|------|------|------|
| `{disasm_for_reference}` | Exported from old-version IDA | Disassembly with symbols |
| `{procedure_for_reference}` | Exported from old-version IDA | Pseudocode with symbols |
| `{disasm_code}` | Exported from new-version IDA | Disassembly without symbols |
| `{procedure}` | Exported from new-version IDA | Pseudocode without symbols |
| `{symbol_name_list}` | Extracted from the old version | List of symbols to locate in the new version |

## Workflow

### Full Flow

```text
Step 1: Prepare data
  - Load the old-version binary into IDA (with PDB/symbols)
  - Load the new-version binary into IDA (no symbols)
  - Find identical anchor functions in both versions (exported functions, string references, etc.)

Step 2: Batch export
  - From the old version, export: disassembly + pseudocode of the anchor functions (with symbol names)
  - From the new version, export: disassembly + pseudocode of the same anchor functions (without symbol names)

Step 3: LLM comparison
  - Fill the data into the prompt template
  - Call the LLM API (recommended: deepseek for high volume at low cost, switch to gpt for very large functions)
  - Parse the returned YAML

Step 4: Apply results
  - Batch-apply the symbol mappings in the YAML to the new IDB
  - Batch rename with idapro_rename or an IDAPython script

Step 5: Iterate
  - Functions migrated in the first round become new anchors
  - Go into those functions and keep comparing internal calls
  - Repeat until all target functions are covered
```

### Anchor Selection Strategy

| Anchor type | Reliability | Description |
|---------|--------|------|
| Exported functions | Highest | Name unchanged, address may change |
| String references | High | String content unchanged, reference location may change |
| Constants/magic numbers | Medium | Signature values unchanged |
| Code patterns | Medium | Function structure similar but addresses all changed |

### Batch Processing Suggestions

- Compare 1 function at a time (avoid context explosion)
- Medium functions (<200 lines) use deepseek
- Very large functions (>500 lines) switch to gpt-4o or claude
- Concurrent calls speed things up (10-20 concurrent)
- Cache results to avoid repeated calls

## Output Format

### The 5 Symbol Types in YAML Output

| Type | Meaning | Key fields |
|------|------|---------|
| `found_vcall` | Virtual function call (indirect call) | `vfunc_offset`, `func_name` |
| `found_call` | Direct function call | `insn_va`, `func_name` |
| `found_funcptr` | Function pointer reference | `insn_va`, `funcptr_name` |
| `found_gv` | Global variable reference | `insn_va`, `gv_name` |
| `found_struct_offset` | Struct offset reference | `offset`, `struct_name`, `member_name` |

### Apply Actions After Parsing

```text
found_call → idapro_rename(addr=call_target, name=func_name)
found_vcall → idapro_set_comments(addr=insn_va, comment="vcall: {func_name} @ +{offset}")
found_funcptr → idapro_rename(addr=funcptr_target, name=funcptr_name)
found_gv → idapro_rename(addr=gv_addr, name=gv_name)
found_struct_offset → idapro_set_comments(addr=insn_va, comment="{struct_name}.{member_name}")
```

## Typical Scenario Examples

### Scenario 1: ntoskrnl.exe missing PDB

```text
Have: ntoskrnl.exe 10.0.26100.2000 + full PDB
Target: ntoskrnl.exe 10.0.26100.2605 (PDB pulled)
Need: locate the new address of PspSetCreateProcessNotifyRoutine

Steps:
1. Load both versions into IDA
2. Find the exported function PsSetCreateProcessNotifyRoutine (present in both versions)
3. In the old version it calls PspSetCreateProcessNotifyRoutine (has symbols)
4. In the new version it calls sub_140822108 (no symbols)
5. The LLM sees it at a glance: sub_140822108 = PspSetCreateProcessNotifyRoutine
6. Batch apply
```

### Scenario 2: Migration After an App Update

```text
Have: full reversing results for target.exe v1.0 (200+ named functions)
Target: target.exe v1.1 (all symbols lost)
Need: batch-migrate 200 function names

Steps:
1. Export disassembly+pseudocode of all named functions from the old version
2. Locate the corresponding anchors in the new version via exported functions/strings
3. Batch-call the LLM for comparison
4. Parse the YAML, batch rename
5. Iterate deeper
```

## LLM Selection Suggestions

| Model | Best for | Cost | Speed |
|------|---------|------|------|
| DeepSeek V3 | Small/medium functions (<200 lines), batch processing | Very low | Fast |
| GPT-4o | Very large functions, complex control flow | Medium | Fast |
| Claude Sonnet | Medium/large functions that need reasoning | Medium | Fast |
| Claude Opus | Extremely complex functions needing deep understanding | High | Slow |

Recommended strategy: DeepSeek by default; auto-upgrade when the context overflows or results are inaccurate.

## Notes

- **Do not throw the whole binary at the LLM** — compare only one function at a time
- **Anchors must be reliable** — if an anchor is matched wrong, everything downstream is wasted
- **Spot-check results manually** — the LLM is not 100% accurate; verify key symbols
- **Cache intermediate results** — avoid repeated calls that waste tokens
- **Mind the context limit** — very large functions (>1000 lines of disassembly) need splitting or a large-context model

---

## On-Demand Bootstrap

### Tool Dependencies

| Tool | Purpose | Auto-installable |
|------|------|-----------|
| IDA Pro | Export disassembly/pseudocode | ✗ (commercial software) |
| Python | Script execution, API calls | ✓ |
| PyYAML | Parse the YAML returned by the LLM | ✓ (pip install pyyaml) |
| LLM API | Perform the comparison | Needs an API key |

### Notes

This skill's core does not depend on heavy tool installation; it mainly relies on:
- IDA Pro already present (managed by the `ida-reverse/` skill)
- Python + requests/httpx (API calls)
- An LLM API endpoint

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master control), `routing.md`
**Trigger condition**: old-version symbols/reversing results exist and need migrating to the new version
**Downstream exits**:
- Binary needs to be opened first → `ida-reverse/`
- Quick recon to confirm version differences → `radare2/`

**Peer module**: `ida-reverse/` (both data export and symbol application go through IDA)


## Task Completion Self-Check (MUST pass before claiming done)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
