---
name: database-security
description: Use for authorized database security assessment covering PostgreSQL/MySQL/MSSQL/Mongo/Redis exposure, authz, UDF/command paths, and misconfiguration review.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Database Security Assessment

## ACTION REQUIRED

1. `NOW`: Read precedent-pentest; **destructive statements are prohibited on production databases** unless explicitly permitted.
2. `NOW`: Record the instance, account privileges, and whether writes/deletes are allowed.
3. `NEXT`: Identify client tool paths.
4. `ACT`: Exposure → authentication → authorization → configuration → safe exploit-chain validation.

## Use Cases

- Unauthorized database access, weak passwords, or incorrect binding to 0.0.0.0
- Excessive privileges and dangerous features (`xp_cmdshell`, `COPY PROGRAM`, `UDF`)
- Lateral movement from an application account to DBA
- NoSQL injection and Redis file writing in authorized environments

## Workflow

```text
□ Network exposure and TLS
□ Account roles and grantees
□ Sensitive-table access control
□ Dangerous configuration: file_priv, xp_cmdshell, load_file
□ Whether audit logging is enabled
□ Backup and snapshot permissions
```

## Toolchain

| Tool | Purpose |
|------|---------|
| Official CLI | Connection and enumeration |
| sqlmap | Injection validation (authorized) |
| nuclei | Known-exposure templates |
| Cloud RDS console audit | Configuration |

## References

- `references/db-misconfig-checklist.md`
- `../pentest-tools/` `../cloud-k8s/`

## Routing Context

**Upstream**: MASTER R35
**Downstream**: OS command acquisition → attack-chain; cloud-hosted services → cloud-k8s

## Completion Self-Check

- [ ] Were unauthorized writes/deletes avoided?
- [ ] Were configuration issues distinguished from exploitable chains?
- [ ] Checklist completed?
