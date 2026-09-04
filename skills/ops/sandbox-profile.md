# Optional Sandbox Tool Profile (vs bootstrap-manifest)

> Z3r0's default images ship with a broad toolset; reverse-skill **does not bundle images** — this table serves as a "coverage comparison" and optional Docker suggestion.

## Capabilities reverse-skill Can Auto-Bootstrap

Source: `skills/scripts/bootstrap-manifest.json` (the file is authoritative):

| Capability | Typical scenario |
|------|----------|
| jadx / apktool / adb / frida / frida-ps | Android |
| r2 / rabin2 | binary CLI |
| idalib-mcp / idapro | IDA MCP |
| jeb-pro | commercial Android / ARM decompiler (manual license install) |
| jshookmcp / reqable-mcp / anything-analyzer / agent-browser | Web/JS/packet capture/browser |
| ghidra-mcp | Ghidra |
| nmap / seclists / proxycat / burpsuite-mcp / pentestswarm | pentest |
| binwalk / pwntools / yara | firmware/pwn/malware |

```powershell
powershell -File skills\scripts\bootstrap-reverse.ps1 -Capability @('jadx','nmap','yara') -StartServices
powershell -File skills\scripts\refresh-tool-index.ps1
```

## Common in Z3r0 Sandboxes but Not Auto-Installed by This Package's Manifest

| Tool | reverse-skill strategy |
|------|-------------------|
| subfinder / amass / httpx / ffuf / nuclei / sqlmap | document install / Kali scripts / external MCP; **do not pretend bootstrap already has them** |
| Full Ghidra GUI | ghidra-mcp capability + manual plugin steps |
| gdb / pwndbg | manual per platform docs; pwntools can be bootstrapped |
| hydra / hashcat | manual or Kali |
| JEB Pro | user manually installs after holding a license; third-party MCP bridges must pass supply-chain review first |
| Reqable desktop client | user installs manually; `reqable-mcp` only registers the official pinned-version MCP runtime |
| SecLists | seclists capability |

## Recommended "Lightweight Docker Operations" Profile (optional, not a dependency)

Only when the user **themselves** has Docker and an authorized lab:

```text
Minimal: nmap + nuclei + sqlmap containers or pentestMCP-like images
Mobile: jadx + apktool + frida on the host
Reverse: IDA/r2 on the host + tool-index
```

**MUST NOT** require the user to install Z3r0 in order to use reverse-skill.

## network_profile Integration

Scanning inside a sandbox is still constrained by the case `scope.md` `network_profile`:

- `offline` → do not start external-scan containers  
- `authorized_target_only` → containers may only hit in_scope targets  
