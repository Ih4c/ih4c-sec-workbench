# Cybersecurity Skills Router — Kali Linux Edition

> This directory is the Kali Linux 2026.1 optimization adaptation layer, tuned specifically for Kali 2026.1 (kernel 6.18), released in March 2026.
> The core knowledge base (`skills/`, `CTF-Sandbox-Orchestrator/`) is shared with the Windows edition; the Kali-specific README and Bash entry points must cover the Windows core capability names while additionally providing Kali-native tooling / MCP capabilities.  
> The root [README-kali.md](../README-kali.md) is only a short jump; **this file is authoritative**.

---

## For AI: Execute Immediately After Reading

```text
1. Detect the package root (the repo root containing skills/ and kali/)
2. Read kali/RULES-kali.md → global injection and tool scanning
3. bash kali/scripts/refresh-tool-index.sh
4. Combat chain shared with the main package:
   - skills/MASTER-ROUTING.md (or pwsh skills/scripts/master-route.ps1)
   - skills/scripts/case-init.ps1 → work/<case>/scope.md
   - Act on the target only after auth.status=granted + network_profile
   - skills/ops/ (evidence chain / roles / timeline / IDENTITY)
5. Report configuration results to the user
```

For general agent bootstrap see the repo root [README_AI.md](../README_AI.md) (read this file only when Kali is detected).

---

## 0. Relationship to the Windows Edition (Capability Name Alignment)

```text
Project root/
├── skills/                    # Shared: SKILL, routing, MASTER-ROUTING, ops, scripts, field-journal
├── CTF-Sandbox-Orchestrator/  # Shared: 40+ CTF sub-skills
├── kali/                      # ← You are here
│   ├── scripts/
│   │   ├── bootstrap-reverse.sh
│   │   ├── refresh-tool-index.sh
│   │   ├── bootstrap-manifest.json
│   │   └── lib/
│   │       └── tool-discovery.sh
│   ├── RULES-kali.md
│   └── README-kali.md
├── RULES.md                   # Windows edition rules
└── Readme.md                  # Windows edition guide
```

### 0.1 Alignment Principles

The Kali entry point is not a simple copy of the Windows README; it is **the same core capability names + additional Kali capabilities**:

- Windows: `skills/scripts/bootstrap-reverse.ps1`
- Kali: `kali/scripts/bootstrap-reverse.sh`
- Generic Linux/macOS: `skills/scripts/bootstrap-reverse.sh`

JEB Pro is a commercial tool that users license and install themselves; Reqable MCP uses the official pinned version of `reqable-mcp-server`, but still requires the Reqable desktop client to be installed separately.

Kali scripts should cover the core capability names from the Windows manifest, e.g., `jadx`, `apktool`, `frida`, `jshookmcp`, `xquik-mcp`, `anything-analyzer`, `idapro`, `r2`, `adb`, `ghidra-mcp`, `seclists`, `burpsuite-mcp`, `nmap`, `pentestswarm`; they may additionally support Kali-native tools such as `mcp-kali-server`, `metasploitmcp`, `hexstrike-ai`, `sstimap`, `xsstrike`, `netexec`, etc.

**Shared parts** (no changes needed):
- All `SKILL.md`, `routing.md`, `MASTER-ROUTING.md`
- The `skills/ops/` combat contracts (scope / evidence chain / roles / timeline)
- The entire `references/` knowledge base
- The `field-journal/` self-evolution mechanism
- The whole `CTF-Sandbox-Orchestrator/`
- `docs-generator/`, `diagram-generator/`
- `skills/scripts/case-init.ps1`, `master-route.ps1` (invocable via pwsh)

**Kali-specific parts**:
- All scripts are bash (`.sh`)
- Package management via `apt`
- Linux-style path conventions (`/opt/`, `~/tools/`, `/usr/bin/`)
- Many tools come preinstalled on Kali, so the bootstrap logic is greatly simplified

---

## 1. Kali's Native Advantages

The following tools are **ready out of the box** on Kali 2026.1 (no bootstrap needed):

### Classic Preinstalled Tools

| Tool | Kali Package | Status |
|------|----------|------|
| nmap | nmap | Preinstalled |
| sqlmap | sqlmap | Preinstalled |
| hashcat | hashcat | Preinstalled |
| john | john | Preinstalled |
| hydra | hydra | Preinstalled |
| metasploit | metasploit-framework | Preinstalled |
| gobuster | gobuster | Preinstalled |
| ffuf | ffuf | Preinstalled |
| radare2 | radare2 | Preinstalled |
| binwalk | binwalk | Preinstalled |
| frida | python3-frida-tools | Preinstalled or pip |
| burpsuite | burpsuite | Preinstalled |
| wireshark | wireshark | Preinstalled |
| nikto | nikto | Preinstalled |
| wfuzz | wfuzz | Preinstalled |
| impacket | impacket-scripts | Preinstalled |
| netexec | netexec | Preinstalled |
| responder | responder | Preinstalled |
| aircrack-ng | aircrack-ng | Preinstalled |
| bloodhound | bloodhound | Installable via apt |
| ghidra | ghidra | Installable via apt |

### New Tools in Kali 2026.1 (March 2026)

| Tool | Package | Purpose |
|------|------|------|
| AdaptixC2 | adaptixc2 | Post-exploitation and adversary simulation framework |
| Atomic-Operator | atomic-operator | Cross-platform Atomic Red Team test execution |
| Fluxion | fluxion | WiFi security auditing and social engineering |
| GEF | gef | Modern GDB enhanced debugging framework |
| MetasploitMCP | metasploitmcp | MCP server interface for Metasploit |
| SSTImap | sstimap | Automatic server-side template injection detection and exploitation |
| WPProbe | wpprobe | Fast WordPress plugin enumeration |
| XSStrike | xsstrike | Advanced XSS scanner |

### New Tools in Kali 2025.4 (December 2025)

| Tool | Package | Purpose |
|------|------|------|
| evil-winrm-py | evil-winrm-py | Python version of WinRM remote command execution |
| hexstrike-ai | hexstrike-ai | AI MCP security automation platform (150+ tools) |
| bpf-linker | bpf-linker | BPF static linker |

### Kali-Native MCP Tools (Key Optimization)

| Tool | Package | Purpose | Install |
|------|------|------|------|
| mcp-kali-server | mcp-kali-server | Kali official MCP; AI directly invokes terminal tools | `apt install mcp-kali-server` |
| MetasploitMCP | metasploitmcp | Metasploit MCP interface | `apt install metasploitmcp` |
| HexStrike AI | hexstrike-ai | MCP automation for 150+ security tools | `apt install hexstrike-ai` |

> **This is the biggest advantage of the Kali edition over the Windows edition**: three MCP tools install directly via apt, no manual GitHub/npm/Docker configuration needed.

This means `bootstrap-reverse.sh` does far less work on Kali than the Windows edition.

---

## 2. Quick Start

### 2.0 One-Command Initialization (Recommended for New Systems)

```bash
# One-command setup for a fresh Kali 2026.1 system (requires root)
sudo bash kali/scripts/quick-setup.sh

# Skip system updates (for slow networks)
sudo bash kali/scripts/quick-setup.sh --skip-update

# Minimal install (skip AD / internal-network tools)
sudo bash kali/scripts/quick-setup.sh --minimal
```

This script automatically performs: system update → install new 2026.1 tools → configure native MCP → install reverse-engineering tools → refresh the index → output a report.

### 2.1 First-Time Configuration

```bash
# 1. Enter the project root
cd /path/to/cybersecurity-skills-router

# 2. Make the scripts executable
chmod +x kali/scripts/*.sh kali/scripts/lib/*.sh

# 3. Refresh the tool index (detect local tool status)
bash kali/scripts/refresh-tool-index.sh

# 4. View the results
cat skills/tool-index.md
```

### 2.2 One-Command Setup of Kali-Native MCP (Highly Recommended)

```bash
# Install the Kali official MCP trio
bash kali/scripts/bootstrap-reverse.sh mcp-kali-server metasploitmcp hexstrike-ai

# After installation the MCP config is written to ~/.claude/mcp.json automatically
# If using Kiro, copy it manually to ~/.kiro/settings/mcp.json
```

### 2.3 Installing the New 2026.1 Tools

```bash
# One-command install of all new tools
bash kali/scripts/bootstrap-reverse.sh adaptixc2 atomic-operator sstimap xsstrike wpprobe fluxion gef

# AD / internal-network penetration suite
bash kali/scripts/bootstrap-reverse.sh coercer evil-winrm-py netexec responder bloodhound certipy
```

### 2.4 Installing Missing Tools

```bash
# Install a single tool
bash kali/scripts/bootstrap-reverse.sh jadx

# Install multiple tools
bash kali/scripts/bootstrap-reverse.sh jadx apktool frida jshookmcp

# Install and start services
bash kali/scripts/bootstrap-reverse.sh idapro --start-services
```

### 2.5 Making AI Clients Route Automatically

Tell your AI client to read `kali/RULES-kali.md`; it will perform the global injection automatically.

---

## 3. Path Conventions

| Purpose | Kali Path |
|------|----------|
| Tool install directory | `~/tools/` or `/opt/` |
| jadx | `/opt/jadx/` or `~/tools/jadx/` |
| apktool | `/usr/local/bin/apktool` (apt) or `~/tools/apktool/` |
| Ghidra | `/opt/ghidra/` or `~/tools/ghidra/` |
| IDA Pro | `/opt/idapro/` (if a Linux edition is available) |
| Android SDK | `~/Android/Sdk/` |
| SecLists | `/usr/share/seclists/` (apt) or `~/tools/SecLists/` |
| Node.js | `/usr/bin/node` (apt/nvm) |
| Python | `/usr/bin/python3` (system built-in) |
| MCP config | `~/.claude/mcp.json` or `~/.kiro/settings/mcp.json` |

---

## 4. Summary of Differences from the Windows Edition

| Dimension | Windows Edition | Kali Edition |
|------|-----------|---------|
| Script language | PowerShell (.ps1) | Bash (.sh) |
| Package management | winget / GitHub Release ZIP | apt / pip / npm / GitHub Release tar.gz |
| Path separator | `\` | `/` |
| Environment variable | `%USERPROFILE%` | `$HOME` |
| Preinstalled tools | Almost none | Many security tools preinstalled |
| IDA startup | `start.ps1` | Start the Linux edition of IDA manually; the script only registers/checks MCP, unless a launcher was added locally |
| MCP config path | `%USERPROFILE%\.claude\mcp.json` | `~/.claude/mcp.json` |
| Port detection | `TcpClient` | `nc -z` or `ss` |

---

## 5. Verification Checklist

```bash
# ─── Basic commands ───
java -version
python3 --version
pip3 --version
node -v
npx -v

# ─── Reverse-engineering tools ───
jadx --version
apktool --version
adb version
frida --version
r2 -v
gdb --version          # GEF auto-loads

# ─── Pentest tools (Kali preinstalled) ───
nmap --version
sqlmap --version
hashcat --version
hydra -h | head -1
msfconsole --version
gobuster version
ffuf -V
nuclei -version

# ─── New Kali 2026.1 tools ───
sstimap -h 2>&1 | head -3
xsstrike -h 2>&1 | head -3
wpprobe --help 2>&1 | head -3
coercer -h 2>&1 | head -3
evil-winrm-py -h 2>&1 | head -3

# ─── AD / internal-network tools ───
netexec --help 2>&1 | head -3
responder -h 2>&1 | head -3
certipy --version 2>&1 | head -1

# ─── Kali-native MCP ───
which kali-server-mcp && echo "mcp-kali-server OK"
which metasploitmcp && echo "metasploitmcp OK"
which hexstrike-ai && echo "hexstrike-ai OK"

# ─── Refresh the tool index ───
bash kali/scripts/refresh-tool-index.sh

# ─── Check MCP services (if configured) ───
nc -z 127.0.0.1 5000 && echo "mcp-kali-server OK" || echo "mcp-kali-server offline"
nc -z 127.0.0.1 8085 && echo "metasploitmcp OK" || echo "metasploitmcp offline"
nc -z 127.0.0.1 13337 && echo "IDA MCP OK" || echo "IDA MCP offline"
nc -z 127.0.0.1 23816 && echo "anything-analyzer OK" || echo "anything-analyzer offline"
```

---

## 6. FAQ

### Q: The radare2 bundled with Kali is too old; what should I do?

```bash
# Install the latest version from the official source
bash kali/scripts/bootstrap-reverse.sh r2
# The Kali edition prefers installing/completing radare2 via apt by default; for the latest version, switch to GitHub/source per the platform docs
```

### Q: I use Parrot OS / BlackArch; will this work?

Yes. The scripts detect whether a command exists; they are not bound to a specific distribution. Only the `apt`-related auto-install may need to be changed to `pacman` (BlackArch).

### Q: How do I set up the Linux edition of IDA Pro?

Install IDA to `/opt/idapro/`, then modify the `startScript` path for `idapro` in `kali/scripts/bootstrap-manifest.json`.

### Q: I want to use this system on both Windows and Kali

No problem. The `skills/` directory is synced via Git, and `field-journal/` experience is shared on both sides. When executing scripts, just use `skills/scripts/*.ps1` on Windows and `kali/scripts/*.sh` on Kali.
