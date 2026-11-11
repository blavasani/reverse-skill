# AI Agent Obedience Engineering — Making AI Execute a Workflow After Reading It

> Sources: 2026 multi-source synthesis (Anthropic Skill Engineering, Microsoft Code Words, Strands Steering Hooks, Gradient Flow Harness Engineering)
> Applicable scenarios: AI coding Agents (Claude Code / Codex / Cursor / Cline / Windsurf / Kiro, etc.) that only confirm after reading README/RULES.md, skip steps, or unilaterally omit key operations

---

## Core diagnosis

The root cause of an AI Agent "reading the workflow but not doing the work" is not insufficient model capability, but **semantic escape space in natural-language instructions**:

| Root cause | Explanation |
|------|------|
| **Context attention decay** | In long documents, content in the middle receives less weight from the LLM's attention mechanism; the Agent effectively only "sees" the beginning and end |
| **Semantic override** | When optimizing for "helpfulness," the model creatively reinterprets explicit instructions (for example, interpreting MUST DO X as "it is recommended to do X") |
| **Passive language treated as optional** | "Ready for next step → invoke X" is treated as a suggestion rather than an instruction |
| **Stateless enforcement** | Without an external state machine validating workflow order, the Agent can skip steps without detection |
| **Silent state corruption** | The Agent produces structurally correct but semantically wrong results, and errors accumulate silently |

---

## Technique 1: Critical-First Pattern

**Put what to do next first, and context afterward.**

```
WRONG (Agent ignores):
  [70 lines of project background and tool list]
  → "Next step: run bootstrap to install missing tools"

CORRECT (Agent executes):
  "## Execute immediately: run `bootstrap-reverse.ps1` to check and install missing tools
   → After completion, read routing.md to determine which skill to enter"
  [Then provide project background and tool list]
```

**Principle**: LLMs assign the highest attention weight to content at the beginning and end of a prompt. Content in the middle may be ignored completely.

**Application to this project**:
- The "routing entry point" section of RULES.md should come after trigger keywords and before execution principles
- The first section of each SKILL.md should be "Execute immediately," not "Applicable scenarios"

---

## Technique 2: Directive Over Suggestive

Replace all suggestive language with RFC 2119-level directive language:

| Weak language (the Agent may skip it) | Strong language (the Agent must execute it) |
|---|---|
| "You can try..." | **MUST**: You must execute... |
| "Ready for next step → invoke X" | **NOW**: Invoke X immediately; do not wait for confirmation |
| "It is recommended to read routing.md first" | **REQUIRED**: You must finish reading routing.md before entering any submodule |
| "If tools are missing, you can bootstrap" | **NO EXCUSE**: When a tool is missing, the only correct action is to invoke bootstrap; manual installation and guessing are prohibited |
| "Remember to update field-journal" | **CHECKLIST ENFORCED**: Check off the Checklist item by item when the task is complete; do not claim completion until it is finished |
| "Should..." | **MUST** / **MUST NOT** |

**Key patterns**:
```
MUST — violation = task failure
MUST NOT — violation = security violation
SHOULD — explain the reason when not done
MAY — genuinely optional
```

---

## Technique 3: Excuse Rebuttal Table

**This is the project's most important patch.** AI Agents automatically generate "reasonable excuses" to skip steps when they encounter resistance. List common excuses in advance and rebut each one:

| Common Agent excuse | Rebuttal (mandatory execution) |
|---|---|
| "This step can be skipped; I can directly..." | **Skipping is prohibited.** Every step in the behavior chain is required. If you think a step can be skipped, first state the specific reason and let the user decide. |
| "In my judgment, this is not necessary" | **Your judgment does not apply here.** List the specific criteria used for the judgment and explain why they permit skipping an explicitly written step. |
| "The user probably does not need this" | **Never decide for the user.** Present all options, mark the recommendation, but do not hide alternatives. |
| "I already know how to do this; I do not need to read X" | **Read X first, then act.** Even if you are sure how to proceed, X may contain task-specific constraints. Reading it takes only two seconds. |
| "To save time, I can skip this by running in parallel..." | **The correct way to save time is to run independent steps in parallel, not to skip steps.** If two steps are independent, do them in parallel; if dependent, do them in order. |
| "I have used this tool before and know the path" | **Guessing paths is prohibited.** You must get the actual path from tool-index; installation locations differ between machines. |
| "The task is basically complete, so I do not need the checklist" | **The only definition of task completion is that every Checklist item is checked.** A task with an incomplete Checklist is not complete. |
| "I could not find tool-index, so I will guess the path" | **A missing file is 100 times safer than a wrong guess.** When tool-index is missing, first run refresh-tool-index.ps1 to generate it. |
| "The user did not explicitly ask for a report, so I will not write one" | **A report is the default, not optional.** After a security task, always produce a report unless the user explicitly says "no report." |
| "This is too simple to record in the journal" | **Simple tasks also have lessons worth recording.** At minimum record: target type + what was used + whether anything unexpected happened; one line is enough. |
| "The user asked me to redo the import table/one step, but I did something else more useful" | **Redo = redo the named step** (or a legitimate prerequisite path confirmed by the user). MUST update the corresponding Evidence; do not substitute an unrelated step or silently skip it. Unpacking is a **prerequisite** for a readable IAT, not a substitute for import-table Evidence. |
| "The user said not to unpack a packed sample yet and to inspect the import table first; I can report the fake table as complete" | **Feasibility gate:** when X is blocked, MUST explain the blocker, recommend an order, and **ask the user to confirm**. If the user insists, execute and mark `quality=unreadable/packed`; do not use a fake table to make capability conclusions. |
| "The unpacked sample crashes, but I will keep modifying the file on disk" | **Patch 6:** Record E-self-check-crash / E-iat-repair-fail and switch to dynamic analysis (break on CreateFile/GetFileSize). Do not endlessly modify the static file. |
| "I cannot fix the IAT, so I will try several more static packer tools" | **IAT repair rule:** prioritize automatic/semi-automatic repair; if the tool errors or the repaired sample cannot run → immediately stop static IAT work, record E-iat-repair-fail, and switch to dynamic API breakpoints to capture the data. Do not endlessly retry static repair. |
| ".NET / no import table, so the hard gate does not apply; I will skip it" | **Equivalent anchor is still MUST:** for .NET, use dnSpy/IL/metadata summaries to populate the E-imports semantic slot; DLL/SYS files must also include E-exports. Do not leave it empty. |

**Usage**: Place this table near the end of RULES.md or another instruction file (a high-attention area). The Agent sees the rebuttals before looking for excuses.

---

## Technique 4: Five Skill-Engineering Patterns (Anthropic 2026 official)

| Pattern | Applicable scenario | Key technique |
|---|---|---|
| **Linear Flow** | Clear procedures (deployment, installation) | Provide safe defaults and negative instructions ("MUST NOT use --force") |
| **Decision Tree** | Platform navigation, troubleshooting | Tree navigation + progressive loading of `references/` |
| **Iterative Loop** | TDD, review-fix loops | Put hard rules first + use the **excuse rebuttal table** to block shortcuts |
| **Baton Loop** | Multi-session, multi-Agent collaboration | Externalize state to `next-prompt.md` (MUST write it before exiting) |
| **Multi-Phase + Checkpoints** | Complex workflows spanning multiple days | Orchestrator "parent" skill + human Go/No-Go checkpoints, with time cost marked |

**Mapping to this project**:
- Complete behavior chain = Linear Flow (execute 15 steps in order)
- Routing matrix = Decision Tree (three-dimensional matching)
- Checklist = Multi-Phase Checkpoint (check off every step)
- Field Journal = Baton Loop (externalize state across sessions)

---

## Technique 5: In-Band Enforcement Checks (Steering Hooks concept)

Do not rely on the AI's "self-discipline"; embed self-check instructions in the prompt:

```
Before claiming "task complete," MUST first self-check:
1. Did I skip any step in the behavior chain? Which one?
2. Did I guess any tool path? If so, what is the actual path in tool-index?
3. Are all Checklist items checked? If not, why?
4. If any answer above is "yes"/"not checked," the task is not complete.
   Return to the corresponding step and execute it again; do not claim completion.
```

This makes the Agent audit itself before saying "done," which is more immediate than external validation.

---

## Technique 6: Opaque Identifiers (Code Words) — for API/tool parameters

Microsoft 2026 research found that semantic parameter names trigger the model's tendency to "helpfully optimize."

```
WRONG: { "query": "...", "top": 9 }        → 68.4% parameter adherence
CORRECT: { "query": "...", "code": "alpha" } → 100% parameter adherence
```

**Use cases**:
- When passing exact configuration to a bootstrap script, use short codes instead of semantic parameters
- For parameters requiring a strong guarantee in tool calls, use a code-word mapping

---

## Technique 7: Dual AI Validation Loop

```
AI A (executor) writes output
  ↓
AI B (reviewer) checks against rules
  ↓ pass
Output to user
  ↓ fail
Return to AI A for correction, with specific violation citations
```

**Application in this project**:
- Embed a "self-review" step in RULES.md: before outputting a report, the Agent uses its own capabilities to check each Checklist item
- If an item is incomplete, return to the corresponding step and complete it

---

## Technique 8: Context-window layout optimization

LLM attention distribution (high → low):
```
[First 10%] ████████████ ← highest attention; put "execute immediately" instructions here
[Middle 80%] ████░░░░░░░░ ← attention decreases; put reference material here
[Final 10%] ████████████ ← attention rises again; put "do not skip" and the Checklist here
```

**Specific application**:
1. **First 10%**: immediate execution instructions + trigger keywords
2. **Middle 80%**: detailed workflow, reference links, tool list
3. **Final 10%**: excuse rebuttal table + hard Checklist + prohibited-behavior list

---

## Practical Prompt Templates

### Template A: Forced-start template (embed at the beginning of RULES.md)

```markdown
## CRITICAL: After reading this document, you must immediately execute the following operations (do not only confirm; actually execute them)

1. **NOW**: Detect the directory containing this file → this is the package root
2. **NOW**: If this is the first use, write this rule into global configuration (see the global-injection section)
3. **NEXT**: Read `skills/SKILL.md` → `skills/routing.md` → determine which sub-skill to enter
4. **NEXT**: Read `skills/tool-index.md` to confirm tool status
5. **THEN**: Start the actual task; do not remain in an "already read" state

If you only reply "read," "complete," or "I understand" without actually executing the operations above,
you have failed. The user needs tools installed, code analyzed, and vulnerabilities verified,
not a confirmation message.
```

### Template B: Submodule entry template (embed at the beginning of each SKILL.md)

```markdown
## ACTION REQUIRED (execute immediately after reading; do not wait)

After reading this file:
1. Confirm that you understand this skill's applicable scenarios
2. Check whether the required tools exist locally (read `../tool-index.md`)
3. If a tool is missing → invoke bootstrap
4. If the tools exist → start the first workflow step
5. If uncertain → list the specific questions; do not remain silent
```

### Template C: Task-completion self-check template (embed at the end of each SKILL.md)

```markdown
## Completion self-check (MUST confirm each item before claiming completion)

□ I actually executed every step in the behavior chain (nothing was skipped)
□ I did not guess any tool path (all came from tool-index.md)
□ I produced reproducible commands/scripts/report (not only a description of steps)
□ I updated the field-journal (if there were lessons)
□ I executed the post-completion Checklist (report + diagrams + lessons written back)
```

---

## Prohibited behavior (from the Agent-obedience perspective)

- Do not read RULES.md and reply only "understood; tell me the specific task"
  → Correct approach: perform global injection → read SKILL.md → read routing.md → determine the entry point
- Do not say "steps 1–4 are complete" when you only read them
  → Correct approach: distinguish "document read" from "operation executed"; the latter produces actual side effects
- Do not say "task complete" without executing the Checklist
  → The Checklist is the only definition of task completion
- Do not use "based on experience" instead of reading tool-index
  → Paths differ by machine; reading tool-index is the only reliable way to locate them

---

## Summary: If you can change only one thing

**Add an "execute immediately" instruction block at the beginning of RULES.md**, using strong directive words such as bold, CRITICAL, and NOW.

This has the highest return on investment. Most Agent "inaction" comes from automatically entering a "wait for the user's instructions" mode after reading a file. A forced-action block can break that pattern.

If you can change a second thing: **add the excuse rebuttal table**. An Agent looks for an excuse to stop at the first obstacle; block those excuses in advance.
