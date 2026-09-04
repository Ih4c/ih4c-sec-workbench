---
name: docs-generator
description: |
  Creates task-oriented technical documentation with progressive disclosure. Use when writing READMEs, API docs, architecture docs, or markdown documentation.
  Also use this skill at the END of any completed reverse engineering, penetration testing, CTF, or security analysis task to generate a formal report in the user's project directory.
  Trigger keywords: 写报告, 写文档, 出报告, writeup, 技术文档, report, documentation.
---

# Technical Documentation

## ACTION REQUIRED (Execute Immediately After Reading)

1. `NOW`: Confirm whether the current task hits this skill's scope
2. `NOW`: Read `../tool-index.md` to verify tool availability and actual paths
3. `NEXT`: When a tool is missing, call bootstrap — never guess paths
4. `ACT`: Enter the first step of the "Workflow" and execute; do not stop at confirmation state

For writing style, tone, and voice guidance, use `Skill(ce:writer)` with **The Engineer** persona.

## Security / Reverse Task Documentation Output

When a reverse / pentest / CTF / security analysis task completes, this skill is responsible for generating formal technical documentation in the **user's project directory**.

### Trigger timing

1. Reverse task complete, core conclusions produced (algorithm reconstruction, signature cracking, bypass solutions, etc.)
2. Pentest complete, vulnerabilities found and verified
3. CTF challenge solved, flag obtained
4. User explicitly asks to "write a report/document/writeup"

### Template selection

| Task type | Template to use |
|---------|---------|
| APK/binary/.so reverse | `references/security-report-templates.md` → Reverse Engineering Report |
| Pentest / vulnerability discovery | `references/security-report-templates.md` → Penetration Test Report |
| CTF solving | `references/security-report-templates.md` → CTF Writeup |
| JS/Web signature reverse | `references/security-report-templates.md` → Signature Reverse Report |
| Malware / APT / virus analysis report | `references/security-report-templates.md` + **`references/vendor-report-rules.md`** |
| General technical docs | `references/templates.md` → README / API docs |

### Vendor report structure (Issue #65)

Formal security-category reports **MUST** read `references/vendor-report-rules.md` (structure only; never copy vendor originals). Select a vendor flavor only when the task evidence or an explicit user request supports it; ordinary reverse and other tasks use `flavor = null`.

| Flavor / Overlay | When to use | Primary reference skeleton |
|------------------|--------|------------|
| `malware` | clear malicious samples, trojans, white-and-black (白加黑) loading, phishing payload delivery | Huorong-style: overview → flow → sample analysis → incident response → IOC |
| `apt` | APT/campaign/group/multi-stage infection chain/industry targeting | Kaspersky Securelist-style: summary → infection chain → investigation narrative → Interesting findings → technical analysis → detection & mitigation → IOC |
| `flavor = null` | ordinary APK/ELF/PE/Mach-O reverse, algorithm/firmware analysis, pentest / CTF / JS signing | original task template + Base common elements; do not apply malware/APT-specific sections |
| thin `vuln` | user explicitly requests vulnerability/patch/CVE technical analysis | overview → impact/reproduction → crash and patch analysis → protection recommendations (overlaid on null; not a third default full-text flavor) |

Principle: **quality over quantity of templates** — only 2 vendor full-text flavors; `vuln` is only an optional thin overlay, no third default full-text template set.
Takes effect **alongside** the §0 Evidence→Finding→Path contract; on conflict the Evidence contract wins.

### Output guidelines

- **Output location**: the user's current project directory (not the skill package directory)
- **File name format**: `YYYY-MM-DD_[type]-[target-short-name]-report.md`
- **If the project has a `docs/` directory**: prefer placing files under `docs/`
- **Encoding**: UTF-8
- **Language**: follows the user's conversation language (Chinese conversation produces a Chinese report, English conversation an English report)

### Quality requirements

- Every code block must be directly runnable or have explicit context
- No placeholders/TODOs
- Key findings must be backed by evidence
- Reproduction steps must let a third party reproduce independently
- Sensitive information (real tokens, passwords, internal URLs) is replaced with placeholders
- **MUST** include the Evidence → Finding → Path chain (see `../ops/evidence-finding-path.md` and template §0)
- **MUST** read `references/vendor-report-rules.md`: select `malware` / `apt` or `flavor = null` (vulnerability tasks may overlay thin `vuln`); without a flavor, output only the original task template and applicable Base elements, no forced IOC/ATT&CK
- **SHOULD** reference the case `scope.md` / `timeline.md` (`../scripts/case-init.ps1`)

### Diagram integration

When generating a report, call the `diagram-generator` skill at appropriate places to produce visual diagrams:

| Report type | Suggested diagrams | Diagram types |
|---------|---------|---------|
| Reverse engineering report | function call graph, data flow diagram | Mermaid flowchart / sequenceDiagram |
| Pentest report | attack path diagram, network topology | Mermaid flowchart / Graphviz |
| CTF Writeup | solution approach flowchart | Mermaid flowchart |
| JS signature reverse report | request chain sequence diagram, algorithm flowchart | Mermaid sequenceDiagram / flowchart |

Diagrams are embedded in the report markdown as Mermaid code blocks so they render directly on GitHub/GitLab.

---

## Core Principles

### 1. Progressive Disclosure

Reveal information in layers:

| Layer | Content | User Question |
|-------|---------|---------------|
| 1 | One-sentence description | What is it? |
| 2 | Quick start code block | How do I use it? |
| 3 | Full API reference | What are my options? |
| 4 | Architecture deep dive | How does it work? |

**Warnings, breaking changes, and prerequisites go at the TOP.**

### 2. Task-Oriented Writing

```markdown
<!-- Bad: Feature-oriented -->
## AuthService Class
The AuthService class provides authentication methods...

<!-- Good: Task-oriented -->
## Authenticating Users
To authenticate a user, call login() with credentials:
```

### 3. Show, Don't Tell

Every concept needs a concrete example.

## Formatting Standards

- **Sentence case headings**: "Getting started" not "Getting Started"
- **Max 3 heading levels**: Deeper means split the doc
- **Always specify language** in code blocks
- **Relative paths** for internal links
- **Tables** for structured data with 3+ attributes

## Quality Checklist

- [ ] Code examples tested and runnable
- [ ] No placeholder text or TODOs
- [ ] Matches actual code behavior
- [ ] Scannable without reading everything
- [ ] Reader knows what to do next

## Anti-Patterns

| Problem | Fix |
|---------|-----|
| Wall of text | Break up with headings, bullets, code, tables |
| Buried critical info | Warnings/breaking changes at TOP |
| Missing error docs | Always document what can go wrong |

## Templates

For README, API endpoint, and file organization templates, see [references/templates.md](references/templates.md).

## Related Skills

- `Skill(ce:writer)` - Writing style, tone, and voice (load The Engineer persona)
- `Skill(ce:visualizing-with-mermaid)` - Architecture and flow diagrams


---

## On-Demand Bootstrap

This skill depends on no external tools — pure text generation. No bootstrap needed.

If diagrams need to be rendered for embedding into reports, it calls the `diagram-generator/` skill.

---

## Routing Context

**Upstream entry**: all security/reverse skills automatically invoke this skill after the task completes
**Trigger methods**:
- Automatic: executed as step 9 of the behavior chain after task completion
- Manual: user says "写报告", "出文档", "writeup"

**Peer modules**:
- `apk-reverse/` — generates a reverse report after APK reverse completes
- `ida-reverse/` — generates a reverse report after binary analysis completes
- `radare2/` — generates a reverse report after CLI analysis completes
- `js-reverse/` — generates a signature report after JS signature reverse completes
- `reverse-engineering/` — generates a reverse report after general reverse completes
- `field-journal/` — report content also serves as the data source for the evolution log

**Security report templates**: `references/security-report-templates.md`
**Vendor report rules**: `references/vendor-report-rules.md` (flavor: malware | apt | null; optional overlay: vuln)
**General document templates**: `references/templates.md`


## Task Completion Self-Check (MUST pass before claiming completion)

- [ ] Did I execute every step of the workflow (rather than only reading it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Does the report contain Evidence / Finding / Path (ops contract)?
- [ ] Did I complete and write back the Checklist items required by RULES?
