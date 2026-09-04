---
name: dotnet-reverse
description: .NET / C# 二进制逆向。Reverse engineering of .NET / C# binaries. Use when the target is a .NET assembly (managed .exe/.dll, PE header with CLR), a C# compiled artifact (including NativeAOT), a red-team Sharp* tool (Rubeus / SharpHound / SharpHound etc.), an obfuscated .NET program (ConfuserEx / SmartAssembly / Babel / Eazfuscator), or a .NET loader / info-stealer / shell-wrapped malware. 优先用 dnSpyEx + de4dot；需要 AI 直接操作时联动 dnSpy MCP。Not for pure native binaries (route to reverse-engineering / ida-reverse).
license: MIT
compatibility: Requires a filesystem-based code agent or CLI with shell access; Windows host preferred (dnSpyEx is a Windows GUI); Linux/macOS can use the ILSpy/de4dot CLI + mono/dotnet runtime.
allowed-tools: Bash Read Write Edit Glob Grep Task WebFetch WebSearch
metadata:
  user-invocable: "false"
---

# .NET / C# Reverse Work Standards

## ACTION REQUIRED (Execute Immediately After Reading)

1. `NOW`: Confirm with DIE/`file`/the CLR header that the target is .NET managed (otherwise SWITCH to `ida-reverse/` / `reverse-engineering/`)
2. `NOW`: If obfuscation is suspected → run `de4dot` first to unpack, producing `*-clean.exe`, and keep the original sample
3. `NEXT`: Static analysis with dnSpyEx (or the dnSpy MCP / `ilspycmd`): browse the C# view and inspect key branches in the **IL view**
4. `ACT`: When plaintext/C2 is needed, debug dynamically; when logic must change, prefer **IL patching** over C# recompilation
5. At the end of each phase, offer the user a 3-6 item next-step menu (including report export)

## Applicable Scope

Prefer this skill when the task falls into these scenarios:

- Identifying and reversing .NET / C# compiled artifacts (managed PE / .exe / .dll)
- Analyzing red-team Sharp* toolchains (Rubeus, SharpHound, SharpShell, etc.)
- Deobfuscating packers such as ConfuserEx / SmartAssembly / Babel / Eazfuscator / .NET Reactor
- Reversing the decryption and C2 logic of .NET loaders / info-stealers / RATs
- Patching C# programs (changing branches, changing constants, keygen)
- Analyzing the Mono/Unity managed layer before IL2CPP (note: IL2CPP output is native — go to `reverse-engineering/` + seed-014)

If the target is a pure native binary (C/C++/Go/Rust compiled, no CLR), use `reverse-engineering/`, `ida-reverse/`, or `radare2/` instead.

## Core Principles

- **Identify before acting**: first confirm the program is .NET managed (CLR in the PE header + `#~` / `#Strings` streams + the mscoree `_CorExeMain` entry), then decide to use dnSpy rather than IDA
- **IL over C#**: dnSpyEx's C# decompiler loses/distorts information (compiler-generated state machines, async/await, yield); key branches and patches MUST be done in the **IL editor**; the C# view is only for quick browsing
- **de4dot first**: when an obfuscator is present, run `de4dot` for one round before static analysis, otherwise strings/control flow are all garbled
- **MCP linkage**: if a dnSpy MCP (`dnspy_*` tools) is registered in the environment, prefer its tool surfaces for decompile / IL inspection instead of switching back and forth to the GUI
- **Evidence-based output**: deobfuscation artifacts, extracted configuration/C2/key, and patch diffs must all be persisted

## Toolchain Map

| Capability | Preferred | Notes |
|------|------|------|
| Decompile + debug + patch | **dnSpyEx** | the ace; the only GUI with an IL editor; old dnSpy is unmaintained — use the Ex fork |
| Lightweight CLI / headless decompile | **ILSpy** (`ilspycmd`) | good for batch, scripting, Linux/macOS |
| Deobfuscation | **de4dot** | the default cure for ConfuserEx family, SmartAssembly, and other mainstream packers |
| Obfuscator identification | **Detect It Easy (DIE)** / **file** | determine the packer first, then choose the de4dot arguments |
| Programmatic IL manipulation | **dnlib** | write C# scripts to batch-modify metadata / string decryptors |
| Direct AI operation | **dnSpy MCP** | tool surfaces such as `dnspy_decompile` / `dnspy_inspect_il` |

> Prerequisites: on Windows install dnSpyEx + de4dot (choco or releases); on Linux/macOS use `ilspycmd` + the `dotnet runtime`. See the installation matrix in `references/sharp-tools.md`.

## Six-Phase Workflow

### 1. Identify (.NET detection)

Confirm the target is managed — do not analyze a native PE as if it were .NET:

```powershell
# Windows
file target.exe                       # "PE32 executable ... for MS Windows" is not enough
# the key: check for CLR
powershell -c "[System.Reflection.AssemblyName]::GetAssemblyName('target.exe')"
# or
drag it straight into dnSpyEx — if it opens, it is managed

# Generic
strings target.exe | grep -iE "mscoree|_CorExeMain|mscorlib|System\\."
```

**.NET identification markers:**
- PE header `Data Directory[14]` (CLR Runtime Header) is nonzero
- `mscoree.dll` import / `_CorExeMain` entry
- `#~`, `#Strings`, `#US`, `#GUID`, `#Blob` metadata streams
- `mscorlib` / `System.Private.CoreLib` strings

**NativeAOT exception:** compiled to native, no CLR header, but with `System.Private.CoreLib` strings and reworked type metadata — this goes to `reverse-engineering/` (IDA/r2); this skill only flags it for identification.

### 2. Detect (obfuscator detection)

```powershell
# Quick identification with DIE
diec target.exe                        # Detect It Easy CLI
# or drag into dnSpyEx and check for many garbled class names / control-flow obfuscation
```

Common obfuscators → unpacking strategy (details in `references/obfuscators.md`):

| Obfuscator | Features | de4dot handling |
|--------|------|------------|
| ConfuserEx (1.0.0 / 2.x) | `<module>` anti-tamper, control-flow obfuscation, string encryption | `de4dot target.exe` usually auto-detects |
| SmartAssembly | `circular`/`string encoding`, resource compression | `de4dot target.exe` |
| Babel.NET | method body encryption, control flow | `de4dot target.exe` |
| Eazfuscator.NET | string/resource encryption | `de4dot`; some versions need manual work |
| .NET Reactor | anti-tamper + necrobit | `de4dot`; newer versions may fail and need manual work |

### 3. Deobfuscate

```powershell
# de4dot auto-detects most packers by default
de4dot target.exe -o target-clean.exe

# Specify the type (when auto-detection fails)
de4dot --type cfze target.exe          # ConfuserEx
de4dot --type sa target.exe            # SmartAssembly

# Multiple layers / de4dot reports unknown
de4dot --detect target.exe             # see what it identifies
# may need to patch anti-tamper before de4dot (see references/obfuscators.md)
```

Artifact: `target-clean.exe` — use it for the rest of the analysis. **Keep the original sample** for comparison.

### 4. Static Analyze

Load the unpacked sample in dnSpyEx:

- **C# view**: quickly browse class structure, method signatures, strings (for locating)
- **IL view**: key branches, crypto logic, and state machines MUST be viewed in IL (right-click → Edit IL, or the IL view)
- Find the entry: `Main` / `Startup` / the module initializer (`Module .cctor`)
- Find key logic: search for `flag`, `password`, `verify`, `check`, `encrypt`, `http`, `Config`

```text
Locate the string → find reverse references → find the method using it → inspect the branch logic in the IL view
```

### 5. Dynamic (dynamic debugging)

dnSpyEx debugger: attach to the process / start debugging, set breakpoints on key methods, observe at runtime:
- Decrypted plaintext strings (many obfuscators only decrypt strings at runtime)
- C2 addresses, config decryption results
- Exception-driven control flow (anti-debug commonly hides the real path in `try/catch`)

> .NET dynamic debugging is far friendlier than native — you can directly see object values and string contents. Prefer dynamic over grinding through static.

### 6. Patch (modify on demand)

```text
dnSpyEx → right-click the method → Edit Method (C#) or Edit IL
  - change a branch: ldc.i4.0 → ldc.i4.1 (false→true)
  - change a constant: edit the string/number directly
  - remove a check: nop out the whole block
File → Save Module → replace the original file
```

**IL patching is more reliable than C# patching**: C# recompilation can fail (missing references, syntax errors), while IL edits almost never distort the code. See `references/common-workflow.md`.

## Trigger-Scenario Routing

Users enter this skill by saying things like:
- ".NET / C# binary reverse" ("C# 程序反编译" / ".NET / C# 二进制逆向")
- "dnSpy analysis" ("dnSpy 分析" / "dnSpyEx patch")
- "deobfuscate / unpack ConfuserEx / SmartAssembly / Babel" ("ConfuserEx / SmartAssembly / Babel 脱混淆 / 脱壳")
- "Sharp* tool analysis" (Rubeus / SharpHound / SharpShell)
- ".NET malware / loader / info-stealer reverse" (".NET malware / loader / info-stealer 逆向")
- "C# program patch / keygen / branch modification" ("C# 程序 patch / keygen / 修改判断")

## When to Switch Out

- Unity games compiled with IL2CPP → `reverse-engineering/` + `seed-014_unity-il2cpp-reverse.md` (IL2CPP is native; dnSpy does not apply)
- NativeAOT artifacts → `reverse-engineering/` (same reason: native)
- Pure native PE (no CLR) → `reverse-engineering/` / `ida-reverse/`
- Symbols/functions need bulk migration to another version → `binary-diff/`
- Attack path / call-chain diagrams needed → `diagram-generator/`

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master), `routing.md`
**Downstream exit**:
- IL2CPP / NativeAOT (native) → `reverse-engineering/`
- Deep native .so/.dll segment analysis → `ida-reverse/` / `radare2/`
- AI needs to operate dnSpy directly → register and link the dnSpy MCP (see `references/sharp-tools.md`)

**Peer modules**:
- `reverse-engineering/languages-compiled.md` (the .NET intro points to this module)
- `apk-reverse/` (Xamarin/MAUI Android reverse can switch back to this module for the C# layer)

## Reference Documents

- [references/obfuscators.md](references/obfuscators.md) — ConfuserEx / SmartAssembly / Babel / Eazfuscator / .NET Reactor deobfuscation in detail + anti-tamper bypass
- [references/common-workflow.md](references/common-workflow.md) — full workflow, IL patch reliability, string-decryptor extraction, state machine recognition
- [references/sharp-tools.md](references/sharp-tools.md) — red-team Sharp* tool analysis, tool installation matrix, dnSpy MCP integration, community resource index

## Task Completion Self-Check

- [ ] Did I confirm the CLR / managed identity (or already SWITCH out of this skill)?
- [ ] Was the obfuscated sample de4dot / equivalently unpacked before deep analysis?
- [ ] Was key logic verified in the IL view (rather than only C# pseudocode)?
- [ ] Were artifacts (clean sample / config / patch diff) persisted and reproducible?
- [ ] Did I offer a next-step menu or report exit?
