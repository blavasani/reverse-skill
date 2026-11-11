# Agent Skill Supply-Chain Security (Package Specialty)

> Sources combined: OWASP Agentic Skills Top 10 (AST10), Anthropic Agent Skills security guidance, and public poisoning incidents (such as ClawHavoc; see the AST10 timeline)
> Retrieval date: 2026-07-17
> Applies when installing, writing, or merging **any** skill, MCP, or bootstrap script

Static audit of this package's **executable-script surface** (backdoors / database deletion / pipeline execution): [`docs/PACKAGE-SECURITY-AUDIT.md`](../../docs/PACKAGE-SECURITY-AUDIT.md).

## 1. Why reverse-skill handles this separately

This package:

- Guides AI to **execute commands and download bootstrap assets**
- Reaches local resources and networks through MCP
- Writes to field-journal / reports

A malicious skill can cause credential theft, persistent prompts, and supply-chain backdoors.
We use **documentation gates + a tool source of truth**, rather than creating another skill app store.

## 2. Threat comparison (condensed AST10 approach)

| Risk class | Manifestation | Package control |
|--------|------|----------|
| Malicious/poisoned skill | Induces exfiltration or writes memory/backdoors | Trust only this repository + user-authorized external sources; manually read SKILL.md and scripts for external sources first |
| Excessive permissions | Unrestricted `curl \| bash`, full-disk reads | Bootstrap exposes manifest capabilities only; scope `network_profile` |
| Dependency poisoning | Malicious pip/npm packages | Prefer official releases; record versions in tool-index |
| Blind MCP trust | Unaudited MCP servers | tool-index registration status + port probing; do not trust remote MCP by default |
| MCP/CLI auto-execution poisoning | Repository `.env` changes such as `CODEX_HOME` cause malicious MCP execution at startup (HackTricks / CVE-style cases) | Do not trust default MCP configuration in repositories; check env and the MCP list before starting the Agent |
| Prompt injection in a skill | Concealed instructions in SKILL body | Review the diff; forbid "execution instructions hidden in HTML comments" without user approval |
| Scope drift | Skill induces broader scanning / "automatically break into an entire domain" | ops/scope-contract: out_of_scope + auth; forbid sweeping scans without in_scope |
| Skill-stack overload | Mounting too many skills at once causes missed findings (observed in public evaluations) | Load only PRIMARY + necessary secondary skills (MASTER-ROUTING) |

## 3. MUST checklist for installing external skills

```text
□ Source: official org / audited list (such as ToB curated) / user-owned
□ Read all SKILL.md + scripts/* + package dependencies
□ No mysterious outbound connections or default steps reading ~/.ssh / browser databases
□ If routing conflicts with this package: follow this package's MASTER-ROUTING + scope
□ Do not copy into a monorepo unless CONTRIBUTING and redaction requirements are followed
□ Update skills/references/community-security-skills.md with the source date
```

## 4. Boundary with bootstrap / MCP

| Action | Allowed | Prohibited |
|------|------|------|
| `bootstrap-reverse.ps1 -Capability X` | X ∈ bootstrap-manifest.json | Any new name without changing the manifest |
| Register MCP | User confirmation + tool-index refresh | Silently write global MCP configuration pointing to an unknown URL |
| Run a one-click community Python pentest | Authorized lab + source review first | Direct production targets + unknown scripts |

## 5. Package authors/contributors

- New skill: CONTRIBUTING + ACTION REQUIRED + complete self-check
- Cite community content: mark URL + date (this file / community-security-skills.md)
- If suspicious behavior is found: stop execution, inform the user, and do not automatically "try to bypass" it

## 6. Quick self-check (before each external-material merge)

```powershell
# List script extensions that will be introduced
Get-ChildItem -Recurse -Include *.ps1,*.sh,*.py,*.js | Select-Object FullName
# Broad search for dangerous patterns (manual review, not exhaustive)
# Run in the external directory: Select-String -Pattern 'Invoke-WebRequest|curl .\||wget .\||~/.ssh|exfil'
```

## 7. Related

- Identity: `IDENTITY.md`
- External directory: `../references/community-security-skills.md`
- Authorization: `scope-contract.md` + `field-journal/precedent-auth.md`
