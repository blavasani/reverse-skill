---
name: code-audit
description: Use for authorized source-code security review and SAST workflows including Semgrep, CodeQL patterns, dangerous API hunting, and fix verification.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Source Code Security Audit

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` or confirm authorization for code auditing
2. `NOW`: Confirm **source-code/repository access** (without source, a binary goes to the RE skill)
3. `NOW`: Define the language stack and scope (directory/service/PR diff)
4. `NEXT`: Check tool-index; semgrep and others
5. `ACT`: Threat-model sketch → automated scan → manual validation

## Applicable Scenarios

- White-box audits and PR/differential security reviews
- SAST with Semgrep / CodeQL / Bandit / gosec and similar tools
- Dangerous APIs, injection points, missing authorization, and cryptographic misuse
- Division of responsibility with `supply-chain-security/`: this skill focuses on **owned code logic**, while supply-chain security focuses on dependencies and pipelines

## Workflow

### 1. Scope and Threat Model

```text
□ Trust boundaries: user input, files, deserialization, SSRF, authorization middleware
□ High-value assets: authorization, payments, admin endpoints, and key handling
```

### 2. Automated Scanning

```bash
semgrep --config auto .
# Or use the project ruleset
semgrep --config p/owasp-top-ten .
```

### 3. Manual Validation (MUST)

```text
□ Every SAST hit: reachable? exploitable? false positive?
□ Authorization: IDOR/privilege escalation, missing validation, incorrect multi-tenant isolation
□ Injection: SQL/command/template/LDAP
□ Cryptography: hard-coded keys, ECB, custom crypto
```

### 4. Deliverables

```text
Finding: location + data flow + PoC + remediation advice
Optional ATT&CK / CWE identifier
```

## Toolchain

| Tool | Language/Scenario |
|------|------------------|
| Semgrep | Fast multi-language rules |
| CodeQL | Deep data flow (GitHub) |
| Bandit | Python |
| gosec / staticcheck | Go |
| SpotBugs / FindSecBugs | Java |

## References

- `references/sast-review-checklist.md`
- `../supply-chain-security/` `../api-security/` `../llm-security/` (agent code)

## Routing Context

**Upstream**: MASTER R26  
**Role**: `ops/role-map.md` cae  
**Downstream**: dependency vulnerabilities → supply-chain; runtime validation → pentest-tools

## Task Completion Checklist

- [ ] Was the result manually validated rather than just pasting scanner output?
- [ ] Does it include remediation advice?
- [ ] Is it limited to the authorized repository scope?
- [ ] Checklist complete?