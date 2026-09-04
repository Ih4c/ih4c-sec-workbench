---
name: macos-reverse
description: Use for authorized macOS and Mach-O reverse engineering including codesign, Objective-C/Swift recovery, endpoint security surfaces, and Apple platform malware analysis.
---

# macOS / Mach-O Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md`
2. `NOW`: Confirm the target is macOS/Mach-O/App bundle (iOS IPA → `mobile-reverse/`)
3. `NEXT`: tool-index; jtool2/lldb, etc.
4. `ACT`: signature and load information → static → dynamic (lldb/Frida)

## Applicable Scenarios

- Mach-O executables / dylibs / frameworks
- .app bundles, LaunchAgents/Daemons
- Objective-C / Swift symbols and runtime
- Notarization/codesigning, Hardened Runtime, TCC-related behavior analysis
- Static/dynamic analysis of macOS malware (in coordination with malware-analysis)

## Workflow

### 1. Bundle and Signature

```bash
file target
codesign -dv --verbose=4 target
spctl -a -vv target 2>&1
otool -L target
```

### 2. Static

```text
□ class-dump / swift-demangle / Hopper / Ghidra / IDA
□ Strings and XPC service names, TCC-sensitive APIs
□ LC_LOAD_dylib dependencies and rpath
```

### 3. Dynamic

```text
□ lldb / Frida
□ fs_usage / log stream observation
□ Network: coordinate with protocol-reverse or a proxy
```

## Tool Chain

| Tool | Purpose |
|------|------|
| otool / nm / codesign | Bundled with the system |
| Hopper / Ghidra / IDA | Decompilation |
| class-dump / dsdump | ObjC |
| Frida / lldb | Dynamic |
| jtool2 | Mach-O |

## References

- `references/macho-triage.md`
- `../mobile-reverse/` (iOS), `../ghidra-reverse/`, `../malware-analysis/`

## Routing Context

**Upstream**: MASTER R31
**Downstream**: iOS → mobile-reverse; generic samples → malware-analysis

## Task Completion Self-Check

- [ ] Was the codesign/Hardened Runtime status recorded?
- [ ] Are there address-level/symbol-level conclusions?
- [ ] Checklist?
