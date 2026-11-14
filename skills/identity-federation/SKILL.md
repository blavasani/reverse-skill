---
name: identity-federation
description: Use for authorized assessment of federated identity systems including SAML, OIDC, OAuth2 flows, SSO misconfiguration, and token confusion issues.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Identity Federation (SAML / OIDC / OAuth)

## ACTION REQUIRED (Execute immediately after reading)

1. `NOW`: Read precedent-pentest; add SSO test accounts and IdP/SP scope to scope
2. `NOW`: Do not use brute-force attempts that could lock real user accounts
3. `NEXT`: Packet-capture tools and documentation (metadata URL)
4. `ACT`: Map the protocol flow → identify common misconfigurations → validate

## Applicable scenarios

- SAML Response signature/assertion tampering surface (classic vulnerability pattern)
- Missing PKCE for OIDC implicit/authorization-code flows
- redirect_uri / state / nonce issues
- IdP and SP metadata or multi-tenant issuer confusion
- Complements `api-security` JWT attacks (this skill focuses on federation and SSO flows)

## Workflow

```text
□ 画清：User → SP → IdP → Token → SP
□ 收集：/.well-known/openid-configuration、SAML metadata
□ 检查：redirect_uri 精确匹配、state 绑定、PKCE
□ 检查：SAML 签名覆盖范围、algorithm 降级
□ 会话固定与登出失效
```

## Toolchain

| Tool | Purpose |
|------|------|
| Burp + SAML Raider, etc. | Assertion editing (authorized) |
| jwt_tool | JWT segments |
| Browser DevTools | Redirect chain |
| IdP administration logs | Auditing |

## References

- `references/sso-flow-checklist.md`
- `../api-security/` `../windows-ad/` (enterprise IdP)

## Routing context

**Upstream**: MASTER R37
**Downstream**: Pure API JWT → api-security; cloud IdP → cloud-k8s

## Completion checklist

- [ ] Is the complete SSO flow mapped?
- [ ] Does each Finding include reproduction steps and impact?
- [ ] Checklist?