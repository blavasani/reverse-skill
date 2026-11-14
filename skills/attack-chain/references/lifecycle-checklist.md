# Penetration/Attack-Chain Lifecycle Checklist

> Integrates community pentest skill packages (such as the six-stage Orizon claude-code-pentest package) with this package's `attack-chain` and `ops`.
> Inspiration: public Claude pentest lifecycle skills (researched 2026-07); **commands and authorization are governed by this package's scope**.
> Date: 2026-07-17

## Before Use

- [ ] `case-init` is complete, with `auth.status=granted`
- [ ] `network_profile` is not misused as unrestricted access against production
- [ ] `lead` has assigned `specialist_roles` (`ops/role-map.md`)

## Stage Gates

| Stage | Role | Package skill | Completion criterion |
|------|------|------------|----------|
| 0 Scope | lead | ops/scope-contract | ready_for_act |
| 1 Recon | cie | pentest-tools | assets list + timeline |
| 2 Enum/Vuln | cpe | pentest-tools / api-security | Candidate F-* draft |
| 3 Validate | cpe | pentest-tools | E-* + validated Finding |
| 4 Post-ex (if authorized) | cpe/lead | latter half of attack-chain | Does not exceed out_of_scope |
| 5 RE Support | cre | ida/apk/js/… | Only when a client or binary is required |
| 6 Report | doc | docs-generator | Evidence→Finding→Path |
| 7 Journal | lead | field-journal | Redacted |

## Differences from “fully automated compromise from one domain” skills

| Common in external automation packages | reverse-skill |
|------------------|---------------|
| Default aggressive scanning of the domain | A scoped asset list is mandatory |
| Writing reports from weak evidence | The E/F/P chain is mandatory |
| One session without roles | role-map handoff |
| No tool index | tool-index + bootstrap |

## At Least One Timeline Entry per Stage

See the format in `ops/timeline-workitem.md`.
