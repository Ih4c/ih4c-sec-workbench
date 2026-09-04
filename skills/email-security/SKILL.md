---
name: email-security
description: Use for authorized email security review including phishing analysis, header authentication (SPF/DKIM/DMARC), BEC patterns, and mailbox token abuse research.
---

# Email Security & Phishing Analysis

## ACTION REQUIRED (Execute Immediately After Reading)

1. `NOW`: Confirm authorization (analyzing sample emails / tenant configuration review — per `../field-journal/precedent-pentest.md` and `../field-journal/precedent-auth.md`; scope.md is case tracking + network profile)
2. `NOW`: Do not re-deliver malicious samples to real users
3. `ACT`: Header authentication → content/URL → attachment sandbox → tenant control-plane recommendations

## Applicable Scenarios

- Phishing email dissection and IOC extraction
- SPF/DKIM/DMARC configuration assessment
- BEC (Business Email Compromise) patterns
- OAuth app phishing / mailbox token abuse (combining llm/cloud identity)
- Security awareness exercise design (authorized)

## Workflow

```text
□ Full raw headers: Received chain, From/Return-Path consistency
□ SPF/DKIM/DMARC alignment results
□ URL sandbox and attachment static analysis (combine malware-analysis)
□ Impersonated brand and reply-to address discrepancies
□ Tenant: anti-phishing policies, external tagging, MFA, OAuth app consent
```

## Toolchain

| Tool | Purpose |
|------|------|
| Email client "view source" | Headers |
| dig/nslookup | SPF/DMARC records |
| urlscan / sandbox | Links and attachments |
| Tenant admin center | Policies |

## References

- `references/email-auth-checklist.md`
- `../malware-analysis/` `../attack-chain/` (phishing stage) `../windows-ad/` (tokens)

## Routing Context

**Upstream**: MASTER R36  
**MUST NOT**: mass-test phishing against third-party domains without authorization

## Task Completion Self-Check

- [ ] Is the header-authentication conclusion complete?
- [ ] Are the IOCs detection-ready (combine threat-hunting)?
- [ ] Checklist?
