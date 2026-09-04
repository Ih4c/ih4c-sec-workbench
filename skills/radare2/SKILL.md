---
name: radare2
description: |
  Use this skill whenever the user wants to analyze binaries with radare2/r2 from the command line, including reverse engineering, disassembly, function analysis, strings/import inspection, patching, binary diffing, hex inspection, or r2 scripting. Also use it when the user mentions PE/ELF/Mach-O/DEX/WASM files together with CLI analysis, `rabin2`, `rasm2`, `radiff2`, `r2pipe`, or asks for radare2 command help on Windows/Linux/macOS.
---

# radare2

A binary analysis skill built around the `radare2` CLI. The focus is on completing recon, analysis, localization, export, and lightweight modification directly from the command line, without relying on a GUI.

## ACTION REQUIRED (Execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm that this skill's operations are pre-authorized routine work
2. `NOW`: Confirm whether the current task falls within this skill's scope
3. `NEXT`: Read `../tool-index.md` to verify tool availability and real paths
4. `NEXT`: Call bootstrap when tools are missing — never guess paths
5. `ACT`: Enter the first step of the workflow and execute; do not stop at a confirmation state

## Scope

Prefer this skill when the user intends to:

- Analyze `exe`, `dll`, `so`, `elf`, `apk`, `dex`, `wasm` and similar files with `r2` / `radare2`
- Ask how to use `rabin2`, `rasm2`, `radiff2`, `rahash2`, `rax2`
- Do command-line disassembly, view functions, strings, imports/exports, cross-references, or apply patches
- Write `radare2` batch commands, `-c` automation commands, or `r2pipe` scripts

If the user explicitly wants GUI reversing, Hex-Rays-style pseudocode, or an IDA workflow, prefer `ida-reverse`. For web JS reversing, prefer `reverse-engineering`.

## Environment Check First

Do not assume `r2` is available. Check first:

```powershell
r2 -v
rabin2 -v
```

If not installed, check common install locations or suggest installation.

Common Windows executables:

- `radare2.exe`
- `rabin2.exe`
- `rasm2.exe`
- `radiff2.exe`
- `rahash2.exe`
- `rax2.exe`
- `r2pm.exe`

## Bundled Resources

This skill ships two resources. Reuse them first; do not improvise a new set of duplicate commands each time.

### `scripts/recon.ps1`

A standard recon script, suitable for a first-pass overview analysis. It outputs:

- Basic info
- Sections
- Imports
- Exports
- Strings
- Optional `r2 -A` auto-analysis summary

Usage:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "C:\path\to\sample.exe"
```

To also run `r2` auto-analysis:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "C:\path\to\sample.exe" -RunAnalysis
```

### `references/cheatsheet.md`

When you need more command detail, common scenario templates, or a quick syntax refresher, read this cheatsheet instead of guessing from memory.

## Known Phenomena

### Occasional `.sdb` missing warning on Windows

During `rabin2` recon of some PE files, a warning like the following may appear:

```text
ERROR: Cannot find ...\share\format\dll\*.sdb
```

If the main output still returns normally, this usually does not affect basic recon conclusions — continue the analysis. Do not declare analysis failure just because of such an incidental warning.

## Basic Principles

### 1. Recon first, dig deep later

Do not run a full auto-analysis immediately. First use lightweight commands to confirm file type, architecture, entry point, strings, and import table, then decide whether to run `aaa`, `aaaa`, or targeted analysis.

### 2. Prefer the minimal sufficient command set

`radare2` has a huge command surface; users usually only need the shortest path:

- File info: `rabin2 -I`
- Strings: `rabin2 -z`
- Imports/exports: `rabin2 -i` / `rabin2 -E`
- Interactive analysis: enter `r2 <file>`, then run local commands

### 3. Be careful before modifying

If the user wants to patch a binary:

- Default to read-only first: `r2 <file>`
- Use write mode only when modification is clearly needed: `r2 -w <file>` or `oo+` inside a session
- Warn about the risks before modifying, to avoid accidentally overwriting the original file

## Common Workflows

## Workflow 1: Quick Recon

Best when you have just received a binary file.

### Hard gate (MUST — workflow 2 and beyond are forbidden until met)

For binaries with an import table (PE/ELF/Mach-O, etc.), you **MUST** complete an import-table check and record it as Evidence before moving on to function-level analysis or dynamic steps:

1. Run `rabin2 -i <sample>` (or the imports section of `recon.ps1` output); for DLL/SYS also MUST run `rabin2 -E` and record `E-exports`
2. Write the full/categorized import-table result into Evidence (suggested id: `E-imports` or `E-triage-imports`), at minimum containing:
   - Reproducible command (`repro_command`)
   - Categorized summary of key imports: network / file / crypto / process injection / registry / other suspicious APIs
   - If the import table is empty, fails to parse, or the tool errors: still MUST record the failure and raw output as Evidence — **never silently skip**
   - An import table that is "too clean" (only basic DLLs): MUST note suspicion of dynamic loading, SHOULD move to dynamic API capture
3. .NET and other binaries without a traditional IAT: MUST go through the equivalent anchor (dnSpy/IL/metadata summary) into the same Evidence semantics slot — no empty pass-through
4. Packed-sample IAT repair: x86 use ImportREC (or equivalent), x64 use Scylla (or equivalent). On repair failure MUST record `E-iat-repair-fail` then move to dynamic API breakpoints; **forbidden** to grind indefinitely on the static IAT (see `reverse-engineering/references/re-agent-workflow.md` §1.2)
5. When the user explicitly asks to "redo the import-table check / recheck imports / redo the IAT": MUST redo the named step itself (when blocked, go through the feasibility gate first: state the prerequisite + ask confirmation; if forced, mark quality=unreadable), **forbidden** to substitute an unrelated step to fake completion

Before an import-table (or legitimate equivalent anchor / IAT-failure bypass) Evidence entry is recorded: MUST NOT claim "basic recon complete", MUST NOT enter workflow 2+ deep-dive conclusions.

Prefer running the bundled script directly:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "sample.exe"
```

If you only need the manual minimal commands:

```powershell
rabin2 -I sample.exe
rabin2 -z sample.exe
rabin2 -i sample.exe
rabin2 -E sample.exe
```

Focus points:

- File format, bitness, architecture, platform
- Entry point address
- Suspicious strings: URLs, paths, error messages, registry, command-line arguments
- Imported functions: network, file, crypto, process injection, registry operations (**MUST be recorded as Evidence, see hard gate above**)

## Workflow 2: Interactive Function Analysis

```powershell
r2 sample.exe
```

Once inside, commonly used:

```text
aaa          # standard auto-analysis
afl          # list functions
iz           # list strings
iS           # list sections
is           # list symbols
s entry0     # jump to entry point
pdf          # disassemble current function
VV           # enter visual mode (if the terminal is suitable)
q            # quit
```

Notes:

- Prefer `aaa` by default; do not start with the heavier `aaaa`
- If the sample is large or analysis is slow, analyze near the entry point first and expand manually

## Workflow 3: Locating main / Key Logic

```text
afl~main
afl~sym.
iz~http
iz~error
axt <addr>
```

Approach:

- Start from `main`, the entry point, and string references
- Use `axt` to find who references a given string or address
- Once the reference point is found, `s <addr>` then `pdf`

## Workflow 4: Hex and Memory Inspection

```text
px 64        # 64 bytes of hex from current address
pd 20        # disassemble 20 instructions
psz          # read string at current address
pxa          # friendlier hex view
```

## Workflow 5: Binary Patching

Only use when the user explicitly asks to modify the file:

```powershell
r2 -w sample.exe
```

Once inside, for example:

```text
s 0x401000
wa nop
wa jmp 0x401050
wq
```

Common write operations:

- `wa <asm>`: write assembly
- `wx <hex>`: write raw bytes
- `wq`: write and quit

Back up the original file before modifying. If the user did not mention a backup, remind them at least once.

## Workflow 6: Non-Interactive Automation

Suitable for one-shot output:

```powershell
r2 -A -q -c "afl;iz;ii;q" sample.exe
```

Common flags:

- `-A`: auto-analyze at startup
- `-q`: quiet mode
- `-c`: run a command string

If there are many commands, prefer arranging them in a readable order rather than cramming an unmaintainable mega-string.

It is better to start from the bundled recon script and then decide whether custom commands are needed.

## Common Sub-Tools

### `rabin2`

Suitable for static info extraction:

```powershell
rabin2 -I sample.exe   # basic info
rabin2 -S sample.exe   # sections
rabin2 -s sample.exe   # symbols
rabin2 -i sample.exe   # imports
rabin2 -E sample.exe   # exports
rabin2 -z sample.exe   # strings
rabin2 -zz sample.exe  # more detailed strings
```

### `rasm2`

Suitable for quick assemble/disassemble:

```powershell
rasm2 -d "9090"
rasm2 -a x86 -b 64 "xor eax, eax"
```

### `radiff2`

Suitable for diffing two binaries:

```powershell
radiff2 old.exe new.exe
radiff2 -C old.exe new.exe
```

### `rahash2`

Suitable for hashing:

```powershell
rahash2 -a md5 sample.exe
rahash2 -a sha256 sample.exe
```

### `rax2`

Suitable for radix and encoding conversions:

```powershell
rax2 0x401000
rax2 4198400
rax2 -s hello
```

## Recommended Analysis Order

When facing an unknown sample, follow this order:

1. `rabin2 -I` — format, architecture, entry point
2. `rabin2 -z` — strings
3. `rabin2 -i` — imported functions — **MUST + Evidence (hard gate, see Workflow 1)**
4. If interactive analysis is needed, enter `r2` (only after step 3's Evidence is recorded)
5. `aaa` first, then `afl` / `iz` / `pdf`
6. Locate key functions progressively via string references, import calls, and entry flow

The benefit of this order is low noise and fast orientation. Step 3 is not an optional optimization; it is the hard gate before deep diving.

## Windows Notes

- When paths contain spaces, quote commands correctly
- If `r2` is not found in the current terminal, PATH may have just been updated — open a new terminal and retry
- Some samples require administrator rights to read, but do not proactively elevate by default unless the user explicitly needs it
- Before dynamic debugging of a suspicious sample, confirm the user's intent to avoid mishandling

## Output Style

When the user wants you to actually analyze a file rather than just list commands:

- First give the recon result summary
- Then list key evidence: strings, imports, functions, addresses
- Finally give next-step suggestions or continue the deep dive

Do not just enumerate commands without explaining why.

## Typical Request Examples

### Example 1: Analyze an exe

User: `Take a look at what this exe does — radare2 is fine`

Approach:

1. Run `rabin2 -I/-z/-i` first
2. Decide whether to enter `r2`
3. Use `aaa`, `afl`, `pdf` to dig into the entry point and key string references

### Example 2: Find where a string is called

User: `Which function triggers this error string?`

Approach:

1. Use `iz~keyword` to find the string address
2. Use `axt <addr>` to find references
3. Jump to the reference point `s <addr>` then `pdf`

### Example 3: Change a jump

User: `Change this jne to je`

Approach:

1. Confirm the target address first
2. Explicitly state that write mode is required
3. Use `wa je <target>` or directly `wx`
4. Re-disassemble after the change to verify

## Practices to Avoid

- Do not treat `radare2` as a tool with only the `aaa` command
- Do not open the user's file in write mode without explaining the risks
- Do not draw conclusions before basic recon is done
- **Forbidden to skip the import-table check** (`rabin2 -i` / recon imports): without an Evidence entry you must not proceed; when the user asks to redo the import table, you are forbidden to switch to a different step instead
- **Forbidden to grind statically after IAT repair fails**: record `E-iat-repair-fail` then move dynamic; forbidden to use only ImportREC on 64-bit samples
- Do not misroute web JS reversing into this skill; that is `reverse-engineering`'s scope

## Reference Material

- Command cheatsheet: `references/cheatsheet.md`
- Standard recon script: `scripts/recon.ps1`

## radare2-skills Ecosystem

The radare2-skills project (radareorg/radare2-skills) offers a fuller ecosystem of tools and workflows:

- **r2xsql**: SQL queries over a binary's import table / strings / functions
- **r2mcp / r2http**: MCP tooling and HTTP stateful command channels
- **radius2**: symbolic execution, symbolic dynamic analysis
- **r2pm**: plugin management, extensions
- **decompiler plugins**: radare2 plugin mechanism

**Usage strategy**:
- When the user mentions `r2xsql`, `r2mcp`, `r2http`, `radius2`, `r2pm`, `rabin2`, `rasm2`, `radiff2`, `rahash2`, `rax2`, route to this skill (radare2/SKILL.md) first
- These tools are only ecosystem accelerators and **cannot bypass**: the authorization gate, `tool-index` verification, Evidence recording of imports, or write-mode confirmation
- Provide minimal reproducible command examples:
  - `r2xsql -s <file> -q "SELECT ..."`
  - `curl.exe -sS --data-binary 'aaa' http://127.0.0.1:9393/cmd`
  - `radius2 -p <binary> ...`
  - `r2pm -ci <plugin>`

This skill keeps the original hard gates and Evidence-chain integrity; no authorization or Evidence step may be skipped.

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master control), `routing.md`
**Upstream alternatives**: `ida-reverse/` (escalate to IDA when decompilation/pseudocode is needed)
**Downstream exits**:
- Dynamic analysis needed → `reverse-engineering/tools-dynamic.md` (Frida/GDB)
- Deep decompilation needed → `ida-reverse/`
- After PAT finds interesting strings that need cross-referencing → `ida-reverse/` (IDA xref is more powerful)

**Sibling modules**: `ida-reverse/` (complementary: r2 recon is fast, IDA decompiles deep)

## On-Demand Bootstrap

This skill's entry scripts hook into the unified bootstrap system. When radare2 is missing, they do not fail directly but attempt an automatic install.

### Automation Capability Boundaries

| Tool | Auto-installable | Install method | Notes |
|------|-----------|---------|------|
| r2 | Yes | GitHub Release ZIP (w64) | Auto-downloaded and extracted to `%USERPROFILE%\Tools\radare2\` |
| rabin2 | Yes | Same as above (bundled in the radare2 release) | — |
| rasm2 | Yes | Same as above | — |
| radiff2 | Yes | Same as above | — |
| rahash2 | Yes | Same as above | — |
| rax2 | Yes | Same as above | — |

### Bootstrap Trigger Points

- `scripts/recon.ps1`: automatically calls `bootstrap-reverse.ps1` when `rabin2` or `r2` is missing

### When Bootstrap Fails

If the automatic install fails (no network, GitHub API rate limit, etc.), the script raises a clear error with a manual install link.

Manual install: download `radare2-*-w64.zip` from https://github.com/radareorg/radare2/releases, extract to `%USERPROFILE%\Tools\radare2\`, and make sure the `bin\` directory is on PATH.


## Task Completion Self-Check (MUST pass before claiming completion)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Was the import-table check executed and recorded as Evidence (E-imports / E-triage-imports or .NET equivalent)? Does the DLL/SYS include E-exports?
- [ ] Was an IAT repair failure recorded as E-iat-repair-fail and did I move dynamic? Did redo requests return to the same step?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the Checklist items required by RULES?
