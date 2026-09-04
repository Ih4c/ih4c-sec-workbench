# Security / Reverse / Pentest Technical Document Templates

This file provides document templates for security projects such as reverse engineering, penetration testing, and vulnerability analysis. After a task completes, the AI should create a document in the user's project directory and output according to the corresponding template.

---

## 0. Evidence Chain (all security reports MUST include)

> Full contract: `skills/ops/evidence-finding-path.md`  
> Case directory: `work/<case>/` (`case-init.ps1`)

The report body **MUST** include the following sections (they may be merged into "Core Findings", but the fields MUST NOT be omitted):

### 0.1 Scope summary
- Link to `scope.md`: `auth` / `in_scope` / `network_profile`
- No scope → task completion MUST NOT be claimed

### 0.2 Evidence
At least 1 entry, fields: `E-id` / `source_ref` / `repro_command` / `content_hash|n/a`

### 0.3 Findings
Each: `F-id` / `severity|n/a_re` / `evidence_ids` / `confidence` / `location` / `status`

### 0.4 Path
At least 1 `P-id`: `path_type=attack|callflow|solve`; steps may attach E/F

### 0.5 Timeline summary
Link to `timeline.md` or embed the key 3-10 appended records

---

---

## 0.6 Vendor structure overlay (professional vendor report structure)

> Full rules: `references/vendor-report-rules.md` (Issue #65)  
> **MUST** be read and a flavor selected when generating formal security reports; **extract only the structure; copying vendor originals/IOC instances is forbidden**.

| Flavor / Overlay | Scenario | One-line skeleton |
|------------------|------|------------|
| `malware` | clear malicious sample / common trojan / white-and-black (白加黑) loading | Huorong-style: overview → flow → sample analysis → incident response → IOC |
| `apt` | APT / campaign / multi-stage chain | Kaspersky-style: summary → infection chain → investigation → Interesting findings → technical analysis → detection & mitigation → IOC |
| `flavor = null` | general reverse / pentest / CTF / JS signing | this section's task template + applicable Base common elements |
| thin `vuln` | vulnerability / patch / CVE technical analysis (explicit) | overview → impact/reproduction → crash and patch analysis → protection recommendations |

**Common elements (G1-G7) summary**: G1 Executive summary MUST · G2 Scope MUST · G3 E/F/P MUST · G4 IOC MUST only for `malware`/`apt` · G5 Recommendations MUST for `malware`/`apt`/`vuln` · G6 Appendix SHOULD · G7 ATT&CK MUST for `apt`

Flavor selection and section order follow `vendor-report-rules.md`; when it conflicts with §0.1-0.5, **the Evidence contract wins**.

## 1. Reverse Engineering Report Template

```markdown
# [Target name] Reverse Analysis Report

> Analysis date: YYYY-MM-DD
> Analyst: [AI / Human]
> Toolchain: [jadx / IDA / radare2 / Frida / ...]

## 1. Target overview

| Property | Value |
|------|---|
| File name | |
| File type | APK / ELF / PE / Mach-O / ... |
| Size | |
| MD5 | |
| SHA256 | |
| Package / entry | |

## 2. Analysis objectives

<!-- Core questions this reverse pass must answer -->

## 3. Static analysis

### 3.1 Basic information
<!-- Architecture, compiler, protection mechanisms, string characteristics -->

### 3.1.1 Import table / dependencies (binary MUST)
<!-- Write the E-imports / E-triage-imports summary; record Evidence even on failure; skipping is forbidden -->

### 3.2 Key functions/classes
<!-- List the located key logic, with code snippets -->

### 3.3 Crypto/signing algorithm
<!-- If crypto is involved, explain the algorithm, key source, and parameter construction -->

## 4. Dynamic analysis

### 4.1 Hook records
<!-- Frida / xposed / other hook targets and results -->

### 4.2 Runtime behavior
<!-- Network requests, file operations, process behavior -->

## 5. Core findings

<!-- Numbered key conclusions -->

1. ...
2. ...
3. ...

## 6. Reproduction steps

<!-- Let others reproduce your analysis results -->

```bash
# 关键命令
```

## 7. Open questions

<!-- Points not fully resolved -->

## 8. Attachments

<!-- Hook scripts, decryption code, screenshots, etc. -->
```

---

---

## 1b. Malware / APT Reports (vendor flavor)

When the task is malware analysis, virus report, or APT/campaign analysis, do **not** deliver using only the "Reverse Engineering" skeleton above; ordinary reverse tasks keep the original template and do not auto-select a vendor flavor:

1. Read `vendor-report-rules.md` and select `malware` or `apt`
2. Output following the corresponding section order
3. Still **MUST** include the §0 Evidence chain; `malware` / `apt` flavors additionally **MUST** include an IOC table
4. Static analysis of binary samples **MUST** include import-table Evidence (consistent with the radare2/ida/malware hard gate)

## 1c. Vulnerability Technical Analysis Report (thin `vuln` overlay)

When the task is **OS/component vulnerability, patch diff, CVE technical analysis**, or the user explicitly requests a "vulnerability technical analysis report":

1. Read §3b of `vendor-report-rules.md` and use the thin `vuln` section order (**not** the full malware/apt flavor)
2. **MUST** include: impact scope, reproduction within authorization or explicit n/a, crash/root-cause or patch-diff Evidence, protection/patch recommendations
3. **MUST** include the §0 Evidence→Finding→Path chain
4. **MUST NOT** extend PoC onto unauthorized targets, or copy externally weaponized exploitation details

## 2. Penetration Test Report Template

```markdown
# [Target] Penetration Test Report

> Test date: YYYY-MM-DD
> Test scope: [URL / IP / app name]
> Authorization status: [Authorized / CTF / Lab environment]

## 1. Executive summary

<!-- One paragraph: what was tested, what was found, risk level -->

## 2. Test scope

| Item | Details |
|------|------|
| Target | |
| Test type | Black-box / Gray-box / White-box |
| Test time | |
| Tools | |

## 3. Findings summary

| # | Vulnerability | Severity | Status |
|---|---------|---------|------|
| 1 | | High/Med/Low/Info | Verified/Pending |

## 4. Vulnerability details

### 4.1 [Vulnerability name]

**Severity**: High / Medium / Low

**Description**:

**Impact**:

**Reproduction steps**:

1. ...
2. ...
3. ...

**Evidence**:

```
<!-- 请求/响应/截图/payload -->
```

**Remediation**:

## 5. Attack path

<!-- If there is a full attack chain, draw the path -->

```
Entry → Recon → Exploit → Privilege escalation → Goal achieved
```

## 6. Tools and environment

| Tool | Version | Purpose |
|------|------|------|
| | | |

## 7. Remediation summary

| Priority | Recommendation |
|--------|------|
| P0 | |
| P1 | |
| P2 | |

## 8. Appendix

<!-- Full payloads, scripts, configuration files, etc. -->
```

---

## 3. CTF Writeup Template

```markdown
# [Competition name] - [Challenge name] Writeup

> Category: Web / Reverse / Pwn / Crypto / Misc / Forensics
> Difficulty: Easy / Medium / Hard
> Points: N pts
> Time to solve:

## Challenge description

<!-- Original challenge description -->

## Solution approach

### Step one: Information gathering
<!-- What was observed -->

### Step two: Vulnerability/break-in point
<!-- What key point was found -->

### Step three: Exploitation
<!-- How it was exploited -->

## Key code/Payload

```python
# exploit code
```

## Flag

```
flag{...}
```

## Pitfalls

<!-- Detours taken along the way -->

## Knowledge points

<!-- Knowledge involved in this challenge, for later review -->
```

---

## 4. JS/Web Signature Reverse Report Template

```markdown
# [Site/App] Signature Parameter Reverse Report

> Analysis date: YYYY-MM-DD
> Target endpoint: [URL]
> Signature field: [field name]

## 1. Target request

```http
POST /api/xxx HTTP/1.1
Host: example.com

param1=xxx&sign=<目标字段>
```

## 2. Location process

### 2.1 Breakpoint/Hook approach
<!-- How the signature generation site was located -->

### 2.2 Call stack
<!-- Key call chain -->

## 3. Algorithm reconstruction

### 3.1 Algorithm type
<!-- HMAC-SHA256 / AES / custom / ... -->

### 3.2 Parameter construction
<!-- Which fields participate in the signing, sorting rules, separators -->

### 3.3 Key source
<!-- Hardcoded / returned by API / timestamp-derived / ... -->

## 4. Local reproduction code

```javascript
// Node.js 复现
```

## 5. Verification results

<!-- Compare signatures generated by the reproduction code against the real request -->

## 6. Anti-scraping/risk-control considerations

<!-- Rate limits, device fingerprinting, environment detection, etc. -->
```

---

## 5. Document Output Guidelines

### Output location

- Documents default to the **user's current project directory** (not the skill package directory)
- File name format: `YYYY-MM-DD_[type]-[target-short-name]-report.md`
- If the user's project has a `docs/` directory, prefer placing documents under `docs/`

### Output timing

The AI automatically invokes this skill to generate documents at the following points:

1. Reverse task complete, core conclusions produced
2. Penetration test complete, vulnerabilities found and verified
3. CTF challenge solved, flag obtained
4. User explicitly asks to "write a report/document"

### Quality requirements

- Every code block must be directly runnable or have explicit context
- No placeholders/TODOs (if a part is genuinely unfinished, mark it "to be completed (待补充)" and state the reason)
- Key findings must be backed by evidence (command output, screenshot descriptions, code snippets)
- Reproduction steps must allow a third party to reproduce independently
