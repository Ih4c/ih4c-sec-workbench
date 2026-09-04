---
name: ghidra-reverse
description: Use for free/open reverse engineering with Ghidra (headless or GUI), including decompile, cross-refs, and optional Ghidra MCP workflows when IDA is unavailable.
---

# Ghidra Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md`
2. `NOW`: Confirm the task needs **Ghidra** (no IDA / prefer open source / batch headless)
3. `NEXT`: Read `../tool-index.md` for ghidra / ghidra-mcp paths
4. `NEXT`: Missing tool → bootstrap `ghidra-mcp` (if the manifest supports it) or install Ghidra via the manual steps
5. `ACT`: Import the sample → auto-analyze → export decompilation of the key functions

## Applicable Scenarios

- Primary reverse engineering entry when no IDA license is available
- Batch headless analysis / decompilation in CI
- Ghidra scripting (Java / Python Jython / PyGhidra) automation
- Interop with `binary-diff` / `patch-diff-exploit` via ghidriff

## Division of Labor with IDA

| Need | Priority |
|------|------|
| Already have IDA MCP for deep dive | `ida-reverse/` |
| Open source / batch / teaching | **This skill** |
| CLI-only quick recon | `radare2/` |

## Workflow

### 1. Project and Auto-Analysis

```text
□ New Project → Import file → Analyze (default analyzers)
□ Record language/compiler identification results and base address
□ Mark entry point, export table, string xrefs
```

### 2. Key Functions

```text
□ Trace back from strings / imported APIs
□ Reconstruct algorithms in the Decompile window
□ Rename functions/variables; write Plate comments
□ Hand off to Frida/GDB when dynamic analysis is needed (reverse-engineering dynamic chapter)
```

### 3. Headless (batch)

```bash
# Example: the analyzeHeadless path varies by install; MUST get it from tool-index
analyzeHeadless /path/to/project Proj -import sample.bin -postScript ExportDecomp.py
```

### 4. MCP (if configured)

```text
□ Confirm the ghidra MCP port (commonly 8765; tool-index is authoritative)
□ Pull decompilation / xrefs via MCP tools; never guess the port
```

## Toolchain

| Tool | Purpose | Bootstrap |
|------|------|------|
| Ghidra | Primary decompilation tool | Manual release / package manager |
| ghidra-mcp | AI bridge | bootstrap capability name `ghidra-mcp` |
| ghidriff | Patch diffing | See `patch-diff-exploit` |

## References

- `references/ghidra-cheatsheet.md`
- `../ida-reverse/` `../radare2/` `../binary-diff/`

## Routing Context

**Upstream**: MASTER R22
**Downstream**: dynamic verification → Frida/GDB; exploitation → `pwn-chain`
**Peer**: `ida-reverse` (commercial deep dive)

## Task Completion Self-Check

- [ ] Based on real Ghidra/tool-index paths?
- [ ] Function addresses noted and renamed?
- [ ] Reproducible steps present?
- [ ] Checklist / journal?
