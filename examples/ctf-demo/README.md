# examples/ctf-demo — Complete Workflow Example

> This directory demonstrates the standard reverse-skill workflow: **routing → authorization gate → timeline → evidence chain → report**.
> The content is a fictional example (CTF lab) intended only to demonstrate the workflow.

## Workflow Demonstration

```text
1. User task: "Analyze this CTF pwn challenge; it has a stack overflow."
2. Routing: `master-route.ps1 -Hint "CTF pwn stack overflow"` → PRIMARY R17 (`pwn-chain`)
3. Authorization: `case-init.ps1 -Hint ... -CaseName ctf-demo -AuthGranted` → `scope.md`
4. Execution: append to the timeline, record evidence `E-001` and `E-002`, and update work items
5. Deliverables: a report from `docs-generator` and a sanitized field-journal entry
```

## Files

| File | Description |
|------|-------------|
| `scope.md` | Case scope (auth granted / target / network_profile) |
| `timeline.md` | Append-only timeline |
| `workitems.md` | Work items and coverage |
| `evidence/` | Example evidence records (E-001 reproduction command, E-002 crash output) |
| `report/` | Example final-report structure |

## Real Usage

```powershell
# 初始化真实 case（授权目标）
powershell -NoProfile -ExecutionPolicy Bypass -File skills/scripts/case-init.ps1 `
  -Hint "你的任务" -CaseName my-case -AuthGranted -TargetUrl "https://target/" `
  -NetworkProfile authorized_target_only

# 追加证据
powershell -File skills/scripts/append-evidence.ps1 -CaseRoot work\my-case `
  -Id E-001 -Title "..." -ReproCommand "..."
```

> Note: Real cases should be stored under `work/<case>/` (gitignored to prevent data leakage); this example directory remains in Git for reference.
