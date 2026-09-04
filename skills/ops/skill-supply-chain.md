# Agent Skill Supply Chain Security (this package's flavor)

> Sources synthesized from: OWASP Agentic Skills Top 10 (AST10), Anthropic Agent Skills security recommendations, and public poisoning incidents (e.g. ClawHavoc, see the AST10 timeline)  
> Retrieval date: 2026-07-17  
> Applies to: installing/writing/merging **any** skill, MCP, or bootstrap script

Static audit of this package's **executable script surface** (backdoors / destructive commands / pipe execution): [`docs/PACKAGE-SECURITY-AUDIT.md`](../../docs/PACKAGE-SECURITY-AUDIT.md).

## 1. Why reverse-skill Manages This Separately

This package will:

- Direct AI to **execute commands and bootstrap downloads**
- Touch local and network resources through MCP  
- Write to field-journal / reports  

Malicious skills can cause: credential theft, persistence prompts, supply chain backdoors.  
We use **document gates + a tool-truth source**, rather than building another skill app store.

## 2. Threat Mapping (condensed AST10 thinking)

| Risk class | Manifestation | This package's control |
|--------|------|----------|
| Malicious/poisoned skill | induce exfiltration, write memory/backdoors | trust only this repo + user-verbally-authorized external sources; for external sources, read the SKILL.md and scripts first |
| Excessive permissions | indiscriminate `curl \| bash`, full-disk reads | bootstrap covers manifest capabilities only; scope `network_profile` |
| Dependency poisoning | malicious pip/npm packages | prefer official releases; record versions in tool-index |
| Blind MCP trust | unaudited MCP servers | tool-index registration status + port probing; do not trust remote MCP by default |
| Auto-execution poisoning via MCP/CLI | repo `.env` tampering with `CODEX_HOME` etc. causes malicious MCP to execute at startup (HackTricks / CVE-style cases) | do not trust default MCP configs inside repos; check env and the MCP list before starting agents |
| Prompt injection into skills | hidden instructions buried in SKILL body | review diffs; no "execution instructions hidden in HTML comments" without the user |
| Scope drift | skill induces broader scanning / "fully auto-pwn one domain" | ops/scope-contract: out_of_scope + auth; no wild scanning without in_scope |
| Skill-stacking overload | mounting too many skills at once increases misses (public eval observations) | load only PRIMARY + necessary secondary (MASTER-ROUTING) |

## 3. MUST Checklist for Installing External Skills

```text
□ Source: official org / audited list (e.g. ToB curated) / user-owned
□ Read all of SKILL.md + scripts/* + package dependencies
□ No mysterious external connections, no default steps reading ~/.ssh / browser stores
□ When conflicting with this package's routing: this package's MASTER-ROUTING + scope wins
□ Do not copy into the monorepo unless going through CONTRIBUTING and sanitization
□ Update skills/references/community-security-skills.md to record the source date
```

## 4. Boundaries with bootstrap / MCP

| Action | Allowed | Forbidden |
|------|------|------|
| `bootstrap-reverse.ps1 -Capability X` | X ∈ bootstrap-manifest.json | arbitrary new names without changing the manifest |
| Registering an MCP | user confirmation + tool-index refresh | silently writing a global MCP pointing at an unknown URL |
| Running community one-shot pentest Python | authorized lab + after reading the source | production targets directly + unknown scripts |

## 5. This Package's Authors/Contributors

- New skills: CONTRIBUTING + ACTION REQUIRED + completion self-checks  
- Citing community content: mark URL + date (this file / community-security-skills.md)  
- On suspicious behavior: stop execution, inform the user, do not automatically "try to bypass"

## 6. Quick Self-Check (before every merge of external material)

```powershell
# List the script extensions about to be introduced
Get-ChildItem -Recurse -Include *.ps1,*.sh,*.py,*.js | Select-Object FullName
# Coarse search for dangerous patterns (human review required, not exhaustive)
# Run inside the external directory: Select-String -Pattern 'Invoke-WebRequest|curl .\||wget .\||~/.ssh|exfil'
```

## 7. Related

- Identity: `IDENTITY.md`  
- External directory: `../references/community-security-skills.md`  
- Authorization: `scope-contract.md` + `field-journal/precedent-auth.md`  
