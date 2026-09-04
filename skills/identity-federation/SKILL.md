---
name: identity-federation
description: Use for authorized assessment of federated identity systems including SAML, OIDC, OAuth2 flows, SSO misconfiguration, and token confusion issues.
---

# Identity Federation (SAML / OIDC / OAuth)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read precedent-pentest; SSO test accounts and the IdP/SP scope go into scope.md (case tracking + network profile; authorization per `../field-journal/precedent-auth.md`)
2. `NOW`: Brute-force attempts that lock out real user accounts are forbidden
3. `NEXT`: Packet capture tools and documentation (metadata URLs)
4. `ACT`: Protocol flow mapping → common misconfigurations → verification

## Applicable Scenarios

- SAML Response signature/assertion tampering surface (classic flaw patterns)
- OIDC implicit/authorization-code with missing PKCE
- redirect_uri / state / nonce issues
- IdP vs SP metadata, multi-tenant issuer confusion
- Complements `api-security` JWT attacks (this skill leans toward federation and SSO flows)

## Workflow

```text
□ Draw clearly: User → SP → IdP → Token → SP
□ Collect: /.well-known/openid-configuration, SAML metadata
□ Check: redirect_uri exact match, state binding, PKCE
□ Check: SAML signature coverage, algorithm downgrade
□ Session fixation and logout invalidation
```

## Toolchain

| Tool | Purpose |
|------|------|
| Burp + SAML Raider, etc. | Assertion editing (authorized) |
| jwt_tool | JWT portions |
| Browser DevTools | Redirect chains |
| IdP admin logs | Auditing |

## References

- `references/sso-flow-checklist.md`
- `../api-security/` `../windows-ad/` (enterprise IdPs)

## Routing Context

**Upstream**: MASTER R37
**Downstream**: pure-API JWT → api-security; cloud IdP → cloud-k8s

## Task Completion Self-Check

- [ ] Full SSO flow mapped?
- [ ] Every Finding has reproduction and impact?
- [ ] Checklist?
