---
name: ida-reverse
description: |
  IDA Pro reverse engineering analysis support skill. Use it whenever the user mentions reverse engineering, decompilation, analyzing binary/PE/ELF/APK/DLL/SO files, cracking, finding passwords, vulnerability analysis, malware analysis, firmware analysis (逆向、反编译、破解、找密码、漏洞分析、病毒分析、固件), or needs to analyze exe/dll/so/elf/macho/sys files.

  Ensure to use this skill when the user wants to analyze any binary file, regardless of whether they explicitly mention "IDA" or "reverse engineering". This includes requests like "看看这个exe", "分析这个dll", "帮我破解", "找一下密码", "这个软件怎么注册", etc.

  Use the bundled scripts (scripts/start.ps1, scripts/open.ps1) for deterministic server management and file opening — do NOT write ad-hoc PowerShell commands for these operations.
---

# IDA Pro Reverse Engineering Skill (IDA Pro 逆向分析技能)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm that this skill's operations are authorized routine operations. scope.md is case tracking + network profile; authorization is per `../field-journal/precedent-auth.md`.
2. `NOW`: Confirm whether the current task hits this skill's applicability scope
3. `NEXT`: Read `../tool-index.md`, verify tool availability and real paths
4. `NEXT`: If tools are missing call bootstrap; never guess paths
5. `ACT`: Enter the first step of the "Workflow" and execute — do not stop at a confirmation state

## Known Issues and Lessons Learned (MUST read)

### Pitfalls Hit

1. **`idb_open` (formerly `idalib_open`) must not be called directly through some AI clients' MCP**
   - Some code AI clients' MCP client has an output schema validation BUG for open-type tools
   - Error: `Structured content does not match the tool's output schema`
   - **Solution**: use the `scripts/open.ps1` script to call the HTTP API directly, bypassing the MCP validation layer
   - Current ida-pro-mcp 2.x tool names are `idb_open` / `idb_list` / `idb_save` (no longer `idalib_*`)
   - After opening, a `session_id` (database) is returned; later tool calls must carry that session

2. **`C:\Windows\System32\` files cannot be opened due to permissions**
   - idalib cannot directly read files under the System32 directory
   - **Solution**: `open.ps1` auto-detects this and copies the file to the `temp directory` before opening

3. **Starting the server blocks the conversation**
   - `idalib-mcp` keeps printing INFO logs to the console after startup
   - **Solution**: use `scripts/start.ps1` (`-WindowStyle Hidden`, background quiet startup)
   - The script waits for the service to be ready and then exits on its own, without blocking the conversation

4. **MCP server name cannot contain hyphens**
   - Previously `ida-pro-mcp` was used as the server name, which could cause tool registration problems
   - **Current config**: server name `idapro`, tool prefix `idapro_*`

5. **Remote HTTP vs Local Stdio**
   - `type:"local"` (stdio) mode: `idalib_open` has the same schema validation problem
   - `type:"remote"` (HTTP) mode: you can open files directly via the script first, then use the MCP tools
   - **Current approach**: Remote HTTP mode

6. **PR #389 fixed part of the schema issues**
   - Author mrexodia merged the fix via PR #389 after issue #388
   - It fixed the structuredContent schema in HTTP mode, but validation on some code AI clients' side still has issues
   - The latest `main` branch version is installed

7. **idalib timeout leaves orphan worker processes holding file locks**
   - After the first `open.ps1` timeout, idalib's python worker child process can become an orphan, holding onto `.id0`/`.id1`/`.nam`
   - Any later tool call or manual drag into IDA GUI reports "insufficient permissions"
   - **Forbidden** `taskkill /F /T` on the process tree — `/T` also kills the GUI `ida.exe` child processes
   - **Solution**: `start.ps1` replaces the managed supervisor only when the port has no listener, or when `tools/list` returns fast but `py_eval` is missing (old supervisor); RPC timeout with 13337 still listening is treated as busy and not killed
   - **Fallback**: when `open.ps1` detects the old database is locked, it copies it to Temp with a GUID prefix and opens automatically

8. **Opening with auto-analysis looks like a hang**
   - `idalib_open(run_auto_analysis=true)` can take a long time to respond, but the backend is still opening and analyzing
   - Previously the user side saw "PowerShell with no output", easily misjudged as the script hanging
   - **Current solution**: `open.ps1` gained `-TimeoutSeconds`, and switched to background request + foreground polling + periodic progress output
   - When polling finds the session ready it returns `OK:filename:session_id` early; on timeout it returns `ERR:open_timeout_xxs`

9. **HTTP MCP silently exits after logon**
   - Cursor/Claude's `type: http` does not launch the process for you; the old scheduled task ran only once at logon
   - `pythonw` has no console, so the Application log is empty on crash too
   - **Solution**: `start.ps1` reuses when healthy by default; `watchdog.ps1` patrols every minute; logs under `%LOCALAPPDATA%\reverse-skill\ida-mcp\`
   - Install: `scripts/install-autostart.ps1`. If Cursor's port is not up at startup, you still need one manual refresh in the MCP panel

### Workflow Principles

| Step | What to Do | What to Use |
|------|--------|--------|
| 1 | Ensure the HTTP server is running | `scripts/start.ps1` (no args) |
| 2 | Open the target binary | `scripts/open.ps1 -Path "xxx.exe"` |
| 3 | Use the MCP analysis tools | Call `idapro_*` / HTTP tools directly (~65, depending on version) |
| 4 | Analysis done | Tools remain available |

## Script Resources

### start.ps1 — start the MCP HTTP server

Path: `scripts/start.ps1`

- Auto-resolves `IDADIR` (environment variable / portable desktop path / common install paths)
- Prefers IDA-bundled `Python314\python.exe -m ida_pro_mcp.idalib_supervisor`
- First probes `http://127.0.0.1:13337/mcp` by default; if healthy prints `OK:<n>:reuse` and exits
- 13337 listening but `tools/list` times out → `WARN:busy` / `OK:busy:reuse`, **no kill** (the supervisor is single-threaded and cannot answer while opening a database)
- Replaces the managed supervisor only when the port has no listener, or when it answers fast but lacks `py_eval`; **never kills `ida.exe`, no `taskkill /T`**
- When the GUI holds 13337, prints `WARN:gui_busy` and exits, does not start another supervisor
- On success prints `OK:<tool count>` (~66 currently); on failure prints `ERR:timeout`
- Supervisor log: `%LOCALAPPDATA%\reverse-skill\ida-mcp\supervisor.log`
- The server runs in the background and does not block the conversation

**Invocation**:
```
powershell -File "<skill-root>\ida-reverse\scripts\start.ps1"
```

### watchdog.ps1 / install-autostart.ps1 — keep-alive

- `watchdog.ps1`: probes 13337; if healthy `OK:<n>:reuse`, only calls `start.ps1` when it is down
- `install-autostart.ps1`: registers the scheduled task `reverse-skill-ida-mcp` (at logon + every minute)
- Log: `%LOCALAPPDATA%\reverse-skill\ida-mcp\watchdog.log`

### open.ps1 — open a binary file

Path: `scripts/open.ps1`

- Calls `idb_open` directly through the HTTP API, bypassing MCP schema validation
- Auto-detects System32 paths and copies to a temp directory
- Auto-cleans same-named old database files (`.id0`/`.id1`/`.nam`/`.til`/`.i64`)
- Auto-degrades when the old database is locked: copies to Temp with a GUID prefix and opens, no error
- Runs the open request in the background to avoid long synchronous waits making the script unresponsive
- Supports `-TimeoutSeconds`; after the timeout returns `ERR:open_timeout_xxs`, never hangs forever
- Prints `INFO:opening:elapsed/timeout seconds` every 10 seconds so you can tell analysis is still running
- On success prints `OK:filename:session_id`; appends `(temp copy)` when degraded
- On failure auto-retries via the Temp copy

**Invocation**:
```
powershell -File "<skill-root>\ida-reverse\scripts\open.ps1" -Path "C:\path\to\file.exe"
```

**Optional parameters**:
```
# Specify SessionId
powershell -File "scripts\open.ps1" -Path "file.exe" -SessionId "my_session"

# Skip auto-analysis (recommended for large files)
powershell -File "scripts\open.ps1" -Path "large.exe" -NoAutoAnalysis

# Set a timeout, to avoid long no-response when using auto-analysis
powershell -File "scripts\open.ps1" -Path "file.exe" -TimeoutSeconds 600
```

**Output conventions**:
```
# Analysis in progress (printed every 10 seconds)
INFO:opening:11/600s

# Opened successfully
OK:sample.exe:abcd1234

# Opened successfully, but degraded to a Temp copy due to a file lock
OK:1234abcd-sample.exe:abcd1234 (temp copy)

# Reached the timeout limit
ERR:open_timeout_600s
```

**Notes from real measurements**:
- `Snipaste.exe` with auto-analysis took ~`324s` to return success — that is "analysis takes long", not "script deadlocked"
- So for GUI programs or more complex samples, prefer explicitly setting `-TimeoutSeconds 600`

## Core Tool List

### Overview Analysis (first step)
- `idapro_survey_binary(detail_level="minimal")` — quick profile: function count, strings, segments, entry point, import classification (crypto/network/file IO)
- `idapro_list_funcs(queries)` — list functions (paged, filtered by name)
- `idapro_list_globals(queries)` — list global variables
- `idapro_entity_query(kind, filter)` — unified query: functions/globals/imports/strings/names

### Decompilation and Disassembly
- `idapro_decompile(addr)` — decompile to pseudocode
- `idapro_disasm(addr, max_instructions=N)` — disassemble
- `idapro_analyze_function(addr, include_asm=false)` — combined analysis (pseudocode+strings+constants+callers+callees+blocks)
- `idapro_func_profile(queries)` — function profile metrics

### Cross References and Data Flow
- `idapro_xrefs_to(addrs)` — see who references the target address
- `idapro_xref_query(addr, direction)` — advanced xref query (direction/type filters)
- `idapro_callees(addrs)` — child function list
- `idapro_callgraph(roots, max_depth)` — call graph
- `idapro_trace_data_flow(addr, direction, max_depth)` — data flow tracing (forward/backward)

### Search
- `idapro_find_regex(pattern, limit)` — regex string search
- `idapro_search_text(pattern)` — search text in the disassembly listing
- `idapro_find_bytes(patterns, limit)` — byte pattern search (supports ?? wildcards)
- `idapro_find(type, targets)` — advanced search (immediates/strings/references)

### Memory and Data
- `idapro_get_bytes(addrs)` — read raw bytes
- `idapro_get_string(addrs)` — read strings
- `idapro_get_int(queries)` — read integer values
- `idapro_get_global_value(queries)` — read global variable values
- `idapro_read_struct(queries)` — read struct field values
- `idapro_search_structs(filter)` — search structs

### Modification Operations
- `idapro_set_comments(items)` — add comments (synced both ways between disasm and decompile)
- `idapro_append_comments(items)` — append comments
- `idapro_rename(batch)` — batch rename (functions/globals/locals/stack variables)
- `idapro_patch_asm(items)` — patch assembly instructions
- `idapro_patch(patches)` — patch bytes
- `idapro_define_func(items)` — define a function
- `idapro_undefine(items)` — undefine
- `idapro_define_code(items)` — turn bytes into code

### Type System
- `idapro_declare_type(decls)` — declare C structs/enums/unions
- `idapro_set_type(edits)` — apply types to functions/globals/locals
- `idapro_infer_types(addrs)` — infer types
- `idapro_type_query(queries)` — query declared types
- `idapro_type_inspect(queries)` — inspect type details

### Stack Frames
- `idapro_stack_frame(addrs)` — view stack frame variables
- `idapro_declare_stack(items)` — declare stack variables
- `idapro_delete_stack(items)` — delete stack variables

### Signatures
- `idapro_make_signature(addrs)` — generate a unique byte signature for an address
- `idapro_make_signature_for_function(addrs)` — generate a signature for a function
- `idapro_find_xref_signatures(addrs)` — generate signatures for code referencing an address

### Debugger (requires ?ext=dbg)
- `idapro_open_file(file_path)` — open a file in a GUI IDA instance
- Debugger tools are hidden by default; enable with the URL parameter `?ext=dbg`

### Session Management (ida-pro-mcp 2.x)
- `idapro_idb_open` / HTTP `idb_open` — ⚠️ recommended to open with `open.ps1`
- `idapro_idb_list` / HTTP `idb_list` — list all sessions
- `idapro_idb_save` / HTTP `idb_save` — save the database
- Most analysis tools need a `database=<session_id>` parameter (the session output by open.ps1)

### Misc
- `idapro_int_convert(inputs)` — base conversion (**MUST use this, never compute bases yourself!**)
- `idapro_export_funcs(addrs, format)` — export functions (json/c_header/prototypes)
- `idapro_py_eval(code)` — execute Python in the IDA context
- `idapro_server_health()` — server health check
- `idapro_server_warmup()` — warm up subsystems (string cache, Hex-Rays, etc.)

## Full Reverse Engineering Workflow

### Step 1: Start the Server

**Path A — Headless idalib (requires a valid license)**
```
powershell -File "scripts/start.ps1"
```
Output `OK:<tool count>` (~65 currently) means ready.

**Path B — GUI + plugin (when the idalib license fails or interactive analysis is needed)**
```
powershell -File "scripts/start-gui.ps1" -Path "C:\target.exe"
```
Or double-click the portable `Launch-IDA-Pro.cmd` and open the sample in IDA.

Once `[MCP] ... port=13337` appears in the Output window, the MCP tools are usable.

Generic integration steps are in `LOCAL-SETUP.md`.

### Step 2: Open the File

Headless:
```
powershell -File "scripts/open.ps1" -Path "C:\target.exe" -TimeoutSeconds 600
```
`OK:filename:session_id` means success (a trailing `(temp copy)` means it auto-degraded to a temp copy).

If `ERR:idalib_license:...` appears, switch to Path B (GUI mode) instead of retrying open.ps1 repeatedly.

GUI mode: just Open the sample directly in IDA; no open.ps1 needed.

### Step 3: Global Overview (including the import-table hard gate)
```
idapro_survey_binary(detail_level="minimal")
```
Focus on:
- Architecture (x86/x64/ARM)
- Entry point (main/WinMain/DllMain)
- Interesting strings (URLs, paths, error messages)
- **Import classification (MUST)**: crypto functions / network APIs / file operations / process injection / registry — must be recorded as Evidence (suggested id: `E-imports`), via `idapro_entity_query(kind="imports")` or the imports section of the survey output
- **DLL/SYS**: exports table alongside imports table (Evidence `E-exports`)
- **.NET**: when there is no traditional IAT, use a summary of modules/metadata/managed references as the equivalent anchor in the E-imports semantic slot
- **Clean import table**: note the suspicion of dynamic loading, driving dynamic API breakpoint verification
- Hot functions (high-xref functions are usually key logic)

**Hard gate**: before the imports view/classification summary (or a legitimate equivalent anchor) is written to Evidence, MUST NOT proceed to draw conclusions from Step 4 deep dives, MUST NOT claim the survey is complete. If the import table is empty or the query fails, the failure symptom MUST still be recorded. On packed-binary IAT repair failure, MUST record `E-iat-repair-fail` and switch to dynamic debugging to catch APIs; endless static grinding is forbidden. When the user asks to redo the import-table/IAT check, MUST redo the named step (if blocked, the feasibility gate applies: state the blocker + confirm; if forced, mark quality=unreadable); substituting an unrelated step is forbidden.

### Step 4: Dive Into Key Functions
```
idapro_analyze_function(addr="key function name")
```
Or:
```
idapro_decompile(addr="function name")
idapro_disasm(addr="function name", max_instructions=50)
```

### Step 5: Data Flow and Cross References
```
idapro_xrefs_to(addrs="key address/string")
idapro_callgraph(roots=["key function"], max_depth=3)
idapro_trace_data_flow(addr="key address", direction="backward", max_depth=5)
```

### Step 6: Annotate and Refine
```
idapro_set_comments(items=[{"addr": "0x140001000", "comment": "your understanding"}])
idapro_rename(batch={"func": [{"addr": "function address", "name": "meaningful name"}]})
```

### Step 7: Output Report
After analysis, generate a `report.md` recording findings and steps.

## Prompt Engineering Guidelines

1. **Never compute bases by hand** — whenever you need to convert numbers, use `idapro_int_convert`
2. **survey first, then dive** — look at the overview first, then analyze targets
3. **Keep adding comments and renames** — continuously update function and variable names during analysis to improve the accuracy of later analysis
4. **Follow cross references** — when you find interesting data/strings, use `xrefs_to` to see who references them
5. **Obfuscated code** — first do preprocessing such as string decryption, import-hash removal, and control-flow-flattening removal
6. **C++ STL code** — after identifying library functions with FLIRT/Lumina, then analyze the business logic
7. **No brute-forcing** — analysis should derive solutions from the disassembly; use simple Python only to assist computation
8. **"No database bound"** — no binary is open yet; run `open.ps1` first
9. **"Failed to open database"** — old database files may be locked; `open.ps1` auto-degrades to a Temp copy (output carries a `(temp copy)` marker)
10. **Opening GUI/complex samples with auto-analysis** — add `-TimeoutSeconds 600` by default; do not misjudge a long `INFO:opening:...` as a script hang

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master control), `routing.md`
**Upstream alternative**: `radare2/` (if you don't want to start IDA, quick r2 recon first)
**Downstream exits**:
- Need Frida dynamic verification → `reverse-engineering/tools-dynamic.md`
- Need symbolic execution/angr → `reverse-engineering/tools-dynamic.md`
- Need generic reverse methodology → `reverse-engineering/SKILL.md`

**Peer modules**: `radare2/` (alternative when IDA is unavailable)

---

## On-Demand Bootstrap (按需自举)

This skill's entry scripts are wired into the unified bootstrap system.

### Automation Capability Boundaries

| Tool | Auto-installable | Install Method | Notes |
|------|-----------|---------|------|
| idalib-mcp | Yes | pip install (from GitHub) | Auto-installs when `start.ps1` finds it missing |
| IDA Pro itself | No | Commercial software, manual install required | Point the `IDADIR` environment variable at the install directory |

### Install Steps (verified)

```cmd
# 1. Set the IDA path (replace with your actual IDA install directory)
setx IDADIR "<your IDA install directory>"

# 2. Install ida-pro-mcp from GitHub (the PyPI ida-mcp is a different project, don't install the wrong one!)
pip install git+https://github.com/mrexodia/ida-pro-mcp.git

# 3. Install the IDA plugin (choose Streamable HTTP + Global + select all clients)
ida-pro-mcp --install

# 4. Restart IDA Pro, open the target file
# The plugin auto-listens on 127.0.0.1:13337

# 5. Verify
ida-pro-mcp --config
```

> ⚠️ **Note**: the `ida-mcp` package on PyPI (author jtsylve) is a different project, not the one we need.
> Must install `mrexodia/ida-pro-mcp` from GitHub.

### Bootstrap Trigger Points

- `scripts/start.ps1`: auto-invokes `bootstrap-reverse.ps1` when `idalib-mcp` is missing
- MCP registration: bootstrap auto-writes `idapro` into the Claude MCP config

### Prerequisites

- IDA Pro installed and the `IDADIR` environment variable set (or the in-script default paths are correct)
- Recommended: use `ida-pro-mcp` inside IDA's bundled Python314 (already built into the portable version)
- Typical local config:
  - User env `IDADIR` → IDA install directory (contains `ida.exe`)
  - Optional `~\Tools\bin\idalib-mcp.cmd` / `ida-pro-mcp.cmd` wrappers
  - Client MCP server name: keep only `idapro` → `http://127.0.0.1:13337/mcp`


## Task Completion Self-Check (MUST pass before claiming done)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Is survey/imports written to Evidence (E-imports or equivalent)? Does the DLL/SYS include E-exports? Is an IAT failure recorded as E-iat-repair-fail?
- [ ] If the user asked to redo the import table/IAT, did I redo the same step?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
