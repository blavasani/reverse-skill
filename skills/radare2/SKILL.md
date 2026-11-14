---
name: radare2
description: |
  Use this skill whenever the user wants to analyze binaries with radare2/r2 from the command line, including reverse engineering, disassembly, function analysis, strings/import inspection, patching, binary diffing, hex inspection, or r2 scripting. Also use it when the user mentions PE/ELF/Mach-O/DEX/WASM files together with CLI analysis, `rabin2`, `rasm2`, `radiff2`, `r2pipe`, or asks for radare2 command help on Windows/Linux/macOS.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# radare2

Binary analysis using the `radare2` CLI for reconnaissance, analysis, locating and exporting data, and lightweight modifications without a GUI.

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` and confirm the skill's operations are authorized routine operations.
2. `NOW`: Confirm the task is within this skill's scope.
3. `NEXT`: Read `../tool-index.md` and verify tool availability and actual paths.
4. `NEXT`: Use bootstrap when tools are missing; do not guess paths.
5. `ACT`: Execute the first workflow step; do not stop after confirmation.

## Scope

Prefer this skill when the user wants to:

- Analyze files such as `exe`, `dll`, `so`, `elf`, `apk`, `dex`, or `wasm` with `r2` / `radare2`.
- Learn how to use `rabin2`, `rasm2`, `radiff2`, `rahash2`, or `rax2`.
- Disassemble from the command line, inspect functions, strings, imports and exports, check cross-references, or patch a binary.
- Write `radare2` batch commands, `-c` automation commands, or `r2pipe` scripts.

Prefer `ida-reverse` for GUI reverse engineering, Hex-Rays-style pseudocode, or an IDA workflow. Prefer `reverse-engineering` for web JS reverse engineering.

## Check the environment first

Do not assume `r2` is available. First check:

```powershell
r2 -v
rabin2 -v
```

If it is not installed, check common installation locations or prompt for installation.

Common Windows executables:

- `radare2.exe`
- `rabin2.exe`
- `rasm2.exe`
- `radiff2.exe`
- `rahash2.exe`
- `rax2.exe`
- `r2pm.exe`

## Included resources

Reuse the two resources included with this skill rather than recreating equivalent command sequences.

### `scripts/recon.ps1`

The standard reconnaissance script provides an initial overview and outputs:

- Basic information
- Sections
- Imports
- Exports
- Strings
- An optional `r2 -A` automatic analysis summary

Run:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "C:\path\to\sample.exe"
```

To include `r2` automatic analysis:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "C:\path\to\sample.exe" -RunAnalysis
```

### `references/cheatsheet.md`

Read this cheatsheet for command details, common scenario templates, or syntax reminders instead of guessing.

## Known behavior

### Intermittent missing `.sdb` warnings on Windows

Reconnaissance of some PE files with `rabin2` may produce a warning such as:

```text
ERROR: Cannot find ...\share\format\dll\*.sdb
```

If the main output is otherwise normal, this usually does not affect basic reconnaissance conclusions; continue the analysis. Do not treat this warning alone as an analysis failure.

## Principles

### 1. Reconnaissance before deeper analysis

Do not start with full automatic analysis. Use lightweight commands to check the file type, architecture, entry point, strings, and import table before deciding whether to run `aaa`, `aaaa`, or targeted analysis.

### 2. Prefer the minimum sufficient commands

Among the many `radare2` commands, choose the shortest path that meets the need:

- File information: `rabin2 -I`
- Strings: `rabin2 -z`
- Imports and exports: `rabin2 -i` / `rabin2 -E`
- Interactive analysis: `r2 <file>`, followed by targeted commands.

### 3. Exercise caution before modification

If the user requests a binary patch:

- Open read-only by default: `r2 <file>`.
- Use write mode only when modification is explicitly required: `r2 -w <file>` or `oo+` within the session.
- Explain the risks before modifying the file and avoid accidentally overwriting the original.

## Common workflows

## Workflow 1: Quick reconnaissance

Use for the initial examination of a binary.

### Hard gate (MUST satisfy before workflow 2 or later)

For binaries with import tables, such as PE/ELF/Mach-O, **MUST** inspect the import table and record Evidence before function-level analysis or dynamic steps:

1. Run `rabin2 -i <sample>` (or use the imports section of `recon.ps1` output); for DLL/SYS, also MUST run `rabin2 -E` and record `E-exports`.
2. Record the complete or categorized import table in Evidence (suggested id: `E-imports` or `E-triage-imports`), including at least:
   - The reproduction command (`repro_command`).
   - A summary of key imports by category: network / file / cryptography / process injection / registry / other suspicious APIs.
   - If the import table is empty, parsing fails, or the tool reports an error, MUST still record the failure and raw output as Evidence; **do not silently skip it**.
   - If the import table is unusually clean (only basic DLLs), MUST note suspected dynamic loading and SHOULD move to dynamic API capture.
3. For .NET and others without a traditional IAT, MUST record an equivalent anchor (dnSpy/IL/metadata summary) in the same Evidence semantic slot; do not skip it.
4. For packed sample IAT repair, use ImportREC (or equivalent) for x86 and Scylla (or equivalent) for x64. If repair fails, MUST record `E-iat-repair-fail` and move to dynamic API breakpoints; **do not** persist indefinitely with the static IAT (see `reverse-engineering/references/re-agent-workflow.md` §1.2).
5. When the user explicitly requests a repeat import-table check or IAT redo, MUST repeat the named step itself. If blocked, first apply the feasibility gate: explain the prerequisites and request confirmation; if forced, mark quality=unreadable. **Do not substitute an unrelated step and claim completion**.

Before recording import-table Evidence (or a valid equivalent anchor / IAT failure bypass), MUST NOT claim basic reconnaissance is complete or proceed to deeper conclusions in workflow 2 or later.

Prefer running the included script directly:

```powershell
powershell -File "<skill-root>\radare2\scripts\recon.ps1" -TargetPath "sample.exe"
```

If only minimal manual commands are needed, use:

```powershell
rabin2 -I sample.exe
rabin2 -z sample.exe
rabin2 -i sample.exe
rabin2 -E sample.exe
```

Inspect:

- File format, bitness, architecture, and platform.
- Entry point address.
- Suspicious strings: URLs, paths, errors, registry entries, and command-line arguments.
- Imported functions: network, file, cryptography, process injection, and registry operations (**MUST record Evidence; see the hard gate above**).

## Workflow 2: Interactive function analysis

```powershell
r2 sample.exe
```

Common commands after opening:

```text
aaa          # 常规自动分析
afl          # 列出函数
iz           # 列出字符串
iS           # 列节区
is           # 列符号
s entry0     # 跳到入口点
pdf          # 反汇编当前函数
VV           # 进入可视化模式（如果终端适合）
q            # 退出
```

Notes:

- Prefer `aaa` by default; do not start with the heavier `aaaa`.
- If the sample is large or analysis is slow, analyze only the area near the entry point and expand manually.

## Workflow 3: Locate main / key logic

```text
afl~main
afl~sym.
iz~http
iz~error
axt <addr>
```

Approach:

- Start with `main`, the entry point, and string references.
- Use `axt` to find references to a string or address.
- After finding a reference, use `s <addr>` and `pdf`.

## Workflow 4: Hexadecimal and memory inspection

```text
px 64        # 当前地址起 64 字节十六进制
pd 20        # 反汇编 20 条指令
psz          # 读取当前地址字符串
pxa          # 更友好的十六进制视图
```

## Workflow 5: Binary patching

Use only when the user explicitly requests file modification:

```powershell
r2 -w sample.exe
```

For example, after opening:

```text
s 0x401000
wa nop
wa jmp 0x401050
wq
```

Common write operations:

- `wa <asm>`: write assembly.
- `wx <hex>`: write raw bytes.
- `wq`: write and exit.

Prefer backing up the original before modification. If the user did not mention a backup, remind them at least once.

## Workflow 6: Non-interactive automation

Use for one-time output:

```powershell
r2 -A -q -c "afl;iz;ii;q" sample.exe
```

Common options:

- `-A`: automatic analysis on startup.
- `-q`: quiet mode.
- `-c`: execute a command sequence.

When many commands are needed, arrange them in a readable order instead of creating an excessively long, hard-to-maintain sequence.

Prefer starting with the included reconnaissance script, then decide whether custom commands are needed.

## Common tools

### `rabin2`

For static information extraction:

```powershell
rabin2 -I sample.exe   # 基本信息
rabin2 -S sample.exe   # 节区
rabin2 -s sample.exe   # 符号
rabin2 -i sample.exe   # 导入
rabin2 -E sample.exe   # 导出
rabin2 -z sample.exe   # 字符串
rabin2 -zz sample.exe  # 更详细字符串
```

### `rasm2`

For quick assembly/disassembly:

```powershell
rasm2 -d "9090"
rasm2 -a x86 -b 64 "xor eax, eax"
```

### `radiff2`

For comparing two binaries:

```powershell
radiff2 old.exe new.exe
radiff2 -C old.exe new.exe
```

### `rahash2`

For calculating hashes:

```powershell
rahash2 -a md5 sample.exe
rahash2 -a sha256 sample.exe
```

### `rax2`

For base and encoding conversion:

```powershell
rax2 0x401000
rax2 4198400
rax2 -s hello
```

## Recommended analysis order

For an unknown sample, proceed in this order:

1. Use `rabin2 -I` to inspect format, architecture, and entry point.
2. Use `rabin2 -z` to inspect strings.
3. Use `rabin2 -i` to inspect imported functions — **MUST + Evidence (hard gate; see workflow 1)**.
4. If interactive analysis is needed, enter `r2` only after recording the Evidence from step 3.
5. Run `aaa`, then `afl` / `iz` / `pdf`.
6. Locate key functions using string references, import calls, and entry-point flow.

This order reduces noise and quickly establishes direction. Step 3 is a hard gate before deeper analysis, not an optional optimization.

## Windows considerations

- Quote paths containing spaces correctly in commands.
- If the current terminal cannot find `r2` after a `PATH` update, try a new terminal.
- Some samples require administrator access to read, but do not elevate by default unless explicitly required by the user.
- Confirm the user's intent before dynamically debugging suspicious samples to avoid unintended actions.

## Output style

When asked to analyze a file rather than merely provide commands:

- Summarize reconnaissance results first.
- List key evidence: strings, imports, functions, and addresses.
- Provide recommended next steps or continue deeper analysis.

Do not merely list commands without explaining their purpose.

## Example requests

### Example 1: Analyze an exe

User: "Check what this exe does using radare2."

Approach:

1. Start with `rabin2 -I/-z/-i`.
2. Decide whether to enter `r2`.
3. Use `aaa`, `afl`, and `pdf` to investigate the entry point and key string references.

### Example 2: Find where a string is referenced

User: "Which function triggers this error string?"

Approach:

1. Use `iz~关键字` to find the string address.
2. Use `axt <addr>` to find references.
3. Jump to a reference with `s <addr>`, then use `pdf`.

### Example 3: Change a jump

User: "Change this jne to je."

Approach:

1. Confirm the target address.
2. Explicitly state that write mode is needed.
3. Use `wa je <target>` or `wx` directly.
4. Disassemble again after the change to verify it.

## Practices to avoid

- Do not treat `radare2` as if `aaa` were its only command.
- Do not open a user's file in write mode without explaining the risks.
- Do not draw conclusions before basic reconnaissance.
- **Do not skip import-table inspection** (`rabin2 -i` / recon imports): do not proceed without recording Evidence; do not substitute other steps when the user requests a repeat import-table check.
- **Do not persist with static analysis after IAT repair fails**: record `E-iat-repair-fail` and move to dynamic analysis; do not use only ImportREC for 64-bit samples.
- Do not route web JS reverse engineering to this skill; it belongs to `reverse-engineering`.

## References

- Command cheatsheet: `references/cheatsheet.md`
- Standard reconnaissance script: `scripts/recon.ps1`

## radare2-skills ecosystem

The radare2-skills project (radareorg/radare2-skills) provides additional ecosystem tools and workflows:

- **r2xsql**: SQL queries for binary imports / strings / functions.
- **r2mcp / r2http**: MCP tools and a stateful HTTP command channel.
- **radius2**: symbolic execution and symbolic dynamic analysis.
- **r2pm**: plugin management and extensions.
- **decompiler plugins**: the radare2 plugin mechanism.

**Usage policy**:
- Prefer routing to this skill (radare2/SKILL.md) when the user mentions `r2xsql`, `r2mcp`, `r2http`, `radius2`, `r2pm`, `rabin2`, `rasm2`, `radiff2`, `rahash2`, or `rax2`.
- These tools are ecosystem accelerators and **cannot bypass** the authorization gate, `tool-index` verification, Evidence recording, or write-mode confirmation.
- Provide minimal reproducible command examples:
  - `r2xsql -s <file> -q "SELECT ..."`
  - `curl.exe -sS --data-binary 'aaa' http://127.0.0.1:9393/cmd`
  - `radius2 -p <binary> ...`
  - `r2pm -ci <plugin>`

This skill retains its hard gates and evidence chain; no authorization or Evidence step may be skipped.

---

## Routing context

**Upstream entry**: `skills/SKILL.md` (controller), `routing.md`
**Upstream alternative**: `ida-reverse/` (escalate to IDA for decompilation/pseudocode)
**Downstream routes**:
- Dynamic analysis → `reverse-engineering/tools-dynamic.md` (Frida/GDB)
- Deep decompilation → `ida-reverse/`
- Cross-references for interesting strings found by PAT → `ida-reverse/` (IDA has stronger xref capabilities)

**Related peer module**: `ida-reverse/` (complementary: fast r2 reconnaissance, deeper IDA decompilation)

## On-Demand Bootstrap

This skill's entry script uses the unified bootstrap system. When radare2 is missing, it automatically attempts installation instead of failing immediately.

### Automation capabilities

| Tool | Automatic installation | Method | Notes |
|------|-----------|---------|------|
| r2 | ✓ | GitHub Release ZIP (w64) | Automatically downloads and extracts to `%USERPROFILE%\Tools\radare2\` |
| rabin2 | ✓ | Same as above (included in the radare2 distribution) | — |
| rasm2 | ✓ | Same as above | — |
| radiff2 | ✓ | Same as above | — |
| rahash2 | ✓ | Same as above | — |
| rax2 | ✓ | Same as above | — |

### Bootstrap trigger

- `scripts/recon.ps1`: automatically invokes `bootstrap-reverse.ps1` when `rabin2` or `r2` is missing.

### Bootstrap failure

If automatic installation fails (for example, network failure or GitHub API rate limiting), the script reports a clear error and a manual installation link.

Manual installation: Download `radare2-*-w64.zip` from https://github.com/radareorg/radare2/releases, extract it to `%USERPROFILE%\Tools\radare2\`, and ensure the `bin\` directory is on PATH.

## Completion self-check (MUST pass before claiming completion)

- [ ] Did I execute every workflow step rather than merely read it?
- [ ] Was import-table inspection performed and recorded as Evidence (E-imports / E-triage-imports or .NET equivalent)? Is E-exports included for DLL/SYS?
- [ ] Was IAT repair failure recorded as E-iat-repair-fail and followed by dynamic analysis? Did a repeat request return to the same step?
- [ ] Did I use actual tool paths from `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and record the Checklist items required by RULES?
