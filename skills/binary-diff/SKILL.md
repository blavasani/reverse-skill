---
name: binary-diff
description: "Use the binary-diff specialist workflow after routing and authorization. Follow its ACTION REQUIRED steps and preserve evidence provenance."
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Cross-Version Symbol Migration (Binary Diff)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm this skill's actions are authorized routine operations
2. `NOW`: Confirm that the current task matches this skill's scope
3. `NEXT`: Read `../tool-index.md` and verify tool availability and actual paths
4. `NEXT`: Invoke bootstrap when a tool is missing; do not guess paths
5. `ACT`: Enter and execute the first workflow step; do not stop in a confirmation state

## Scope

Use this skill when the task matches one of these scenarios:

1. **Kernel/driver missing PDB** — symbols exist for an older ntoskrnl.exe, the newer PDB was removed by Microsoft, and older symbols are needed to infer newer non-exported function addresses
2. **Symbol migration after a program update** — a program was previously reverse-engineered, then updated; migrate the old results in bulk instead of reversing it again
3. **Protection update** — the old version has complete reverse-engineering results and the new version needs rapid location of the same function's new offset
4. **Any binary-comparison scenario with old symbols and a new symbol-less build**

### Division of Responsibility with Other Skills

| Scenario | Use |
|------|--------|
| Reverse-engineer a binary from scratch | `ida-reverse/` or `radare2/` |
| Migrate old results to a new version | **this skill** |
| Compare two completely different binaries | BinDiff / Diaphora (traditional tools) |

### Core Advantages

Compared with traditional approaches:

| Approach | Cost for 200 functions | Time | Accuracy |
|------|--------------|------|--------|
| Manually compare two IDA windows | Free but exhausting | Several hours | High |
| BinDiff automatic matching | Free | Fast | Medium (fails when structure changes substantially) |
| Delegate entirely to an agent (CC/Codex) | 50-100 yuan | Slow | High |
| **This skill (batch LLM comparison)** | **~1 yuan** | **~10 seconds/function** | **High** |

## Core Principle

```text
Older function (with symbols)       Same function in newer version (without symbols)
          ↓                                      ↓
Exported disassembly + pseudocode     Exported disassembly + pseudocode
          ↓                                      ↓
          └──────── Structured LLM comparison ────────┘
                              ↓
                   Output YAML (symbol mapping table)
                              ↓
                   Programmatic parsing → batch application to the new IDB
```

Key points:
- The prompt is a fixed template filled programmatically
- Input and output formats are fixed and parsed programmatically
- The LLM only handles "looking at two code fragments and finding the correspondence"
- Time and token costs are very low

## Prompt Template

### Standard Comparison Prompt

```text
I have disassembly outputs and procedure code of the same function.

This is the function for reference:

**Disassembly for Reference**
```c
{disasm_for_reference}
```

**Procedure code for Reference**
```c
{procedure_for_reference}
```

This is the function you need to reverse-engineering:

**Disassembly to reverse-engineering**
```c
{disasm_code}
```

**Procedure code to reverse-engineering**
```c
{procedure}
```

What you need to do is to collect all references to "{symbol_name_list}" in the function you need to reverse-engineering and output those references as YAML.

Example:
```yaml
found_vcall: # This is for indirect call to virtual function or virtual function pointer fetching.
  - insn_va: '0x180777700' # Always be the instruction with displacement offset
    insn_disasm: call [rax+68h] # Always be the instruction with displacement offset
    vfunc_offset: '0x68'
    func_name: ILoopMode_OnLoopActivate
  - insn_va: '0x180777778' # Always be the instruction with displacement offset
    insn_disasm: mov rax, [rax+80h] # Always be the instruction with displacement offset
    vfunc_offset: '0x80'
    func_name: INetworkMessages_GetNetworkGroupCount

found_call: # This is for direct call to non-virtual regular function.
  - insn_va: '0x180888800'
    insn_disasm: call sub_180999900
    func_name: CLoopMode_RegisterEventMapInternal
  - insn_va: '0x180888880'
    insn_disasm: call sub_180555500
    func_name: CLoopMode_SetSystemState

found_funcptr: # This is for non-virtual regular function pointer.
  - insn_va: '0x180666600' # Must load/reference the function pointer target address
    insn_disasm: lea rdx, sub_15BC910 # Must load/reference the function pointer target address
    funcptr_name: CLoopMode_OnClientPollNetworking

found_gv: # This is for reference to global variable.
  - insn_va: '0x180444400'
    insn_disasm: mov rcx, cs:qword_180666600 # Must load/reference the global variable
    gv_name: g_pNetworkMessages
  - insn_va: '0x180333300'
    insn_disasm: lea rax, unk_180222200 # Must load/reference the global variable
    gv_name: s_EventManager

found_struct_offset: # This is for reference to struct offset. NOTE THAT virtual function pointer should not be here! virtual function pointer should ALWAYS be in found_vcall !
  - insn_va: '0x1801BA12A' # Always be the instruction with displacement offset
    insn_disasm: mov rcx, [r14+58h] # Always be the instruction with displacement offset
    offset: '0x58'
    size: 8
    struct_name: CResourceService
    member_name: m_pEntitySystem
```

If nothing found, output an empty YAML. DO NOT output anything other than the desired YAML. DO NOT collect unrelated symbols.
```

### Variable Reference

| Variable | Source | Description |
|------|------|------|
| `{disasm_for_reference}` | Export from older IDA | Disassembly with symbols |
| `{procedure_for_reference}` | Export from older IDA | Pseudocode with symbols |
| `{disasm_code}` | Export from newer IDA | Disassembly without symbols |
| `{procedure}` | Export from newer IDA | Pseudocode without symbols |
| `{symbol_name_list}` | Extracted from older version | Symbols to locate in the newer version |

## Workflow

### Complete Process

```text
Step 1: Prepare data
  - Load the older binary into IDA (with PDB/symbols)
  - Load the newer binary into IDA (without symbols)
  - Find matching anchor functions in both versions (exported functions, string references, and so on)

Step 2: Batch export
  - Export from the older version: anchor-function disassembly + pseudocode (with symbol names)
  - Export from the newer version: the same anchor-function disassembly + pseudocode (without symbol names)

Step 3: LLM comparison
  - Fill the data into the prompt template
  - Call the LLM API (recommended: deepseek for large, inexpensive workloads; switch to gpt for very large functions)
  - Parse the returned YAML

Step 4: Apply results
  - Apply the YAML symbol mappings in bulk to the newer IDB
  - Batch-rename with idapro_rename or an IDAPython script

Step 5: Iterate
  - Functions migrated in the first round become new anchors
  - Enter those functions and continue comparing internal calls
  - Repeat until all target functions are covered
```

### Anchor Selection Strategy

| Anchor Type | Reliability | Description |
|---------|--------|------|
| Exported function | Highest | Name stays the same; address may change |
| String reference | High | String content stays the same; reference location may change |
| Constant/magic number | Medium | Characteristic value stays the same |
| Code pattern | Medium | Function structure is similar but all addresses change |

### Batch Processing Recommendations

- Compare one function at a time (avoid context overflow)
- Use deepseek for medium functions (<200 lines)
- Use gpt-4o or claude for very large functions (>500 lines)
- Use concurrent calls for speed (10-20 concurrent)
- Cache results to avoid duplicate calls

## Output Format

### Five Symbol Types in YAML Output

| Type | Meaning | Key Fields |
|------|---------|------------|
| `found_vcall` | Virtual-function call (indirect call) | `vfunc_offset`, `func_name` |
| `found_call` | Direct function call | `insn_va`, `func_name` |
| `found_funcptr` | Function-pointer reference | `insn_va`, `funcptr_name` |
| `found_gv` | Global-variable reference | `insn_va`, `gv_name` |
| `found_struct_offset` | Struct-offset reference | `offset`, `struct_name`, `member_name` |

### Application Actions After Parsing

```text
found_call → idapro_rename(addr=call_target, name=func_name)
found_vcall → idapro_set_comments(addr=insn_va, comment="vcall: {func_name} @ +{offset}")
found_funcptr → idapro_rename(addr=funcptr_target, name=funcptr_name)
found_gv → idapro_rename(addr=gv_addr, name=gv_name)
found_struct_offset → idapro_set_comments(addr=insn_va, comment="{struct_name}.{member_name}")
```

## Typical Scenario Examples

### Scenario 1: ntoskrnl.exe Missing PDB

```text
Available: ntoskrnl.exe 10.0.26100.2000 + complete PDB
Target: ntoskrnl.exe 10.0.26100.2605 (PDB removed)
Need: locate the new address of PspSetCreateProcessNotifyRoutine

Steps:
1. Load both versions into IDA
2. Find the exported function PsSetCreateProcessNotifyRoutine (present in both versions)
3. In the older version it calls PspSetCreateProcessNotifyRoutine (with symbols)
4. In the newer version it calls sub_140822108 (without symbols)
5. The LLM immediately identifies: sub_140822108 = PspSetCreateProcessNotifyRoutine
6. Apply in bulk
```

### Scenario 2: Migration After an Application Update

```text
Available: complete reverse-engineering results for target.exe v1.0 (200+ functions named)
Target: target.exe v1.1 (all symbols lost)
Need: migrate 200 function names in bulk

Steps:
1. Export disassembly and pseudocode for all named functions from the older version
2. Find corresponding anchors in the newer version through exports/strings
3. Call the LLM comparison in bulk
4. Parse YAML and batch-rename
5. Iterate deeper
```

## LLM Selection Recommendations

| Model | Suitable Scenario | Cost | Speed |
|------|-------------------|------|------|
| DeepSeek V3 | Small and medium functions (<200 lines), batch processing | Very low | Fast |
| GPT-4o | Very large functions, complex control flow | Medium | Fast |
| Claude Sonnet | Medium and large functions requiring reasoning | Medium | Fast |
| Claude Opus | Extremely complex functions requiring deep understanding | High | Slow |

Recommended strategy: use DeepSeek by default; upgrade automatically when context limits are exceeded or results are inaccurate.

## Notes

- **Do not give the entire binary to the LLM** — compare one function at a time
- **Anchors must be reliable** — if the anchor is wrong, all subsequent work is wasted
- **Manually spot-check results** — the LLM is not 100% accurate; verify critical symbols
- **Cache intermediate results** — avoid duplicate calls and wasted tokens
- **Mind context limits** — very large functions (>1000 lines of disassembly) need splitting or a large-context model

---

## On-Demand Bootstrap

### Tool Dependencies

| Tool | Purpose | Auto-installable |
|------|---------|------------------|
| IDA Pro | Export disassembly/pseudocode | ✗ (commercial software) |
| Python | Run scripts and call APIs | ✓ |
| PyYAML | Parse YAML returned by the LLM | ✓ (pip install pyyaml) |
| LLM API | Perform comparisons | API key required |

### Notes

The core of this skill does not depend on installing heavyweight tools; it mainly requires:
- Existing IDA Pro (managed with the `ida-reverse/` skill)
- Python + requests/httpx (for API calls)
- An LLM API endpoint

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (controller), `routing.md`
**Trigger condition**: old symbols/reverse-engineering results need to be migrated to a newer version
**Downstream exits**:
- Need to open the binary first → `ida-reverse/`
- Need rapid reconnaissance to confirm version differences → `radare2/`

**Related peer module**: `ida-reverse/` (IDA handles both data export and symbol application)

## Task Completion Checklist (MUST pass before claiming completion)

- [ ] Did I execute every workflow step rather than only read it?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
