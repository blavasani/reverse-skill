---
name: windows-ad
description: Use for authorized Active Directory and Windows identity attacks including Kerberos, AD CS, BloodHound paths, NTLM relay, and domain privilege escalation research.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Windows / Active Directory Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md`
2. `NOW`: **Define the authorized scope for domain/AD testing** (including DCs and whether poisoning/relay are allowed)
3. `NOW`: Run case-init; document the network profile and prohibited actions clearly
4. `NEXT`: Check tool-index (impacket/certipy/bloodhound are often installed manually)
5. `ACT`: Start with identity enumeration and the BloodHound graph; do not begin with destructive exploitation

## Applicable scenarios

- Domain penetration testing, Kerberoasting, AS-REP, and delegation
- AD CS certificate attacks (ESC1–ESC8 and related cases)
- BloodHound / SharpHound attack paths
- NTLM Relay / Coercer forced authentication
- Local privilege escalation into domain paths (Potato tools as pivots)

## Relationship to attack-chain

- **Multi-stage activity from the external network to the domain controller** → PRIMARY may remain `attack-chain/`; this skill is the **AD specialist**
- **Identity-focused work already inside the domain** → PRIMARY = this skill

## Workflow

### 1. Enumeration

```bash
# 示例 Impacket / 内置（需凭据与授权）
nxc smb <range> -u user -p pass
bloodhound-python -d domain.local -u user -p pass -c All -ns <DC>
```

### 2. Common paths (graph first, exploitation second)

```text
□ Kerberoast / AS-REP → 离线破解
□ ACL 滥用（GenericAll/WriteDacl）
□ 委派（非约束/约束/基于资源）
□ AD CS 模板错误 → Certipy
□ 中继：LLMNR/NBT-NS + ntlmrelayx（确认授权）
```

### 3. Credentials and lateral movement

```text
□ secretsdump / lsassy / mimikatz（严格授权与清理）
□ PtH / PtT / 黄金票仅在授权红队范围
□ 每步写 Evidence；高危等用户确认
```

## Toolchain

| Tool | Purpose |
|------|------|
| BloodHound / SharpHound | Path graph |
| Certipy | AD CS |
| Impacket / NetExec | Lateral movement and enumeration |
| Rubeus / Mimikatz | Tickets and credentials (with authorization) |
| Coercer / Responder | Forced authentication / poisoning |

## References

- `references/ad-attack-paths.md`
- `../pentest-tools/references/network-attack-defense.md`
- `../attack-chain/`
- seeds: `field-journal/seed-005_ad-certipy-esc1.md` `seed-007_ntlm-relay-coercer.md` `seed-013_kerberoasting-spn.md`

## Routing context

**Upstream**: MASTER R24
**Downstream**: Reporting via `docs-generator`; use `edr-bypass-re` for EDR research
**MUST NOT**: Run unauthorized DCSync / golden-ticket activity against production

## Completion checklist

- [ ] Did graph analysis/enumeration precede exploitation?
- [ ] Are reproducible commands recorded and sanitized?
- [ ] Were scope prohibitions followed?
- [ ] Checklist complete?