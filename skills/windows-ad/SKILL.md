---
name: windows-ad
description: Use for authorized Active Directory and Windows identity attacks including Kerberos, AD CS, BloodHound paths, NTLM relay, and domain privilege escalation research.
---

# Windows / Active Directory Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md`
2. `NOW`: **Domain/AD testing requires an explicit authorization scope** (including DCs, whether poisoning/relaying is allowed)
3. `NOW`: case-init; write the network_profile and forbidden actions clearly into scope.md (scope.md is case tracking + network profile; authorization per `../field-journal/precedent-auth.md`)
4. `NEXT`: tool-index (impacket/certipy/bloodhound, etc., often manual)
5. `ACT`: Start with identity enumeration and the BloodHound graph, not destructive exploitation

## Applicable Scenarios

- Domain penetration, Kerberoasting, AS-REP, delegation
- AD CS (ESC1–ESC8, etc.) certificate attacks
- BloodHound / SharpHound attack paths
- NTLM Relay / Coercer coerced authentication
- Local privilege escalation to domain paths (Potato, etc., as a foothold)

## Relationship to attack-chain

- **Multi-stage from the perimeter to the domain controller** → PRIMARY can remain `attack-chain/`; this skill is the **AD specialty**
- **Already in the domain, focused on identity** → PRIMARY = this skill

## Workflow

### 1. Enumeration

```bash
# Example Impacket / built-in (requires credentials and authorization)
nxc smb <range> -u user -p pass
bloodhound-python -d domain.local -u user -p pass -c All -ns <DC>
```

### 2. Common Paths (graph first, then guns)

```text
□ Kerberoast / AS-REP → offline cracking
□ ACL abuse (GenericAll/WriteDacl)
□ Delegation (unconstrained/constrained/resource-based)
□ AD CS template misconfigurations → Certipy
□ Relay: LLMNR/NBT-NS + ntlmrelayx (confirm authorization)
```

### 3. Credentials and Lateral Movement

```text
□ secretsdump / lsassy / mimikatz (strict authorization and cleanup)
□ PtH / PtT / golden tickets only within an authorized red-team scope
□ Record Evidence at every step; wait for user confirmation on high-risk actions
```

## Toolchain

| Tool | Purpose |
|------|------|
| BloodHound / SharpHound | Path mapping |
| Certipy | AD CS |
| Impacket / NetExec | Lateral movement and enumeration |
| Rubeus / Mimikatz | Tickets and credentials (authorized) |
| Coercer / Responder | Coerced authentication / poisoning |

## References

- `references/ad-attack-paths.md`
- `../pentest-tools/references/network-attack-defense.md`
- `../attack-chain/`
- seeds: `field-journal/seed-005_ad-certipy-esc1.md` `seed-007_ntlm-relay-coercer.md` `seed-013_kerberoasting-spn.md`

## Routing Context

**Upstream**: MASTER R24  
**Downstream**: report `docs-generator`; EDR research needed → `edr-bypass-re`  
**MUST NOT**: unauthorized DCSync / golden tickets against production

## Task Completion Self-Check

- [ ] Was there a graph/enumeration before exploitation?
- [ ] Were reproducible commands recorded and desensitized?
- [ ] Were the scope's forbidden items respected?
- [ ] Checklist?
