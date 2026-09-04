# reverse-skill In-Package Security Audit (Executable Surface)

> Date: 2026-08-02
> Scope: executable scripts and bootstrap manifests under `skills/**/scripts`, `skills/scripts`, `kali/scripts`, `burp-mcp-full`  
> **Excludes**: `src-hunter` / payloader and other **educational payload documents** (their DROP/injection samples are methodology, not auto-executed)

## Verdict (Overall Assessment)

| Level | Finding |
|------|------|
| **Backdoor / deliberate database wipe / disk format** | **Not found** |
| **Piped download-and-execute (curl\|sh / IEX DownloadString)** | **Not found** |
| **Hardcoded cloud keys / private keys** | **Not found** (`sk-` / `BEGIN RSA` in docs are detection examples) |
| **Residual supply-chain risk** | **Partially hardened (medium-low → low)**: pinned away from `@latest`; GitHub downloads support **manifest SHA256 + API digest** |

**Overall: no implanted backdoors or "one-click database wipe" logic currently found on the executable skill-script surface; all dangerous deletions are confined to tool-reinstall temp directories / case output directories.**

### 2026-07-18 Hardening (This Commit)

| Item | Action |
|----|------|
| jshookmcp | `@latest` → `@0.3.4` |
| pentestswarm | `@latest` / docker `:latest` → `@v0.1.0` / `:v0.1.0` |
| jadx | pinned `v1.5.6` + `assetSha256` |
| apktool | pinned `v3.0.2` + `assetSha256` |
| bootstrap PS/sh | `Assert-DownloadedFileIntegrity` / `verify_sha256` after download; manifest hash first, GitHub `digest` second; delete the file and abort on failure |
| Releases without pinned hashes | Still installable, but **WARN** and print the actual sha256 |

### 2026-08-02 Security Fixes

| Item | Fix |
|----|------|
| Kali quick setup | Resolve the sudo user's home with `getent`, removed `eval` |
| Frida process listing | Use a `frida-ps` argument array, removed inline Python code concatenation |
| Burp MCP token | Atomic replacement via a restricted temp file; POSIX file permission fixed to `0600` |
| Burp MCP bridge | Parse by MCP newline-delimited messages; reconnect on demand after Burp starts |
| Anything Analyzer MCP | Bootstrap enables bearer auth by default and registers credentials through an optional host adapter |
| IDA MCP startup | Terminate old processes one by one to avoid multi-PID argument expansion errors |

## Scan Methodology

Searched executable extensions (`.ps1` / `.sh` / `.py` / `.js` / `.java`) for:

- `Invoke-Expression` / `IEX` / `FromBase64String` / `DownloadString`
- `curl|bash` / `wget|sh` piped execution
- `DROP DATABASE|TABLE`, `rm -rf /`, `Remove-Item ... C:\Windows`
- Reverse-shell shapes (`/dev/tcp` abuse, `TcpClient` callback)
- Hidden-window launches (usage re-verified)

Second pass: manual reading of download and deletion paths in `bootstrap-reverse.ps1/.sh`, `mcp-bridge.js`, and the diagram/cryptography Python scripts.

## Findings Detail

### 1. Deletion Operations (All Expected Cleanup, Not Database Wipes)

| Location | Behavior | Risk |
|------|------|------|
| `bootstrap-reverse.ps1` `Expand-ArchiveIntoDirectory` | Deletes the target install directory, then reinstalls; deletes `%TEMP%\reverse-bootstrap-*` | Tool install paths only, not user business data |
| `bootstrap-reverse.ps1` anything-analyzer | `Remove-Item node_modules` then `pnpm install` on failure | Confined to the cloned tool repo |
| `apk-reverse/scripts/decode.*` | Cleans task output directories (jadx/apktool out) | Confined to the task root |
| `case-init.ps1` | Cleans temp directories | Temporary |
| `bootstrap-reverse.sh` | Same temp / install-target cleanup | Same as left |

**Not found**: executable `DROP`/`TRUNCATE` logic targeting `C:\`, system directories, or arbitrary database connection strings.

### 2. Network Behavior (Tool Bootstrapping, Not C2)

| Location | Behavior | Notes |
|------|------|------|
| `bootstrap-reverse.ps1` | Pulls releases from `api.github.com`; downloads zip/jar via `Invoke-WebRequest` | Repo names come from the **manifest allowlist** |
| `bootstrap-reverse.sh` | `curl` / `git clone` / `pipx` / `npm` | Same as above |
| `mcp-bridge.js` | HTTP → Burp on `127.0.0.1:9876` only | Local loopback |
| `ToolDiscovery.ps1` | Probes `http://host:port/mcp` | Health check |
| `kali/.../tool-discovery.sh` | `(echo >/dev/tcp/$host/$port)` | **Port probe**, not a reverse shell |

### 3. Hidden Windows

| Location | Purpose |
|------|------|
| `bootstrap-reverse.ps1` `Start-Process ... -WindowStyle Hidden` | Background-launch `pnpm dev` (anything-analyzer) |
| `ida-reverse/scripts/start.ps1` | Launches IDA-related processes (must stay in background) |

These are service-startup shapes; no hidden malicious payload downloads found.

### 4. "Dangerous Strings" in Docs / Payloads (Not Auto-Executed)

`pentest-tools/src-hunter`, `attack-chain`, etc. are **Markdown/JSON educational material** containing SQL injection, `DROP` examples, and log-cleanup **red-team methodology**.  
These are **never auto-executed by bootstrap or master-route**; execution depends on the AI/human choosing them within an **authorized scope**.

Related constraints: `ops/scope-contract.md`, `ops/skill-supply-chain.md`, `field-journal/precedent-*.md`.

### 5. Residual Supply-Chain Risk (Suggested Follow-Up Hardening, Not a Confirmed Backdoor)

| Item | Risk | Suggestion |
|----|------|------|
| `@jshookmcp/jshook@0.3.4`, `pentestswarm@v0.1.0` in `bootstrap-manifest.json` | Tag drift / supply-chain poisoning surface | Pin versions + checksums |
| GitHub release zips **without SHA256 verification** | Hard to detect a replaced release in time | Add `assetSha256` to the manifest and verify in bootstrap |
| `npm install -g` / `pip` default sources | Inherent dependency-ecosystem risk | Only install manifest capabilities; use private sources/locks in production |

## Executable Script Inventory (Audit Baseline)

```
skills/scripts/*.ps1|*.sh + lib/ToolDiscovery.ps1
skills/apk-reverse/scripts/*
skills/radare2/scripts/*
skills/ida-reverse/scripts/*
skills/browser-automation/scripts/*
skills/diagram-generator/scripts/*.py
skills/case-review/scripts/*.py
kali/scripts/*
burp-mcp-full/mcp-bridge.js (+ Java extension sources)
```

## Suggested Ongoing Checks

```powershell
# Quick executable-surface health check (example)
rg -n "Invoke-Expression|FromBase64String|DownloadString|rm -rf /|DROP DATABASE" skills/scripts skills/*/scripts kali/scripts burp-mcp-full -g "*.ps1" -g "*.sh" -g "*.py" -g "*.js"
```

Executable scripts of newly added skills should be re-run against this checklist before merging; Markdown-only methodology changes are not mandatory.

## Sign-off

- Audit performed: repository-local static scan + manual review of critical paths  
- Result: no backdoor / no automatic database wipe; supply-chain hardening listed as a follow-up improvement  
'@
