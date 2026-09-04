# Penetration / Attack Chain Lifecycle Checklist

> Consolidates community pentest skill packs (e.g., the six phases of Orizon claude-code-pentest) with this package's `attack-chain` + `ops`.  
> Source inspiration: public Claude pentest lifecycle skills (retrieved 2026-07); **commands and authorization are governed by this package's scope**.  
> Date: 2026-07-17

## Before Use

- [ ] `case-init` completed, `auth.status=granted`
- [ ] `network_profile` ≠ mistakenly using unrestricted against production
- [ ] `lead` has assigned specialist_roles (`ops/role-map.md`)

## Phase Gates

| Phase | Role | This package's skill | Completion criteria |
|------|------|------------|----------|
| 0 Scope | lead | ops/scope-contract | ready_for_act |
| 1 Recon | cie | pentest-tools | assets list + timeline |
| 2 Enum/Vuln | cpe | pentest-tools / api-security | draft candidate F-* |
| 3 Validate | cpe | pentest-tools | E-* + validated Finding |
| 4 Post-ex (if authorized) | cpe/lead | second half of attack-chain | do not exceed out_of_scope |
| 5 RE support | cre | ida/apk/js/… | only when client/binary analysis is needed |
| 6 Report | doc | docs-generator | Evidence→Finding→Path |
| 7 Journal | lead | field-journal | anonymized |

## Differences from "fully automated: give it a domain and pwn it" type skills (differentiators)

| Common in external automation packages | reverse-skill |
|------------------|---------------|
| Scans a domain aggressively by default | Scope asset list is mandatory |
| Weak evidence goes straight into the report | E/F/P chain is enforced |
| Single session, no roles | Handoff via role-map |
| No tool index | tool-index + bootstrap |

## At Least One Timeline Entry Per Phase

Format per `ops/timeline-workitem.md`.
