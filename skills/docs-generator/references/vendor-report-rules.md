# Vendor Report Rules (Professional Vendor Report Structure Overlay)

> Issue #65, question 2.  
> **Extract only the structure and writing rules; copying any vendor report body text, figures, real IOC instances, or long verbatim passages is forbidden.**  
> This file is an **overlay**: it does not replace the task templates in `security-report-templates.md`, nor does it weaken §0 Evidence→Finding→Path.

Structure references (public samples, skeleton only):

| Flavor | Primary reference | Scenario |
|--------|--------|------|
| `malware` | Huorong (火绒) Security virus/technical analysis reports | clear common trojans, white-and-black (白加黑) loading, phishing payload delivery, malicious samples |
| `apt` | Kaspersky Securelist / APT campaign reports (e.g., MATA) | APT, group campaigns, multi-stage infection chains, industry targeting |

Principle: **quality over quantity of templates** — only 2 full-text vendor flavors (`malware` / `apt`) + Base common elements + **optional thin overlays** (e.g., `vuln` vulnerability technical analysis). Ordinary reverse, pentest, CTF and JS reports keep their task template and are NOT by default disguised as malware reports; `vuln` is **not** a third default full-text flavor.

---

## 0. When to enable

When `docs-generator` produces **security-category** reports (reverse / malware / pentest wrap-up / user explicitly asks for a "professional report" or "vendor style"), this file **MUST** be read. Select a vendor flavor only when the task evidence or an explicit user request supports it; otherwise use `flavor = null`, overlaying only generic professional elements on the original task template.

| Signal | Flavor / Overlay |
|------|------------------|
| APT / group / campaign / multi-stage C2 / industry targeting / ICS / spear-phish campaign | `apt` |
| clear malicious sample, trojan, stealer, white-and-black (白加黑) loading, impersonation site | `malware` |
| user explicitly requests vulnerability/patch/CVE technical analysis, or task evidence is OS/component vulnerability research | `flavor = null` + **thin overlay `vuln`** (see §3b) |
| ordinary APK/ELF/PE/Mach-O reverse, algorithm analysis, firmware analysis, pentest, CTF, JS signing | `flavor = null`; use the original task template and the minimal set of professional common elements |

When the user explicitly requests "Kaspersky/APT style", "Huorong/virus report", or "vulnerability technical analysis", that overrides the auto-selection.  
**Forbidden** to default ordinary malware/APT/ordinary reverse into the `vuln` profile.

---

## 1. Common Professional Elements (Base)

Apply the following Base elements according to report type. Items marked **MUST** cannot be omitted; elements tied to a specific flavor MUST NOT appear in unrelated tasks merely to fill the template. Where no content applies, use `n/a` and state the reason.

| # | Element | Requirement |
|---|------|------|
| G1 | Executive summary / overview | **MUST**: 3-8 sentences: what was analyzed, most severe conclusions, impact surface, recommended actions |
| G2 | Scope and authorization | **MUST**: link to the case `scope.md` (see template §0.1) |
| G3 | Evidence→Finding→Path | **MUST**: see `security-report-templates.md` §0 and `skills/ops/evidence-finding-path.md` |
| G4 | IOC table | `malware` / `apt` **MUST**; other tasks only when relevant indicators exist |
| G5 | Recommendations / remediation | `malware` / `apt` **MUST**: at least 1 executable recommendation; other tasks follow the original task template |
| G6 | Appendix metadata | **SHOULD**: tools and versions, sample hashes, full reproduction commands |
| G7 | ATT&CK mapping | **MUST** (under `apt`; `n/a` + reason when no technique applies); other tasks **SHOULD** |

### 1.1 IOC table minimal columns

```markdown
| Type | Value | Context | First/last seen | Source evidence | Confidence |
|------|----|--------|---------------|----------|--------|
| file_sha256 / file_md5 / domain / ip:port / url / mutex / path / registry | … | where it was found | YYYY-MM-DD / n/a | E-id | high/med/low |
```

### 1.2 Copyright and security boundaries

- MUST NOT paste paragraphs or figure captions from vendor PDFs/web pages to serve as your own analysis.
- Real tokens, internal URLs, and customer identifiers use placeholders.
- For unauthorized targets, MUST NOT output directly executable attack step details (follow the case scope / RULES).

---

## 2. Flavor: `malware` (Huorong style · explicit selection)

**Narrative goal**: within 5 minutes the reader understands "what it is → how it got in → what the sample does → how to respond → which IOCs exist".

### 2.1 Recommended section order

```markdown
# [Title: one-sentence threat characterization]

> Analysis date / Analyst / Sample identifier (hash)

## 1. Overview
(G1: discovery channel, disguise method, core technical points, whether the product side can detect it — write n/a if unknown)

## 2. Attack / infection flow
(flow diagram: Mermaid or step-by-step list; corresponds to Path `path_type=attack`)

## 3. Sample analysis
### 3.1 Sample attribution
### 3.2 Static analysis
(**MUST** include import-table / basic-identity Evidence: E-imports or equivalent; see the radare2/ida/malware hard gate)
### 3.3 Dynamic analysis / behavior
(write n/a + reason when there are no dynamic conditions)
### 3.4 Core findings (Findings table or numbered list, attaching evidence_ids)

## 4. Incident response procedures
(only within the authorized scope: first confirm scope and preserve evidence — sample, memory, process tree, network connections and logs — then isolate the host; only after approval by the person in charge terminate processes, isolate/remove files, check hosts/startup items, run a full-disk scan and re-verify. MUST NOT delete files directly before evidence preservation.)

## 5. Summary notes
(risk reminders and prevention for general users/operations staff)

## 6. IOC information
(G4 table)

## 7. Evidence chain summary
(§0: E / F / P / Timeline; may merge with §3.4 but fields are not dropped)

## 8. Appendix
(tool versions, reproduction commands, script paths)
```

### 2.2 Style

- Chinese-speaking users default to Chinese; conclusions first, details after.
- Static analysis is layered by "component/phase"; avoid unstructured pastes of long logs.
- Response steps must be independently executable; no empty filler like "strengthen security awareness".

---

## 3. Flavor: `apt` (Kaspersky Securelist style)

**Narrative goal**: tell the campaign-level story — who hit whom, with which chain, when; how the investigation progressed; how the components divided labor; what defenders use to detect it.

### 3.1 Recommended section order

```markdown
# [Campaign/cluster name]: [one-sentence impact]

> Date / Team / Industry and geography scope (when knowable)

## 1. Executive summary
(G1: time window, victim profile, entry vector, family/cluster attribution, duration, most important conclusions)

## 2. The infection chain
(phased: delivery → exploit/loader → main implant → post-exploitation/stealing; explicitly mark unknown segments "limited visibility";
corresponds to Path; a chain diagram is recommended)

## 3. Incident investigation
(investigation narrative: key pivots, internal proxy/C2 characteristics, how the scope expanded; attach Timeline)

## 4. Interesting findings
(3-7 non-obvious points, each attached to E-id / F-id where possible)

## 5. Technical analysis
### 5.1 Component overview table (loader / trojan / stealer / …)
### 5.2 Per-component behavior and configuration
### 5.3 Static highlights (including import-table/packing/persistence Evidence)
### 5.4 Network and C2
(may attach the ATT&CK table G7)

## 6. Detection and mitigation
(detection ideas / hunting leads / mitigation priorities; not empty slogans)

## 7. IOC
(G4; grouped by type)

## 8. Evidence chain summary
(§0 fields)

## 9. Appendix
(sample list and hashes, tool versions, referenced public IDs; do not copy external report bodies)
```

### 3.2 Style

- Timeline and "visibility limits" must be written honestly.
- Interesting findings ≠ restating the overview; write the anomalies that genuinely mattered to the investigation.
- Component analysis uses tables: role / persistence / C2 / dependencies, then expand.

---


## 3b. Thin Overlay: `vuln` (Vulnerability Technical Analysis · Optional)

> Supplement to Issue #65. Structure referenced from the section catalogs of public "OS/component vulnerability technical analysis" reports; **skeleton sections only** — copying PoC messages, exploitation details, or unauthorized attack steps from screenshots/bodies is forbidden.  
> **Not** a third default full-text vendor flavor; overlay only in vulnerability research tasks or on explicit user request.

**Narrative goal**: the reader can quickly see "who is affected → how to confirm/reproduce (within authorization) → root cause and patch diff → how to mitigate".

### Recommended section order

```markdown
## 1. Vulnerability overview
### 1.1 Impact scope (versions/components/configuration prerequisites)
### 1.2 Reproduction (authorized environment; steps reproducible by third parties; no weaponization-tutorial tone)

## 2. Vulnerability analysis
### 2.1 Crash / anomaly analysis (Evidence: crash logs, trigger conditions)
### 2.2 Patch analysis (diff/guard conditions/fix points — attach E-*)
### 2.3 PoC or trigger analysis (only material already available within the authorized scope; describe protocol/input construction at the conceptual level)

## 3. Protection recommendations
### 3.1 Mitigations (configuration/mitigation switches, etc.)
### 3.2 Official patch and verification

## 4. Evidence → Finding → Path (may be merged into each section or as a standalone table)
```

### Hard constraints

- **MUST** have scope/authorization: reproduction and PoC extension are forbidden on unauthorized targets
- **MUST** have E/F/P: reproduction, crash, and patch conclusions all attach evidence_ids
- **MUST NOT** use `vuln` as the default shell for malware/APT
- **MUST NOT** copy exploit code or fully weaponized attack steps from external reports/screenshots
- IOC table: only when network/file indicators exist; otherwise `n/a` or omit

---
## 4. Hooking to the existing task templates

| Task template (`security-report-templates.md`) | Overlay method |
|------------------------------------------|----------|
| 1. Reverse engineering report | default `flavor = null`; keep the original "static/dynamic/reproduction" skeleton and hard-gate Evidence such as import tables; only clear malicious samples get §2 |
| 2. Penetration test report | `flavor = null`; add applicable Base G1-G3, align the attack path to §0 Path, IOC not forced |
| 3. CTF Writeup | `flavor = null`; keep the original challenge, solution approach, and reproduction structure; IOC/ATT&CK not forced |
| 4. JS/Web signature reverse | `flavor = null`; use the original overview → location → algorithm → reproduction skeleton; do not apply malware |
| Malware / APT special topics | explicitly select the `malware` or `apt` full skeleton |

**Conflict resolution**: §0 Evidence chain fields and the scope gate **always win**; a flavor only changes the narrative order and professional wrapper — it MUST NOT remove E/F/P.

---

## 5. Selection pseudocode

```
if user_requests_kaspersky or apt or threat_campaign:
    flavor = apt
elif user_requests_huorong or vir_report or explicit_malware:
    flavor = malware
else:
    flavor = null  # 原任务模板 + Base 中适用的元素
overlay = null
if user_requests_vuln_tech_report or cve_patch_analysis:
    overlay = vuln  # thin only; never a third default full flavor
emit(base_report)
if flavor in (malware, apt):
    emit(report with flavor outline)
elif overlay == vuln:
    emit(report with vuln thin outline)
```

---

## 6. Completion checklist (self-check before finishing a report)

- [ ] Flavor selected, or explicit "task template + minimal set"
- [ ] G1 overview exists and is not empty talk
- [ ] §0 E/F/P fields complete
- [ ] `malware` / `apt` reports have an IOC table (or n/a + reason)
- [ ] `malware` / `apt` reports have executable recommendations/remediation
- [ ] Tasks without a flavor were not stuffed into malware/APT-specific sections
- [ ] `vuln` only enabled for vulnerability tasks; contains the overview/analysis/protection skeleton with E/F/P; no unauthorized PoC weaponization
- [ ] No vendor original pastes, no placeholder/TODO
- [ ] Hard-gate Evidence such as import tables entered the static/technical analysis (if this task did binary analysis)

---

## 7. Source registry

- Kaspersky Securelist, "Updated MATA attacks industrial companies in Eastern Europe": <https://securelist.com/updated-mata-attacks-industrial-companies-in-eastern-europe/110829> (structure reference; accessed: 2026-08-11)
- Huorong (火绒) Security public technical article portal: <https://www.huorong.cn/> (site entry; accessed: 2026-08-11. Specific article URLs, titles, and access dates should be registered when actually cited)
- ATT&CK technique IDs serve only as a normalized mapping and MUST be supported by this case's Evidence; IOCs from external reports MUST NOT be auto-carried into the current report.

---

## 8. Non-goals

- Do not maintain additional full-text templates for Mandiant/CrowdStrike/QiAnXin (奇安信), etc. (the two flavors + optional thin overlay already cover common needs structurally).
- Do not upgrade `vuln` into a default full-text flavor on par with malware/apt.
- Do not auto-crawl vendor sites to fill reports.
- Do not lower the Evidence contract or the authorization scope because of a flavor.
