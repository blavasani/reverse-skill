---
name: ida-reverse
description: "Use the ida-reverse specialist workflow after routing and authorization. Follow its ACTION REQUIRED steps and preserve evidence provenance."
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# IDA Pro Reverse Engineering Skill

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` and confirm that this skill's operations are authorized routine operations
2. `NOW`: Confirm whether the current task falls within this skill's scope
3. `NEXT`: Read `../tool-index.md` and check tool availability and actual paths
4. `NEXT`: Invoke bootstrap if tools are missing; do not guess paths
5. `ACT`: Execute the first workflow step; do not stop at confirmation

## Known Issues and Lessons (Required Reading)

### Pitfalls

1. **Do not call `idb_open` (formerly `idalib_open`) directly through the MCP client in some AI clients**
   - Some coding AI clients' MCP clients have a bug validating the output schema for open tools.
   - Error: `Structured content does not match the tool's output schema`
   - **Solution**: Use `scripts/open.ps1` to call the HTTP API directly, bypassing MCP validation.
   - In ida-pro-mcp 2.x, the tools are named `idb_open` / `idb_list` / `idb_save` (no longer `idalib_*`).
   - Opening a file returns a `session_id` (database); subsequent tool calls must include that session.

2. **Permission denied when opening files in `C:\Windows\System32\`**
   - idalib cannot directly read files in System32.
   - **Solution**: `open.ps1` detects these paths and copies the file to a temporary directory before opening it.

3. **Starting the server blocks the conversation**
   - Once started, `idalib-mcp` continuously writes INFO logs to the console.
   - **Solution**: Use `scripts/start.ps1` (`-WindowStyle Hidden` starts it silently in the background).
   - The script waits for service readiness and exits without blocking the conversation.

4. **MCP server names cannot contain hyphens**
   - The previous server name `ida-pro-mcp` could cause tool registration problems.
   - **Current configuration**: server name `idapro`, tool prefix `idapro_*`.

5. **Remote HTTP vs Local Stdio**
   - In `type:"local"` (stdio) mode, `idalib_open` also has schema validation issues.
   - In `type:"remote"` (HTTP) mode, first open the file directly with the script, then use the MCP tools.
   - **Current approach**: Remote HTTP mode.

6. **PR #389 fixed some schema issues**
   - After issue #388, author mrexodia merged a fix through PR #389.
   - It fixed the structuredContent schema in HTTP mode, but validation still fails in some coding AI clients.
   - The latest `main` branch version is installed.

7. **An idalib timeout leaves orphaned worker processes locking files**
   - After the first `open.ps1` timeout, idalib's Python worker subprocess can become orphaned and keep `.id0`/`.id1`/`.nam` locked.
   - Subsequent tools or manually dragging the file into the IDA GUI report permission errors.
   - **Do not** use `taskkill /F /T` on the process tree: `/T` also kills GUI `ida.exe` child processes.
   - **Solution**: `start.ps1` replaces the managed supervisor only when nothing listens on the port or when `tools/list` returns quickly without `py_eval` (an old supervisor). An RPC timeout while 13337 still listens means busy; do not kill it. While opening a database, `open.ps1` writes `opening.lock`; the watchdog must not use `-Force`.
   - **Deadlock exception**: Only if `tools/list` **fails continuously for more than 3 minutes** (measured from the last-healthy timestamp, not process creation), there is no in-flight `opening.lock`, and the GUI does not own the port, use `-Force` to replace the supervisor; still do not kill `ida.exe`.
   - **Fallback**: When `open.ps1` detects a locked old database, it copies the file to Temp with a GUID prefix.

8. **Opening with automatic analysis can appear to hang**
   - `idalib_open(run_auto_analysis=true)` may not return for a long time while the backend continues opening and analyzing the file.
   - Previously, users saw no PowerShell output and could mistake this for a stuck script.
   - **Current solution**: `open.ps1` adds `-TimeoutSeconds`, runs the request in the background, polls in the foreground, and periodically reports progress.
   - When polling finds the session ready, it returns `OK:filename:session_id` early; on timeout it returns `ERR:open_timeout_xxs`.

9. **HTTP MCP can silently exit after login**
   - Cursor/Claude's `type: http` does not start the process; the old scheduled task ran only once at login.
   - `pythonw` has no console, and the Application log is empty on a crash.
   - **Solution**: `start.ps1` reuses a healthy instance by default; `watchdog.ps1` checks every minute; logs are in `%LOCALAPPDATA%\reverse-skill\ida-mcp\`.
   - Install with `scripts/install-autostart.ps1`. If the port is not ready when the HTTP client starts, manually refresh the MCP panel once.

10. **Streamable HTTP GET `/mcp` can block the single-threaded supervisor**
   - Some HTTP MCP clients send a long-lived GET (SSE) to `/mcp`. The stock `idalib_supervisor` uses `HTTPServer` with `background=False` and handles one request at a time.
   - Result: `tools/list` times out and the client marks `idapro` as error.
   - **Solution**: `run-supervisor.py` replaces HTTP with `ThreadingHTTPServer` and accepts GET `/mcp`; if the patch fails, it is skipped and the supervisor still starts. When stuck, use `scripts/recover.ps1` (immediate `-Force`).

### Workflow Principles

| Step | Action | Tool |
|------|--------|------|
| 1 | Ensure the HTTP server is running | `scripts/start.ps1` (no arguments) |
| 2 | Open the target binary | `scripts/open.ps1 -Path "xxx.exe"` |
| 3 | Use MCP analysis tools | Call `idapro_*` / HTTP tools directly (about 65, depending on version) |
| 4 | Analysis complete | Tools are automatically available |

## Script Resources

### start.ps1 — Start the MCP HTTP Server

Path: `scripts/start.ps1`

- Automatically resolves `IDADIR` (environment variable, portable desktop path, or common installation paths).
- Prefers IDA's bundled `Python314\python.exe -m ida_pro_mcp.idalib_supervisor`.
- First probes `http://127.0.0.1:13337/mcp` by default; if healthy, prints `OK:<n>:reuse` and exits.
- If 13337 is listening but `tools/list` times out, prints `WARN:busy` / `OK:busy:reuse` and **does not kill it** (opening a database or GUI use may prevent a response).
- If `tools/list` **fails continuously for over 3 minutes** (last-healthy timestamp) and there is no `opening.lock`, treats it as a deadlock, prints `INFO:deadlock`, and replaces the supervisor with `-Force`. An in-progress `idb_open` or GUI does not take this path.
- Replaces the managed supervisor only if nothing listens on the port, `py_eval` is missing, or the deadlock condition above occurs; **never kills `ida.exe` or uses `taskkill /T`**.
- If the GUI occupies 13337, prints `WARN:gui_busy` and exits without starting another supervisor.
- On success prints `OK:<tool count>` (currently about 66); on failure prints `ERR:timeout`.
- Supervisor log: `%LOCALAPPDATA%\reverse-skill\ida-mcp\supervisor.log`
- The server runs in the background without blocking the conversation.

**Usage**:
```
powershell -File "<skill-root>\ida-reverse\scripts\start.ps1"
```

### watchdog.ps1 / recover.ps1 / install-autostart.ps1 — Keepalive

- `watchdog.ps1`: probes 13337; reuses a healthy instance (and refreshes last-healthy); reuses when the GUI, an `open.ps1` database-opening lock, or a last-healthy timestamp under 3 minutes indicates busy; runs `start.ps1 -Force` only after `tools/list` fails continuously for over 3 minutes.
- `recover.ps1`: immediately runs `start.ps1 -Force` (does not kill `ida.exe`). Use it when the HTTP client marks `idapro` as error.
- `install-autostart.ps1`: registers the `reverse-skill-ida-mcp` scheduled task (at login and every minute).
- Log: `%LOCALAPPDATA%\reverse-skill\ida-mcp\watchdog.log`

### open.ps1 — Open a Binary File

Path: `scripts/open.ps1`

- Calls `idb_open` directly through the HTTP API, bypassing MCP schema validation.
- Detects System32 paths and copies files to a temporary directory.
- Automatically cleans up old database files with the same name (`.id0`/`.id1`/`.nam`/`.til`/`.i64`).
- If an old database is locked, falls back by copying it to Temp with a GUID prefix and opening the copy without an error.
- Runs the open request in the background to avoid an unresponsive script during a long synchronous wait.
- Supports `-TimeoutSeconds`; returns `ERR:open_timeout_xxs` on timeout instead of hanging indefinitely.
- Prints `INFO:opening:elapsed/timeout seconds` every 10 seconds so progress is visible.
- On success prints `OK:filename:session_id`; adds `(temp copy)` when falling back.
- Retries through a Temp copy on failure.

**Usage**:
```
powershell -File "<skill-root>\ida-reverse\scripts\open.ps1" -Path "C:\path\to\file.exe"
```

**Optional parameters**:
```
# 指定 SessionId
powershell -File "scripts\open.ps1" -Path "file.exe" -SessionId "my_session"

# 跳过自动分析（大文件推荐）
powershell -File "scripts\open.ps1" -Path "large.exe" -NoAutoAnalysis

# 设置超时，避免带自动分析时长时间无返回
powershell -File "scripts\open.ps1" -Path "file.exe" -TimeoutSeconds 600
```

**Output conventions**:
```
# 分析进行中（每 10 秒输出一次）
INFO:opening:11/600s

# 成功打开
OK:sample.exe:abcd1234

# 成功打开，但因锁文件降级到 Temp 副本
OK:1234abcd-sample.exe:abcd1234 (temp copy)

# 达到超时上限
ERR:open_timeout_600s
```

**Empirical notes**:
- With automatic analysis, `Snipaste.exe` took about `324s` to return successfully in testing; this is “long analysis,” not a “script deadlock.”
- Therefore, for GUI programs or complex samples, explicitly set `-TimeoutSeconds 600` first.

## Core Tools

### Overview Analysis (First Step)
- `idapro_survey_binary(detail_level="minimal")` — Quick overview: function count, strings, segments, entry points, and import categories (cryptography/network/file IO)
- `idapro_list_funcs(queries)` — List functions (pagination and name filtering)
- `idapro_list_globals(queries)` — List global variables
- `idapro_entity_query(kind, filter)` — Unified query: functions/globals/imports/strings/names

### Decompilation and Disassembly
- `idapro_decompile(addr)` — Decompile to pseudocode
- `idapro_disasm(addr, max_instructions=N)` — Disassemble
- `idapro_analyze_function(addr, include_asm=false)` — Comprehensive analysis (pseudocode, strings, constants, callers, callees, and blocks)
- `idapro_func_profile(queries)` — Function summary metrics

### Cross-References and Data Flow
- `idapro_xrefs_to(addrs)` — Find references to the target address
- `idapro_xref_query(addr, direction)` — Advanced xref query (direction/type filtering)
- `idapro_callees(addrs)` — List callees
- `idapro_callgraph(roots, max_depth)` — Call graph
- `idapro_trace_data_flow(addr, direction, max_depth)` — Trace data flow (forward/backward)

### Search
- `idapro_find_regex(pattern, limit)` — Search strings using a regular expression
- `idapro_search_text(pattern)` — Search text in the disassembly listing
- `idapro_find_bytes(patterns, limit)` — Search byte patterns (supports ?? wildcards)
- `idapro_find(type, targets)` — Advanced search (immediates/strings/references)

### Memory and Data
- `idapro_get_bytes(addrs)` — Read raw bytes
- `idapro_get_string(addrs)` — Read strings
- `idapro_get_int(queries)` — Read integer values
- `idapro_get_global_value(queries)` — Read global variable values
- `idapro_read_struct(queries)` — Read structure field values
- `idapro_search_structs(filter)` — Search structures

### Modification Operations
- `idapro_set_comments(items)` — Add comments (synchronized between disassembly and decompilation)
- `idapro_append_comments(items)` — Append comments
- `idapro_rename(batch)` — Batch rename (functions/globals/locals/stack variables)
- `idapro_patch_asm(items)` — Patch assembly instructions
- `idapro_patch(patches)` — Patch bytes
- `idapro_define_func(items)` — Define functions
- `idapro_undefine(items)` — Undefine items
- `idapro_define_code(items)` — Convert bytes to code

### Type System
- `idapro_declare_type(decls)` — Declare C structures/enums/unions
- `idapro_set_type(edits)` — Apply types to functions/globals/locals
- `idapro_infer_types(addrs)` — Infer types
- `idapro_type_query(queries)` — Query declared types
- `idapro_type_inspect(queries)` — Inspect type details

### Stack Frames
- `idapro_stack_frame(addrs)` — Inspect stack variables
- `idapro_declare_stack(items)` — Declare stack variables
- `idapro_delete_stack(items)` — Delete stack variables

### Signatures
- `idapro_make_signature(addrs)` — Generate a unique byte signature for an address
- `idapro_make_signature_for_function(addrs)` — Generate a function signature
- `idapro_find_xref_signatures(addrs)` — Generate signatures for code referencing an address

### Debugger (requires ?ext=dbg)
- `idapro_open_file(file_path)` — Open a file in a GUI IDA instance
- Debugger tools are hidden by default; enable them with the URL parameter `?ext=dbg`.

### Session Management (ida-pro-mcp 2.x)
- `idapro_idb_open` / HTTP `idb_open` — ⚠️ Using `open.ps1` to open is recommended
- `idapro_idb_list` / HTTP `idb_list` — List all sessions
- `idapro_idb_save` / HTTP `idb_save` — Save the database
- Most analysis tools require the `database=<session_id>` parameter (the session output by open.ps1).

### Other
- `idapro_int_convert(inputs)` — Convert number bases (**use this; do not calculate conversions yourself!**)
- `idapro_export_funcs(addrs, format)` — Export functions (json/c_header/prototypes)
- `idapro_py_eval(code)` — Execute Python in the IDA context
- `idapro_server_health()` — Check server health
- `idapro_server_warmup()` — Warm up subsystems (string cache, Hex-Rays, etc.)

## Complete Reverse Engineering Workflow

### Step 1: Start the Server

**Path A — Headless idalib (requires a valid license)**
```
powershell -File "scripts/start.ps1"
```
The output `OK:<tool count>` indicates readiness (currently about 65 tools).

**Path B — GUI + plugin (if the idalib license fails or interactive analysis is needed)**
```
powershell -File "scripts/start-gui.ps1" -Path "C:\目标.exe"
```
Alternatively, double-click the portable `Launch-IDA-Pro.cmd` and open the sample in IDA.

Once `[MCP] ... port=13337` appears in the Output window, MCP tools are available.

See `LOCAL-SETUP.md` for general integration steps.

### Step 2: Open the File

Headless:
```
powershell -File "scripts/open.ps1" -Path "C:\目标.exe" -TimeoutSeconds 600
```
Output `OK:filename:session_id` indicates success; a trailing `(temp copy)` indicates automatic fallback to a temporary copy.

If `ERR:idalib_license:...` occurs, switch to path B (GUI mode); do not repeatedly retry open.ps1.

In GUI mode, open the sample directly in IDA; open.ps1 is not needed.

### Step 3: Global Overview (Including the Import Table Hard Gate)
```
idapro_survey_binary(detail_level="minimal")
```
Focus on:
- Architecture (x86/x64/ARM)
- Entry points (main/WinMain/DllMain)
- Interesting strings (URLs, paths, and error messages)
- **Import categories (MUST)**: cryptographic functions / network APIs / file operations / process injection / registry — these must be recorded as Evidence (suggested ID: `E-imports`); use `idapro_entity_query(kind="imports")` or the imports section of the survey output.
- **DLL/SYS**: Record export and import tables together (Evidence `E-exports`).
- **.NET**: When there is no traditional IAT, record module/metadata/managed-reference summaries in the E-imports semantic slot as an equivalent anchor.
- **Clean import table**: Note suspected dynamic loading and drive dynamic API breakpoint validation.
- Hot functions (those with high xref counts are usually key logic).

**Hard gate**: Until the imports view/category summary (or a legitimate equivalent anchor) is recorded in Evidence, MUST NOT enter Step 4 deep-dive conclusions or claim that the survey is complete. If the import table is empty or the query fails, MUST still record the failure. If packed-binary IAT repair fails, MUST record `E-iat-repair-fail` and switch to dynamic debugging to capture APIs; do not persist with static analysis. If the user requests that the import table/IAT check be repeated, MUST repeat the named step (when blocked, use a feasibility gate: explain and confirm; if forced, mark `quality=unreadable`); do not substitute an unrelated step.

### Step 4: Investigate Key Functions
```
idapro_analyze_function(addr="关键函数名")
```
Or:
```
idapro_decompile(addr="函数名")
idapro_disasm(addr="函数名", max_instructions=50)
```

### Step 5: Data Flow and Cross-References
```
idapro_xrefs_to(addrs="关键地址/字符串")
idapro_callgraph(roots=["关键函数"], max_depth=3)
idapro_trace_data_flow(addr="关键地址", direction="backward", max_depth=5)
```

### Step 6: Document and Refine
```
idapro_set_comments(items=[{"addr": "0x140001000", "comment": "你的理解"}])
idapro_rename(batch={"func": [{"addr": "函数地址", "name": "有意义的名字"}]})
```

### Step 7: Produce a Report
After analysis is complete, generate `report.md` to record findings and steps.

## Prompt Engineering Guidelines

1. **Do not manually convert number bases** — Use `idapro_int_convert` whenever a number needs conversion.
2. **Survey before diving deeper** — Review the overview before targeted analysis.
3. **Keep adding comments and renaming** — Update function and variable names throughout analysis to improve subsequent accuracy.
4. **Follow cross-references** — When interesting data or strings are found, use `xrefs_to` to see what references them.
5. **For obfuscated code** — First preprocess it with string decryption, import-hash removal, control-flow unflattening, and similar techniques.
6. **C++ STL code** — Identify library functions with FLIRT/Lumina before analyzing application logic.
7. **Do not brute-force** — Derive the solution from disassembly, using simple Python to assist calculations.
8. **For "No database bound"** — No binary is open yet; run `open.ps1` first.
9. **For "Failed to open database"** — An old database may be locked; `open.ps1` automatically falls back to a Temp copy (output includes `(temp copy)`).
10. **When opening GUI or complex samples with automatic analysis** — Add `-TimeoutSeconds 600` by default; do not mistake prolonged `INFO:opening:...` output for a stuck script.

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master control), `routing.md`
**Upstream alternative**: `radare2/` (use r2 for quick reconnaissance first if you do not want to open IDA)
**Downstream routes**:
- Frida dynamic validation → `reverse-engineering/tools-dynamic.md`
- Symbolic execution/angr → `reverse-engineering/tools-dynamic.md`
- General reverse engineering methodology → `reverse-engineering/SKILL.md`

**Related peer module**: `radare2/` (alternative when IDA is unavailable)

---

## On-Demand Bootstrap

This skill's entry scripts are integrated with the unified bootstrap system.

### Automation Capabilities and Limits

| Tool | Automatic installation | Installation method | Notes |
|------|------------------------|---------------------|-------|
| idalib-mcp | ✓ | pip install (from GitHub) | `start.ps1` installs it automatically if missing |
| IDA Pro itself | ✗ | Commercial software; manual installation required | Set the `IDADIR` environment variable to the installation directory |

### Installation Steps (Verified)

```cmd
# 1. 设置 IDA 路径（替换为你的实际 IDA 安装目录）
setx IDADIR "<你的IDA安装目录>"

# 2. 从 GitHub 安装 ida-pro-mcp（PyPI 上的 ida-mcp 是另一个项目，不要装错！）
pip install git+https://github.com/mrexodia/ida-pro-mcp.git

# 3. 安装 IDA 插件（选择 Streamable HTTP + Global + 全选客户端）
ida-pro-mcp --install

# 4. 重启 IDA Pro，打开目标文件
# 插件自动监听 127.0.0.1:13337

# 5. 验证
ida-pro-mcp --config
```

> ⚠️ **Note**: The `ida-mcp` package on PyPI (author jtsylve) is a different project and is not the one needed here.
> Install `mrexodia/ida-pro-mcp` from GitHub.

### Bootstrap Triggers

- `scripts/start.ps1`: automatically invokes `bootstrap-reverse.ps1` when `idalib-mcp` is missing.
- MCP registration: bootstrap automatically writes `idapro` to the Claude MCP configuration.

### Prerequisites

- IDA Pro is installed and the `IDADIR` environment variable is set (or the script's default path is correct).
- Using `ida-pro-mcp` in IDA's bundled Python314 is recommended (it is included in the portable version).
- Common local configuration:
  - User env `IDADIR` → IDA installation directory (containing `ida.exe`)
  - Optional `~\Tools\bin\idalib-mcp.cmd` / `ida-pro-mcp.cmd` wrappers
  - Keep only the client MCP server name `idapro` → `http://127.0.0.1:13337/mcp`

## Completion Checklist (MUST Pass Before Claiming Completion)

- [ ] Did I execute every workflow step rather than only read it?
- [ ] Was survey/imports written to Evidence (`E-imports` or an equivalent)? Does DLL/SYS include `E-exports`? If IAT failed, was `E-iat-repair-fail` recorded?
- [ ] If the user requested repeating the import table/IAT check, did I repeat the same step?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the Checklist items required by RULES?
