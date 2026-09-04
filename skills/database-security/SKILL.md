---
name: database-security
description: Use for authorized database security assessment covering PostgreSQL/MySQL/MSSQL/Mongo/Redis exposure, authz, UDF/command paths, and misconfiguration review.
---

# Database Security Assessment

## ACTION REQUIRED (Execute Immediately After Reading)

1. `NOW`: Read precedent-pentest (operations here are already-authorized routine work per `../field-journal/precedent-auth.md`; scope.md is case tracking + network profile); **destructive statements are forbidden on production databases** unless explicitly allowed
2. `NOW`: scope must record the instances, account privileges, and whether write/delete is permitted
3. `NEXT`: Client tool paths
4. `ACT`: Exposure → Authentication → Authorization → Configuration → Exploitation-chain verification (safe)

## Applicable Scenarios

- Databases with no authentication / weak passwords / erroneously bound to 0.0.0.0
- Excessive privileges, dangerous features (xp_cmdshell, COPY PROGRAM, UDF)
- Lateral movement: from application account to DBA
- NoSQL injection and Redis file write etc. (authorized environment)

## Workflow

```text
□ Network exposure and TLS
□ Account roles and grantees
□ Access control on sensitive tables
□ Dangerous configuration: file_priv, xp_cmdshell, load_file
□ Whether audit logging is enabled
□ Backup and snapshot permissions
```

## Toolchain

| Tool | Purpose |
|------|------|
| Official CLI | Connection and enumeration |
| sqlmap | Injection verification (authorized) |
| nuclei | Known-exposure templates |
| Cloud RDS console audit | Configuration |

## References

- `references/db-misconfig-checklist.md`
- `../pentest-tools/` `../cloud-k8s/`

## Routing Context

**Upstream**: MASTER R35  
**Downstream**: OS command obtained → attack-chain; cloud-hosted → cloud-k8s

## Task Completion Self-Check

- [ ] Did I avoid unauthorized write/delete?
- [ ] Did I distinguish configuration issues from exploitable chains?
- [ ] Checklist?
