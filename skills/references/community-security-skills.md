# Community Security Skill Ecosystem Survey (2026-07)

> Sources retrieved on: **2026-07-17**  
> Purpose: to let reverse-skill **know what exists outside**, borrow on demand, **not** merge giant external libraries wholesale into this package.  
> Package identity: routing + tool bootstrap + Evidence/scope contract + field-journal (see `ops/IDENTITY.md`).

## 1. High-Value External Repositories (learn from them, do not blindly install)

| Repository | Size / positioning | Value to this package | Risk |
|------|-----------|------------|------|
| [trailofbits/skills](https://github.com/trailofbits/skills) | ToB security research Claude plugin marketplace | Quality benchmark for audit/vuln-analysis/RE plugins | Requires ToB marketplace install; do not default-trust non-curated copies |
| [trailofbits/skills-curated](https://github.com/trailofbits/skills-curated) | Reviewed plugin list | Preferred over arbitrary community skills | Same as above |
| [Orizon-eu/claude-code-pentest](https://github.com/Orizon-eu/claude-code-pentest) | 6 pentest lifecycle skills + pure Python scripts | Recon→exploit→report pipeline comparable to our `attack-chain`+`pentest-tools` | Authorization boundaries need self-check; scripts need sandboxing |
| [trilwu/secskills](https://github.com/trilwu/secskills) | 16 skills + 6 expert subagents | Multi-role division comparable to `ops/role-map.md` | Plugin form, different from this package's monorepo |
| [Masriyan/Claude-Code-CyberSecurity-Skill](https://github.com/Masriyan/Claude-Code-CyberSecurity-Skill) | ~15–19 domain skills (incl. RE/OT/CSOC) | Domain coverage checklist | Less depth than this package's single-domain skills |
| [mukul975/Anthropic-Cybersecurity-Skills](https://github.com/mukul975/Anthropic-Cybersecurity-Skills) | **800+** skills · ATT&CK/NIST mapped | **Framework mapping** and domain catalog are referenceable; do not depend on the whole repo | Far too large; huge maintenance and poisoning surface |
| [Eyadkelleh/awesome-skills-security](https://github.com/Eyadkelleh/awesome-claude-skills-security) | SecLists packaged as agent skills | Dictionary/payload entry point | Overlaps the seclists bootstrap |
| [securityfortech/awesome-security-skills](https://github.com/securityfortech/awesome-security-skills) | Curated list of security skills | Index for discovering new skills | List-type; each item needs individual audit |
| [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) | 1000+ cross-vendor skill index | Discover official/community skills | Not security-specific |
| [anthropics/claude-code-security-review](https://github.com/anthropics/claude-code-security-review) | PR security review GitHub Action | Comparable to our "change audit" scenario on the docs/report side | CI product, not an RE router |
| [agentskills.io](https://agentskills.io) | Open standard for Agent Skills | Aligns frontmatter/directory conventions | The standard itself has no offensive/defensive content |

### 1.1 Second-round search additions (re-searched 2026-07-17)

| Repository / resource | Positioning | Landing point in this package |
|-------------|------|----------|
| [trailofbits/skills](https://github.com/trailofbits/skills) plugins: `audit-context-building` `differential-review` `semgrep-rule-creator` `sharp-edges` `dwarf-expert` `burpsuite-project-parser` | Audit context, differential security review, dangerous APIs, DWARF, Burp project parsing | Compare against `ida-reverse`/`docs-generator`/audit workflows; **no** wholesale merge |
| [HexRaysSA/ida-claude-code-plugins](https://github.com/HexRaysSA/ida-claude-code-plugins) | Official IDA Claude plugins (incl. domain automation, marked unsafe) | Compare against `ida-reverse` MCP paths; unsafe plugins not enabled by default |
| [P4nda0s/reverse-skills](https://github.com/P4nda0s/reverse-skills) | IDA-NO-MCP: export decompilation then analyze; rev-frida/dex-dump/u3d | Complements "offline export when MCP is unavailable" |
| [2389-research/binary-re](https://github.com/2389-research/binary-re) | triage→static(r2/Ghidra)→dynamic(QEMU/GDB/Frida)→synthesis | `reverse-engineering` phase gates, see `re-agent-workflow.md` |
| [incogbyte/android-reverse-engineering-claude-skill](https://github.com/incogbyte/android-reverse-engineering-claude-skill) | APK unpacking, endpoint extraction, adaptive Frida bypass | Compare against `apk-reverse`; dynamic scripts need scope |
| [OwenPawl/cerberus-re-skill](https://github.com/OwenPawl/cerberus-re-skill) | Apple-oriented Ghidra+LLDB+Frida triple loop | Referenceable for the macOS/iOS dynamic loop |
| [ljagiello/ctf-skills](https://github.com/ljagiello/ctf-skills) | CTF reverse/pwn; tools installed on demand | Compare against CTF-Sandbox + `pwn-chain` |
| [shuvonsec/claude-bug-bounty](https://github.com/shuvonsec/claude-bug-bounty) | /recon→/hunt→/validate→/report | Compare against `recon-pipeline.md` + scope gate |
| [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) | Web payloads + a Prompt Injection section | `pentest-tools/payloads` first; LLM topics see `llm-security` |
| [HackTricks](https://hacktricks.wiki/) | Pentest methodology + **AI/MCP abuse** | See the skill-supply-chain MCP section |
| [appsecsanta AI pentesting agents 2026](https://appsecsanta.com/research/ai-pentesting-agents-2026) | Taxonomy of 39+ open-source AI pentest agent architectures | Multiple agents ≠ mandatory; we use role-map |
| Snyk evaluation "more skills ≠ better" | Skill stacking can degrade audit quality | Reinforces the "deep skill + routing" strategy |

## 2. Security Standards and Threats (2025–2026)

| Source | Key point | Landing point in this package |
|------|------|----------|
| [OWASP Agentic Skills Top 10](https://owasp.org/www-project-agentic-skills-top-10/) | Malicious skills, supply chain, permission abuse, memory poisoning, etc. | `ops/skill-supply-chain.md` |
| [Anthropic Agent Skills engineering post](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | Install only trusted sources; review scripts and dependencies | Same as above + bootstrap forbids guessing paths |
| Poisoning campaigns such as ClawHavoc (recorded in AST10) | Bulk malicious skills on registries | Forbidden to one-click-install from unknown registries into this package |

## 3. This Package's Coverage vs. External "Broad" Coverage

| Domain | reverse-skill | Why external packages are not merged wholesale |
|------|---------------|--------------------------------|
| APK/JS/IDA/r2/firmware/pwn | **Deep** skills + scripts | Keeps depth and tool-index binding |
| Pentest/attack chain/SRC | pentest-tools + attack-chain + src-hunter | Orizon-like repos can serve as methodology references |
| LLM/Agent security | llm-security | AST10 strengthens skill self-security |
| Evidence/scope/roles | **ops/** (distinctive) | Most skill packages have no case contract |
| OT/ICS / pure GRC / fraud F3 | No standalone skill | Route miss → propose new addition or external link; do not force-fit |
| 800+ micro skills | Not replicated | Use MASTER routing + domain skills instead of fragmentation |

## 4. Borrowing Rules (MUST)

```text
1. Forbidden to git-submodule a whole 800+-skill repo as a runtime dependency
2. When borrowing: extract "stage/checklist/command patterns" into this package's references or an existing skill
3. External scripts: inspect dependencies and network behavior in an isolated environment first, then consider bootstrap-manifest
4. New scenarios: follow CONTRIBUTING to add a skill and update routing + RULES keywords
5. Cite source URL + retrieval date (this file's format)
6. Before installing/merging, walk the ops/skill-supply-chain.md checklist
7. At runtime load only the MASTER-ROUTING PRIMARY (+ necessary secondary) to avoid skill-stacking overload
```

## 4.1 "Borrowing Artifacts" Already Sunk Into This Package (not external dependencies)

| Artifact | Path |
|------|------|
| RE four stages | `reverse-engineering/references/re-agent-workflow.md` |
| Authorized recon | `pentest-tools/references/recon-pipeline.md` |
| Attack-chain gates | `attack-chain/references/lifecycle-checklist.md` |
| Skill supply chain | `ops/skill-supply-chain.md` |
| Domain coverage | `references/domain-coverage-map.md` |

## 5. Suggested Priorities (for subsequent iterations)

| Priority | Action |
|--------|------|
| P0 done | ops contract, MASTER routing, skill supply-chain security docs |
| P1 | Compare against Orizon/ToB to add pentest stage checklists into attack-chain references |
| P2 | Optional "external skill whitelist" config, not in the default path |
