---
name: email-security
description: Use for authorized email security review including phishing analysis, header authentication (SPF/DKIM/DMARC), BEC patterns, and mailbox token abuse research.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Email Security & Phishing Analysis

## ACTION REQUIRED

1. `NOW`: Confirm authorization (analyze sample emails / review tenant configuration).
2. `NOW`: Do not resend malicious samples to real users.
3. `ACT`: Header authentication → content/URLs → attachment sandbox → tenant control-plane recommendations.

## Use Cases

- Phishing-email deconstruction and IOC extraction
- SPF/DKIM/DMARC configuration assessment
- Business email compromise (BEC) patterns
- OAuth application phishing / mailbox-token abuse (combined with llm/cloud identity)
- Authorized security-awareness exercise design

## Workflow

```text
□ Complete raw headers: Received chain and From/Return-Path consistency
□ SPF/DKIM/DMARC alignment results
□ URL sandboxing and static attachment analysis (combined with malware-analysis)
□ Brand impersonation and reply-address differences
□ Tenant controls: anti-phishing policy, external markers, MFA, OAuth app consent
```

## Toolchain

| Tool | Purpose |
|------|---------|
| Email client “View Source” | Headers |
| dig/nslookup | SPF/DMARC records |
| urlscan / sandbox | Links and attachments |
| Tenant administration center | Policies |

## References

- `references/email-auth-checklist.md`
- `../malware-analysis/` `../attack-chain/` (phishing phase) `../windows-ad/` (tokens)

## Routing Context

**Upstream**: MASTER R36
**MUST NOT**: Bulk-send phishing tests to third-party domains without authorization.

## Completion Self-Check

- [ ] Are header-authentication conclusions complete?
- [ ] Are IOCs detection-ready (combined with threat-hunting)?
- [ ] Checklist completed?
