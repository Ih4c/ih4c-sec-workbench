# Evidence → Finding → Path Evidence Chain

> Inspired by the Z3r0 Evidence Plane, implemented here as a **Markdown field contract**.  
> reverse-skill flavor: bound to the `docs-generator` report templates, sanitized `field-journal` write-backs, and reproducible command recording.

## 1. Evidence (immutable observations)

Each Evidence item is its own paragraph or table row:

```markdown
### E-{nnn}
- title:
- observed_at:
- source_type: command | screenshot | file | log | memory | network | manual
- source_ref: {path or command id}
- content_hash: {sha256 of artifact if file, else n/a}
- artifact_path: {relative path under case root when content_hash is recorded, else n/a}
- repro_command: |
    {exact command}
- raw_excerpt: |
    {sanitized excerpt}
- linked_workitem: WI-{nnn} | n/a
- supersedes: E-{nnn} | none
```

**MUST**: every Finding references at least 1 Evidence; `repro_command` must be runnable by a third party or explicitly note offline limitations.

**CLI helper** (writes `work/<case>/evidence/E-*.md`):

```powershell
powershell -File skills/scripts/append-evidence.ps1 -CaseRoot work/<case> `
  -Id E-001 -Title "..." -ReproCommand "..." -Severity info -Status observed
```

When the evidence is a case-local file, pass `-ArtifactPath` to record a SHA-256 fixity value and a relative artifact path. Review the complete case graph before handoff:

```bash
python3 skills/case-review/scripts/review_case.py work/<case> --verify-hashes --strict
```

The review is read-only and checks scope fields, Evidence records, work item and timeline references, structured Findings, Paths, and artifact hash matches.

## 2. Finding (security/reverse conclusion)

```markdown
### F-{nnn}
- title:
- severity: critical | high | medium | low | info | n/a_re
- category: vuln | misconfig | design | reverse_algo | bypass | other
- status: candidate | validated | false_positive | accepted_risk
- evidence_ids: [E-001, E-002]
- location: {file:line | addr | url | class.method}
- impact:
- confidence: high | medium | low
- repro_steps:
  1.
  2.
- remediation: {or n/a for pure RE}
- optional_attack: {ATT&CK ID or empty}
```

**MUST**: `evidence_ids` is never empty; when `status=validated`, confidence must not be low (unless residual risk is noted).

## 3. Path (attack path / call flow / solve path)

Uniformly called **Path**, interpreted by task type:

| Task | Meaning of Path |
|------|-----------|
| Pentest / attack chain | attack path steps |
| Reverse engineering | key call/data flow steps |
| CTF | solve steps |

```markdown
### P-{nnn}
- title:
- path_type: attack | callflow | solve
- start:
- goal:
- steps:
  1. action: — evidence: E-xxx — finding: F-xxx | none
  2. action: — evidence: E-xxx — finding: F-yyy | none
- residual_risks:
```

**MUST**: each step can be linked to Evidence; a terminal Finding on an attack path that claims "access/data obtained" must have validated Evidence.

## 4. Position in the report

A `docs-generator` security report **MUST** contain:

1. Scope summary (linked to the case `scope.md`)  
2. Evidence table or section  
3. Findings list (including evidence_ids)  
4. At least 1 Path (attack/callflow/solve)  
5. Timeline summary (optional full text linked to `timeline.md`)

See the **Evidence Chain** section of `docs-generator/references/security-report-templates.md`.

## 5. field-journal hook

When writing back to the journal, **SHOULD** excerpt:

- key Evidence ids + commands (within 3)
- 1 core Finding
- one sentence capturing a reusable Path pattern

Full sensitive content belongs only in the user project's report; journal entries **MUST** be sanitized (`anonymization.md`).

## 6. Differences from Z3r0 (flavor)

| Z3r0 | reverse-skill |
|------|----------------|
| PG immutable rows + API | Markdown files + hash fields |
| UI review queue | report + next-step menu + journal |
| deep ATT&CK binding | optional tags, UI not enforced |


## Validated sufficiency (Issue #77 / R4*)

Global bind rule remains: every Finding references **>=1** Evidence.

Promotion to status=validated is stricter (decision cookbook):

| status | Evidence bar |
|--------|----------------|
| preliminary / candidate | >=1 (unchanged) |
| **validated** | **SHOULD >=2 independent** Evidence (best: 1 static + 1 dynamic). A single Evidence item alone MUST NOT silently promote to validated — keep candidate/preliminary, or record residual_risk + human confirm. |
| blocked promotion | record Evidence E-insufficient-evidence |

Full recipes: [nalysis-decision-framework.md](analysis-decision-framework.md) (R4*, R1, R41, R44).
