# AI Agent Obedience Engineering — Making the AI Actually Do the Work After Reading the Workflow

> Source: 2026 synthesis of multiple sources (Anthropic Skill Engineering, Microsoft Code Words, Strands Steering Hooks, Gradient Flow Harness Engineering)
> Applicable scenario: AI coding agents (Claude Code / Codex / Cursor / Cline / Windsurf / Kiro, etc.) that, after reading README/RULES.md, only confirm without executing, skip steps, or take it upon themselves to omit critical operations

---

## Core Problem Diagnosis

The root cause of an AI agent that "reads the workflow but does not work" is not insufficient model capability — it is the **semantic escape space in natural-language instructions**:

| Root cause | Explanation |
|------|------|
| **Context attention decay** | Content in the middle of long documents is down-weighted by the LLM attention mechanism; the agent actually only "sees" the beginning and the end |
| **Semantic override** | While optimizing for "helpfulness", the model creatively re-interprets explicit instructions (e.g., reading MUST DO X as "it is suggested to do X") |
| **Passive language treated as optional** | "Ready for next step → invoke X" is treated as a suggestion rather than an instruction |
| **Stateless enforcement** | Without an external state machine validating workflow order, the agent can skip steps without being noticed |
| **Silent state corruption** | The agent produces results that are structurally correct but semantically wrong, and errors accumulate silently |

---

## Technique 1: Critical-First Pattern

**Put "what to do next" first, and put context after.**

```
WRONG (agent ignores it):
  [70 lines of project background and tool list]
  → "Next step: run bootstrap to install the missing tools"

CORRECT (agent executes):
  "## Execute now: run `bootstrap-reverse.ps1` to check and install missing tools
   → When done, read routing.md to determine which skill to enter"
  [Then project background and tool list]
```

**Rationale**: LLMs assign the highest attention weight to the first and last parts of a prompt. Middle content may be completely ignored.

**Applied to this project**:
- The "Routing entry" section of RULES.md should sit after the trigger keywords and before the execution principles
- The first section of every SKILL.md should be "Execute immediately", not "Scope of application"

---

## Technique 2: Directive Over Suggestive

Replace all "suggestive" language with RFC 2119-level directive language:

| Weak language (agent may skip) | Strong language (agent must enforce) |
|---|---|
| "You can try..." | **MUST**: you must execute... |
| "Ready for next step → invoke X" | **NOW**: call X immediately, do not wait for confirmation |
| "It is recommended to read routing.md first" | **REQUIRED**: routing.md must be fully read before entering any submodule |
| "You may bootstrap if tools are missing" | **NO EXCUSE**: when tools are missing, the only correct action is to call bootstrap; guessing manual installs is forbidden |
| "Remember to update the field-journal" | **CHECKLIST ENFORCED**: tick every Checklist item after the task; you may not claim task completion without doing so |
| "You should..." | **MUST** / **MUST NOT** |

**Key pattern**:
```
MUST — violation = task failure
MUST NOT — violation = security violation
SHOULD — skipping requires an explanation
MAY — genuinely optional
```

---

## Technique 3: Excuse Rebuttal Table

**This is the most critical patch for this project.** When an AI agent meets resistance, it automatically generates "reasonable excuses" to skip steps. List the common excuses in advance and rebut each one:

| Common agent excuse | Rebuttal (enforced) |
|---|---|
| "I can skip this step, let me just..." | **Skipping is forbidden.** Every step in the behavior chain is required. If you think you can skip, first output your specific reason and let the user decide. |
| "Based on my judgment, this isn't necessary" | **Your judgment does not apply here.** List the specific criteria you used for your judgment and explain why those criteria allow skipping an explicitly written step. |
| "The user probably doesn't need this" | **Never decide for the user.** Present all options to the user, mark your recommendation, but do not hide alternatives. |
| "I already know how to do this, no need to read X" | **Read X first, then act.** Even if you are sure you know how to do it, X may contain task-specific constraints. Reading takes only 2 seconds. |
| "To save time, I can skip by doing these in parallel..." | **The correct way to save time is to execute independent steps in parallel, not to skip steps.** If two steps do not depend on each other, do them in parallel; if they do, do them in order. |
| "I've used this tool before, I know the path" | **Guessing paths is forbidden.** You must get the actual path from tool-index; different machines install tools in different locations. |
| "The task is basically done, no checklist needed" | **The only definition of task completion is every Checklist item being ticked.** A task without a completed Checklist is not complete. |
| "I couldn't find tool-index, so I'll just guess the paths" | **A missing file is 100x safer than a wrong guessed path.** When tool-index is missing, run refresh-tool-index.ps1 first to generate it. |
| "The user didn't explicitly ask for a report, so I won't write one" | **Reporting is the default behavior, not an option.** A report must be produced after a security task, unless the user explicitly says "no report". |
| "This is too simple, no journal entry needed" | **Even simple tasks carry pitfall value.** At minimum record: target type + what was used + whether anything unexpected happened; one line is enough. |
| "The user asked me to redo the import table / step X, but I did something else that is more useful instead" | **Redo = redo the exact named step** (or a prerequisite path confirmed by the user). MUST refresh the corresponding Evidence; impersonating it with an unrelated step is forbidden, silently skipping is forbidden. Unpacking is a **prerequisite** for a readable IAT, not a **substitute** for import-table Evidence. |
| "The user said not to unpack the packed sample and to look at the import table first; I'll just hand in the garbage table as done" | **Feasibility gate:** when X is blocked, MUST state the blocker, give the recommended order, and **ask the user to confirm**. If the user insists, execute and mark `quality=unreadable/packed`; drawing capability-negative conclusions from a garbage table is forbidden. |
| "It crashes after unpacking; I'll keep thrashing the file on disk" | **Patch 6:** record E-self-check-crash / E-iat-repair-fail, switch to dynamic (bp CreateFile/GetFileSize). Infinite static file modification is forbidden. |
| "I can't fix the IAT; let me waste more time trying more unpackers statically" | **IAT repair iron rule:** prefer automatic/semi-automatic repair; if the tool errors out or the binary will not run after repair, STOP static IAT immediately, record E-iat-repair-fail, and switch to dynamic API breakpoint capture. Infinite static grinding is forbidden. |
| ".NET / no import table, so the hard gate doesn't apply; I'll skip it" | **An equivalent anchor is still MUST:** for .NET, write the dnSpy/IL/metadata summary into the E-imports semantic slot; for DLL/SYS, E-exports must accompany imports. Passing with nothing is forbidden. |


**How to use**: place this table near the end of RULES.md or other instruction files (the high-attention region). The agent sees the rebuttals before it reaches for an excuse.

---

## Technique 4: Five Skill-Engineering Patterns (Anthropic official, 2026)

| Pattern | Applicable scenario | Key tips |
|---|---|---|
| **Linear Flow** | Processes with clear steps (deployment, installation) | Provide safe defaults; use negative instructions ("MUST NOT use --force") |
| **Decision Tree** | Platform navigation, fault diagnosis | Tree navigation + progressive loading via `references/` |
| **Iterative Loop** | TDD, review-fix cycles | Hard rules up front + an **excuse rebuttal table** to block shortcuts |
| **Baton Loop** | Multi-session, multi-agent collaboration | Externalize state to `next-prompt.md` (MUST write it before exiting) |
| **Multi-Phase + Checkpoints** | Multi-day complex workflows | Orchestrator "parent" skill + human Go/No-Go checkpoints, with time costs annotated |

**Mapping to this project**:
- Full behavior chain = Linear Flow (15 steps executed in order)
- Routing matrix = Decision Tree (three-dimensional matching)
- Checklist = Multi-Phase Checkpoint (every step must be ticked)
- Field Journal = Baton Loop (externalizing state across sessions)

---

## Technique 5: In-Band Enforcement Validation (the Steering Hooks idea)

Instead of relying on the AI's "self-discipline", embed a self-validation instruction in the prompt:

```
Before every claim of "task complete", MUST self-check first:
1. Did I skip any step in the behavior chain? Which one?
2. Did I guess any tool paths? If so, what is the actual path in tool-index?
3. Are all Checklist items ticked? If not, why?
4. If the answer to any item above is "yes"/"unticked", the task is not complete —
   go back to the corresponding step and re-execute; do not declare completion.
```

This method makes the agent audit itself before it says "done", which is more immediate than external validation.

---

## Technique 6: Code Words — for API/tool parameters

Microsoft 2026 research found that semantic parameter names trigger the model's tendency to "helpfully optimize".

```
WRONG: { "query": "...", "top": 9 }        → 68.4% parameter-adherence rate
CORRECT: { "query": "...", "code": "alpha" } → 100% parameter-adherence rate
```

**Application scenarios**:
- When precise configuration must be passed into bootstrap scripts, use short codes instead of semantic parameters
- For parameters in tool calls that need a strong guarantee, use a code-word mapping

---

## Technique 7: Dual AI Review Loop (Dual Validation)

```
AI A (executor) writes the output
  ↓
AI B (reviewer) checks it against the rules
  ↓ passed
Output is given to the user
  ↓ failed
Returned to AI A for correction, with citations of the specific violations
```

**Application in this project**:
- Embed a "self-review" step in RULES.md: before the agent outputs its report, it uses its own capability to check every Checklist item
- If it finds unfinished items, it goes back to the corresponding step and completes them itself

---

## Technique 8: Context Window Layout Optimization

LLM attention distribution (high → low):
```
[first 10%]  ████████████ ← Highest attention — put "execute now" instructions here
[middle 80%] ████░░░░░░░░ ← Attention decays — put reference material here
[last 10%]   ████████████ ← Attention recovers — put "MUST NOT skip" and the Checklist here
```

**Concrete application**:
1. **First 10%**: execute-now instructions + trigger keywords
2. **Middle 80%**: detailed workflow, reference links, tool lists
3. **Last 10%**: excuse rebuttal table + hard Checklist + forbidden-behavior list

---

## Practical Prompt Templates

### Template A: Forced Startup Template (embed at the beginning of RULES.md)

```markdown
## CRITICAL: after reading this document you must immediately execute the following (do not just confirm — actually execute)

1. **NOW**: detect the directory of this file → that is the package root
2. **NOW**: if this is the first use, write these rules into the global configuration (see the Global Injection section)
3. **NEXT**: read `skills/SKILL.md` → `skills/routing.md` → determine which sub-skill to enter
4. **NEXT**: read `skills/tool-index.md` to confirm tool status
5. **THEN**: start executing the actual task; do not stay in the "read it" state

If you only reply "read", "done", or "I understand" without actually executing the steps above,
you have failed. The user needs tools installed, code analyzed, vulnerabilities verified —
not a confirmation message.
```

### Template B: Submodule Entry Template (embed at the beginning of every SKILL.md)

```markdown
## ACTION REQUIRED (execute immediately after reading, do not wait)

After reading this file:
1. Confirm that you understand this skill's applicable scenarios
2. Check whether this machine has the required tools (read `../tool-index.md`)
3. If tools are missing → call bootstrap
4. If tools exist → start with the first workflow step
5. If you are unsure → list the concrete questions; do not stay silent
```

### Template C: Task Completion Self-Check Template (embed at the end of every SKILL.md)

```markdown
## Task completion self-check (MUST confirm item by item before claiming completion)

□ I actually executed every step of the behavior chain (nothing skipped)
□ I did not guess any tool paths (all come from tool-index.md)
□ I produced reproducible commands/scripts/reports (not just descriptions of steps)
□ I updated the field-journal (if any pitfalls were hit)
□ I executed the completion Checklist (report + diagram + experience write-back)
```

---

## Forbidden Behaviors (from the perspective of agent obedience)

- Forbidden: replying "understood, tell me the specific task" after reading RULES.md
  → Correct behavior: execute global injection → read SKILL.md → read routing.md → determine the entry
- Forbidden: saying "steps 1-4 are done" when you only read them
  → Correct behavior: distinguish "read the document" from "executed the operation"; only the latter produces actual side effects
- Forbidden: saying "task complete" without executing the Checklist
  → The Checklist is the only definition of task completion
- Forbidden: substituting "based on experience" for reading tool-index
  → Paths differ across machines; checking tool-index is the only way to locate them

---

## Summary: if you can only change one thing

**Add an "act now" directive at the very beginning of RULES.md**, using strong directive words such as bold, CRITICAL, and NOW.

This is the change with the highest return on investment. Most "not working" agent behavior comes from automatically falling into "waiting for user instructions" mode after reading a file. One mandatory "act now" directive can break that pattern.

If you change a second thing: **add the excuse rebuttal table**. The agent will reach for an excuse and stop at the first resistance; blocking those excuses in advance is the fix.
