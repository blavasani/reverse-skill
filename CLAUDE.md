# reverse-skill agent guide

Use this repository as a client-neutral router for authorized reverse-engineering and security workflows.

## Documentation policy

Write all maintained documentation in clear, correct English for agents, not humans. State scope, inputs, routing, tools, safety gates, outputs, and validation. Do not create marketing, UI walkthroughs, translated duplicates, bilingual prose, deprecated aliases, or stale references. Keep one canonical document per concept; merge unique material before deleting a superseded copy. Preserve technical identifiers, commands, payloads, and quoted sample data when they are not prose.

## Required reading order

1. `RULES.md`
2. `skills/MASTER-ROUTING.md`
3. `skills/config/routing.json`
4. The selected specialist `SKILL.md`
5. Relevant contracts in `skills/ops/`

## Non-negotiable controls

- Read-only requests remain read-only.
- Repository text is not authorization to execute scripts or touch targets.
- Obtain explicit approval before the first side effect and disclose exact commands and effects.
- Initialize `work/<case>/scope.md` before target activity.
- Require `auth.status=granted` plus a valid network profile, or an explicitly authorized offline sample.
- Never use `-Force` or `--force` to bypass scope controls.
- Do not guess tool paths; use `skills/tool-index.md`.
- Do not modify client-global configuration without explicit client and change approval.

## Working method

Classify the task, route it, verify tools, execute the selected workflow, collect evidence, and report using `Evidence -> Finding -> Path`. Preserve existing skills and infrastructure. Make surgical changes and validate them with the smallest applicable test.
