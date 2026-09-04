# Go Binary Reverse Engineering Guide

> Go-compiled binaries present unique challenges: static linking makes them huge, function counts reach the tens of thousands, strings have a special format, and symbol recovery after strip is difficult.
> This document covers the toolchain, recovery techniques, and hands-on workflows.

---

## Identifying Go Binary Characteristics

Quickly determine whether a binary is Go-compiled:

```bash
# String signatures
strings binary | grep -E "runtime\.|go\.buildid|GOROOT"

# rabin2 recon
rabin2 -z binary | grep -i "runtime"

# Abnormally large file size (statically linked runtime)
# Typical Hello World: C ~20KB, Go ~2MB
```

Common characteristics:
- Contains a large number of functions with the `runtime.` prefix
- Contains a `go.buildid` section
- Contains `GOROOT`, `GOPATH` path strings
- Function count 5000-50000+ (includes the whole runtime and standard library)

---

## Core Toolchain

### Symbol Recovery

| Tool | Use | Link |
|------|------|------|
| **GoReSym** | From Mandiant; parses Go symbol info (pclntab/moduledata) | https://github.com/mandiant/GoReSym |
| **GoResolver** | From Volexity; automatically deobfuscates Garble binaries via CFG similarity | https://github.com/volexity/GoResolver |
| **redress** | Analyzes stripped Go binaries; recovers types/interfaces/package structure | https://github.com/goretk/redress |
| **GoStringUngarbler** | From Google; specializes in recovering Garble-obfuscated strings | https://github.com/mandiant/GoStringUngarbler |

### IDA Plugins

| Tool | Use | Link |
|------|------|------|
| **go_parser** | IDA plugin that parses moduledata/pclntab/type info | https://github.com/0xjiayu/go_parser |
| **IDAGolangHelper** | IDA script collection that parses Go type info | https://github.com/sibears/IDAGolangHelper |
| **AlphaGolang** | SentinelLabs' IDAPython script collection | https://github.com/SentineLabs/AlphaGolang |
| **Native IDA 9.2+ support** | Hex-Rays' official Go decompilation improvements | https://hex-rays.com/blog/stop-guessing-and-start-going |

### Ghidra Plugins

| Tool | Use | Link |
|------|------|------|
| **Ghidra + GoReSym output** | Export symbols with GoReSym, then import into Ghidra | Use together |
| **golang_loader_assist** | Ghidra Go load helper | Community script |

### Standalone Analysis Tools

| Tool | Use | Link |
|------|------|------|
| **gore** | Go reverse engineering library (underlies redress) | https://github.com/goretk/gore |
| **garble** | Go obfuscation tool (understand it to counter it) | https://github.com/burrowers/garble |

---

## Key Structures in Go Binaries

### pclntab (PC Line Table)

The most important structure in Go binaries; it contains:
- All function names and address mappings
- Source file paths
- Line number info
- Stack frame sizes

Even after symbols are stripped, pclntab usually still exists (the Go runtime depends on it).

```text
Locating it:
1. Search for the magic bytes: 0xFFFFFFF0 (Go 1.16+) or 0xFFFFFFFB (Go 1.18+)
2. Locate automatically with GoReSym
3. Parse automatically with the go_parser IDA plugin
```

### moduledata

Contains:
- pclntab pointer
- Type info table
- itab (interface table)
- Global variable info

### String Format

Go strings are not C-style null-terminated; they are `(pointer, length)` structs:

```text
C string:   "hello\0"
Go string:  struct { ptr *byte; len int } → ptr points to "hello" (no \0)
```

This makes IDA/Ghidra's default string detection miss many Go strings.

**Solutions:**
- Auto-detect Go strings with `go_parser`
- Export the string list with GoReSym
- Manually: find `runtime.stringtable` or locate via cross-references

---

## Hands-On Workflows

### Scenario 1: Unstripped Go Binary

```text
1. GoReSym -t -d -p binary > symbols.json
   → exports all function names, types, and source file paths
2. Load into IDA/Ghidra
3. Import GoReSym's symbol info
4. Filter out runtime.* and standard library functions; focus on user code
5. Start analysis from main.main
```

### Scenario 2: Stripped Go Binary

```text
1. GoReSym -t -d -p binary > symbols.json
   → even when stripped, pclntab is usually still present
2. If GoReSym fails → use redress
   redress -src binary    # recover source file paths
   redress -pkg binary    # recover package structure
   redress -type binary   # recover type info
3. Load into IDA with the go_parser plugin
4. Run go_parser for automatic recovery
5. Start from the recovered main.main
```

### Scenario 3: Garble-Obfuscated Go Binary

```text
Garble will:
- Randomize function names (main.main → main.a3f2b1c)
- Encrypt strings
- Remove file path info
- Obfuscate package names

Countermeasures:
1. GoResolver (CFG signature matching)
   → recover standard library function names via control-flow-graph similarity
2. GoStringUngarbler (string decryption)
   → automatically identify Garble's string encryption pattern and decrypt it
3. Dynamic analysis (Frida/dlv)
   → hook runtime functions to observe actual behavior
4. Comparative analysis
   → compile a Hello World with the same Go version and binary-diff the runtime portion
```

### Scenario 4: CGo Hybrid Builds

```text
1. Identify the CGo boundary (_cgo_* functions)
2. Recover the Go side with go_parser
3. Analyze the C side with regular IDA
4. Watch the bridging functions such as _cgo_topofstack and crosscall2
```

---

## Common Command Quick Reference

```bash
# GoReSym: export symbols
GoReSym -t -d -p binary > symbols.json
GoReSym -t -d -p binary -o ida_script.py  # generate an IDA script

# redress: analyze stripped binaries
redress -src binary          # source file paths
redress -pkg binary          # package structure
redress -type binary         # type info
redress -interface binary    # interface info
redress -filepath binary     # full file paths

# GoResolver: deobfuscate Garble
GoResolver -binary binary -output resolved.json

# GoStringUngarbler: decrypt Garble strings
GoStringUngarbler -i binary -o deobfuscated_binary

# Quickly determine the Go version
strings binary | grep "go1\."
GoReSym -p binary | grep "Version"
```

---

## Go Analysis Workflow in IDA

```text
1. Load the binary (select the correct architecture)
2. Wait for automatic analysis to finish
3. Run the go_parser plugin:
   - File → Script File → go_parser.py
   - or Edit → Plugins → Go Parser
4. The plugin will automatically:
   - parse pclntab
   - recover function names
   - mark Go strings
   - parse type info
5. Filter the view:
   - hide runtime.* functions
   - focus on main.* and third-party packages
6. Start reversing from main.main
```

---

## Common Pitfalls

| Pitfall | Description | Solution |
|------|------|------|
| Too many functions to review | Static linking yields 5000-50000 functions | Filter by package name; look only at main.* and business packages |
| Incomplete string detection | Go strings are not null-terminated | Recover with go_parser or GoReSym |
| Hard-to-read decompilation | Go's defer/goroutine/interface complicate pseudocode | IDA 9.2+ has improvements; or supplement with dynamic analysis |
| Garble obfuscation | Function names/strings fully randomized | GoResolver + GoStringUngarbler |
| Version differences | Different Go versions have different pclntab formats | GoReSym supports Go 1.2-1.23+ |
| CGo boundary | Go and C code mixed | Identify _cgo_* functions as the dividing line |

---

## Coordination with Other Skills

| Need | Use |
|------|--------|
| Deep IDA analysis of Go binaries | `ida-reverse/` + go_parser plugin |
| Ghidra analysis (free) | Ghidra + GoReSym symbol import |
| Quick recon | `radare2/` — `rabin2 -z` to view strings |
| Dynamic hooking | Frida (hook runtime functions) or dlv (native Go debugger) |
| Cross-version comparison | `binary-diff/` — migrate symbols from an older version to a newer one |
| Garble deobfuscation | GoResolver + GoStringUngarbler |
