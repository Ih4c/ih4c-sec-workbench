# IDA ↔ reverse-skill Integration (Portable) (IDA ↔ reverse-skill 对接)

This page contains generic steps, no absolute paths of any specific machine. The local readiness report lives in `LOCAL-READINESS.md` at the repository root (gitignored).

## Target Shape

| Item | Convention |
|----|------|
| IDA install directory | Environment variable `IDADIR` (contains `ida.exe` or `ida.dll`) |
| HTTP MCP | `http://127.0.0.1:13337/mcp` |
| Client server name | Only **`idapro`** (do not also register `ida-pro-mcp`) |
| Startup | `scripts/start.ps1` (`--unsafe`, no `?ext=dbg`) |
| Opening databases | Prefer `scripts/open.ps1` for large files; do not call `idb_open` directly through some clients |

Two MCP names pointing at the same 13337 register the tools twice and fight the idalib worker for the port.

## Installation

```powershell
setx IDADIR "<your IDA install directory>"

# Must use mrexodia/ida-pro-mcp, do not install the PyPI ida-mcp
python -m pip install "git+https://github.com/mrexodia/ida-pro-mcp.git"

# Activate idalib (adjust path to your local IDA)
python "<IDADIR>\idalib\python\py-activate-idalib.py" -d "<IDADIR>"

# Install plugin + client config
python -m ida_pro_mcp --install --transport streamable-http --scope global
```

## Startup and Keep-Alive

MCP entries of `type: http` do not launch the process for you. When 13337 is not listening, all clients report errors.

| Script | Role |
|------|------|
| `scripts/start.ps1` | If healthy prints `OK:<n>:reuse`; a listening port with RPC timeouts is treated as busy, not killed; replaces the managed supervisor only when nobody is listening or `py_eval` is missing; never kills `ida.exe` |
| `scripts/watchdog.ps1` | Patrols every minute; reuses when busy/healthy; calls `start.ps1` only when down/stale |
| `scripts/install-autostart.ps1` | Registers the scheduled task `reverse-skill-ida-mcp` (at logon + every minute) |
| `scripts/start-gui.ps1` | Opens the GUI plugin when the idalib license fails |
| `scripts/open.ps1` | Calls `idb_open` over HTTP directly, bypassing some clients' schema validation |

Logs: `%LOCALAPPDATA%\reverse-skill\ida-mcp\supervisor.log` and `watchdog.log`.

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "skills\ida-reverse\scripts\start.ps1"
powershell -NoProfile -ExecutionPolicy Bypass -File "skills\ida-reverse\scripts\open.ps1" -Path "C:\path\to\target.exe" -TimeoutSeconds 600
powershell -NoProfile -ExecutionPolicy Bypass -File "skills\ida-reverse\scripts\install-autostart.ps1"
```

When the GUI holds 13337 but does not answer for a while, `start.ps1` prints `WARN:gui_busy` and exits, so an IDA currently analyzing is not killed.

## Clients

All point to Streamable HTTP: `http://127.0.0.1:13337/mcp`, server name `idapro`.

After changing config you must start a new session. If Cursor's port is not listening at startup, bringing the service up later will **not auto-reconnect** — you need to manually refresh in the MCP panel.

## Known Caveats

1. System32 files: `open.ps1` copies them to a temp path (output marked `(temp copy)`)
2. Do not call `idb_open` directly through some clients' MCP
3. `start.ps1` prefers `python -m ida_pro_mcp.idalib_supervisor`, more stable than the `.cmd` wrapper
4. When both a formal install and a desktop portable copy exist, `IDADIR` is authoritative
5. Do not add `?ext=dbg` (debugger tools are not exposed by default)
