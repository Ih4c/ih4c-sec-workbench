---
name: js-reverse
description: Use when doing front-end JavaScript reverse engineering with js-reverse-mcp (前端 JavaScript 逆向); covers signature-chain location (签名链路定位), page observation and forensics (页面观察取证), runtime sampling (运行时采样), local environment patching and reproduction (本地补环境复现), and evidence-based output (证据化输出). Prefer the js-reverse_* tools available in the current environment; when a stronger browser/CDP/Hook surface is needed, coordinate with jshookmcp.
---

# MCP Front-End JS Reverse Engineering Operating Standard

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm that this skill's operations are already-authorized routine operations
2. `NOW`: Confirm whether the current task falls within this skill's scope of application
3. `NEXT`: Read `../tool-index.md`, verify tool availability and actual paths
4. `NEXT`: If tools are missing, call bootstrap; never guess paths
5. `ACT`: Enter step 1 of the workflow below and execute it; do not stop at a confirmation state

## Scope of Application

Prefer this skill when the task belongs to one of the following scenarios:

- Locating API signatures, encrypted parameters, and risk-control fields
- Observing page request chains and script origins
- Capturing function inputs and return values at runtime
- Tracing the trigger point of a particular XHR/Fetch/WebSocket
- Bringing page evidence back to Node for local reproduction and environment patching

If the target is a binary, APK, PE, ELF, DLL, or SO, switch to `ida-reverse`, `radare2`, or `reverse-engineering` instead.

## Default Tool Mapping in the Current Environment

This skill does not assume bare tool names exist; it binds by default to the `js-reverse_*` tools available in the current client environment.

If the current task explicitly mentions `jshookmcp`, `JS hook`, `CDP`, browser breakpoints, network interception, SourceMap, or AST deobfuscation, it still routes through this skill; only the underlying MCP surface switches to `jshookmcp`, rather than treating it as a new top-level entry point.

Prerequisite: `jshookmcp` is not a local bare command-line tool; it is an MCP server that must first be downloaded, explicitly registered, and enabled. Its tool surfaces are only truly callable after it has been connected and enabled in the MCP configuration of the chosen client (Claude, Codex, etc.).

Common mapping:

- `list_scripts` -> `js-reverse_list_scripts`
- `get_script_source` -> `js-reverse_get_script_source`
- `search_in_sources` -> `js-reverse_search_in_sources`
- `break_on_xhr` -> `js-reverse_break_on_xhr`
- `evaluate_script` -> `js-reverse_evaluate_script`
- `get_paused_info` -> `js-reverse_get_paused_info`
- `set_breakpoint_on_text` -> `js-reverse_set_breakpoint_on_text`
- `list_network_requests` -> `js-reverse_list_network_requests`
- `get_request_initiator` -> `js-reverse_get_request_initiator`
- `get_websocket_messages` -> `js-reverse_get_websocket_messages`
- `take_screenshot` -> `js-reverse_take_screenshot`
- `new_page` -> `js-reverse_new_page`
- `navigate_page` -> `js-reverse_navigate_page`
- `select_page` -> `js-reverse_select_page`
- `select_frame` -> `js-reverse_select_frame`
- `pause/resume` -> `js-reverse_pause_or_resume`

If the tool-name prefix changes in the future, update this section first; never guess ad hoc during execution.

### The Position of jshookmcp

- Role: an enhanced execution surface for `js-reverse`, not an independent master controller
- Suitable for: browser automation, CDP debugging, JS Hook, network interception, SourceMap reconstruction, AST-assisted understanding
- Call prerequisite: download `@jshookmcp/jshook` and register it in the MCP client configuration first, then make sure that server is enabled
- Suggested entry: still execute as `Observe → Capture → Rebuild`, only that in the `Observe/Capture` phases you call jshookmcp's browser and Hook capabilities first
- Relationship with anything-analyzer: both can do browser/network-side forensics; anything-analyzer leans more toward packet capture and HTTP analysis, while jshookmcp leans more toward JS runtime, CDP, Hook, and source understanding

## Core Principles

- `Observe-first`
- `Hook-preferred`
- `Breakpoint-last`
- `Rebuild-oriented`
- `Evidence-first`

Observe the page first, then sample minimally, then do local environment patching — never skip forensics and guess the environment directly.

## Five-Phase Workflow

### 1. Observe

Goal: first confirm the target request, relevant scripts, and candidate functions; do not guess the environment.

Default actions:

- Use `js-reverse_new_page` or `js-reverse_navigate_page` to open the target page
- Use `js-reverse_list_network_requests` to find the target request
- Use `js-reverse_get_request_initiator` to trace back the call source
- Use `js-reverse_list_scripts`, `js-reverse_search_in_sources` to narrow down the script scope

Required outputs:

- Target request URL or its signature
- initiator leads
- Suspicious script URLs
- Initial task record

### 2. Capture

Goal: perform minimally invasive sampling of the target request to obtain parameter samples, call order, and runtime evidence.

Rules:

- Prefer `js-reverse_break_on_xhr`
- Prefer `js-reverse_evaluate_script` for lightweight runtime observation
- After a hit, look at `js-reverse_get_paused_info` first
- Only if needed, then use `js-reverse_set_breakpoint_on_text`

### 3. Rebuild

Goal: organize page evidence into locally iterable Node reproduction material.

Rules:

- Local environment patching must be based on page observation evidence
- Speculative patching of `window/document/navigator/crypto/storage` is not allowed
- Record only one minimal causal patch decision at a time

### 4. Patch

Goal: drive environment patching by errors and first divergence until the local script stably produces the target parameters.

Rules:

- Look at what is missing first, then patch it
- Make only one minimal patch decision at a time
- Re-test immediately after every patch
- Write every patch into the task record

### 5. DeepDive

Goal: after the local run works, do deobfuscation, control-flow recovery, and business-logic purification.

Rules:

- If the current task only needs to produce a signature, this phase can be downgraded
- If the algorithm chain will be reused long-term, this phase is mandatory
- Issue #65 obfuscation bypass (U–AV §4): JSVMP (AD) → `E-js-vmp`; CFF + string array (AE) → `E-js-deobf`; DevTools/debugger anti-debugging (AF) → `E-js-anti-debug`. Full trigger table in `../reverse-engineering/references/nonpe-format-cookbook.md`; AST details still via `references/ast-deobfuscation.md`

## Execution Requirements

- All important steps must be written into the local task artifact
- If you cannot explain why a tool is being called, do not call it
- Prefer the ready-made MCP capabilities of `js-reverse_*` or jshookmcp for direct forensics; do not write scripts to reinvent capabilities first
- On failure, fall back per `references/fallbacks.md`
- Output follows `references/output-contract.md`

## Required Reading References

- Automation entry: `references/automation-entry.md`
- Parameter defaults: `references/tool-defaults.md`
- Task input template: `references/task-input-template.md`
- MCP-specific task orchestration: `references/mcp-task-template.md`
- Task artifacts: `references/task-artifacts.md`
- Local reproduction: `references/local-rebuild.md`
- Environment patching: `references/env-patching.md`
- Node reproduction: `references/node-env-rebuild.md`
- Instrumentation: `references/instrumentation.md`
- AST deobfuscation: `references/ast-deobfuscation.md`
- Non-PE/JS obfuscation cookbook U–AV: `../reverse-engineering/references/nonpe-format-cookbook.md` (AD/AE/AF)
- Fallbacks: `references/fallbacks.md`
- Output contract: `references/output-contract.md`

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master controller), `routing.md`
**Upstream alternatives**:
- anything-analyzer MCP (port 23816) browser tools can substitute or supplement
- jshookmcp can serve as a stronger browser/CDP/Hook/Network/SourceMap/AST execution surface
- `reverse-engineering/SKILL.md` (if the target is not front-end JS)

**Downstream exits**:
- Environment patching needed → `references/env-patching.md`
- Local reproduction needed → `references/local-rebuild.md` / `references/node-env-rebuild.md`
- Deobfuscation needed → `references/ast-deobfuscation.md`
- Fall back when stuck → `references/fallbacks.md`

**Peer modules**: anything-analyzer MCP (its browser automation and HTTP capture capabilities can complement this skill)

---

## On-Demand Bootstrap

The MCP capabilities this skill depends on can be installed through the unified bootstrap system; MCP client registration must explicitly select the target, and by default no client-wide configuration is ever written.

### Automation Capability Boundaries

| Capability | Auto-registrable | Method | Notes |
|------|-----------|------|------|
| jshookmcp | yes | npm-mcp (launched via npx) | Registered after explicitly choosing Claude / Codex / Both |
| anything-analyzer | yes | local-http-mcp | Service can be auto-started; client registration must be explicit |
| Node.js | yes | winget install | Runtime dependency |

### Bootstrap Methods

```powershell
# Install and register jshookmcp; Codex can be replaced with Claude or Both
powershell -File "<skill-root>\scripts\bootstrap-reverse.ps1" -Capability @('jshookmcp') -McpHostTarget Codex

# Register and start anything-analyzer
powershell -File "<skill-root>\scripts\bootstrap-reverse.ps1" -Capability @('anything-analyzer') -StartServices -McpHostTarget Codex
```

### Notes

- After `jshookmcp` is registered, the MCP server still has to be **enabled** in the AI client before its tools can be called
- When `-McpHostTarget` is not passed, the capability is only installed/prepared and registration-required is returned; Claude or Codex configuration is not modified
- `anything-analyzer` needs pnpm and the project source; bootstrap clones it and installs dependencies automatically
- If Node.js is not installed, bootstrap first installs Node.js 22 via winget

<br><br>## Task Completion Self-Check (MUST pass before claiming completion)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
