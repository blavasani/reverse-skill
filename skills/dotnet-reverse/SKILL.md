---
name: dotnet-reverse
description: "Use the dotnet-reverse specialist workflow after routing and authorization. Follow its ACTION REQUIRED steps and preserve evidence provenance."
license: MIT
compatibility: "Requires a filesystem-based code agent or CLI with shell access. Windows is preferred for dnSpyEx; Linux and macOS can use ILSpy/de4dot CLI with mono or the dotnet runtime."
allowed-tools: Bash Read Write Edit Glob Grep Task WebFetch WebSearch
metadata:
  user-invocable: "false"
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# .NET / C# Reverse-Engineering Practice

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Use DIE/`file`/the CLR header to confirm the target is managed .NET (otherwise SWITCH to `ida-reverse/` / `reverse-engineering/`)
2. `NOW`: If obfuscation is suspected → run `de4dot` first, produce `*-clean.exe`, and preserve the original sample
3. `NEXT`: Use dnSpyEx (or dnSpy MCP / `ilspycmd`) statically: browse C# and inspect key decisions in the **IL view**
4. `ACT`: Dynamically debug when plaintext/C2 is needed; prefer an **IL patch** over C# recompilation when changing logic
5. At the end of the phase, give the user a menu of 3–6 next steps (including report export)

## Scope

Prefer this skill when the task involves:

- Identifying and reverse-engineering compiled .NET / C# artifacts (managed PE / .exe / .dll)
- Analyzing red-team Sharp* toolchains (Rubeus, SharpHound, SharpShell, and others)
- Deobfuscating ConfuserEx / SmartAssembly / Babel / Eazfuscator / .NET Reactor and similar protectors
- Reverse-engineering decryption and C2 logic in .NET loaders / info-stealers / RATs
- Patching C# programs (changing decisions, constants, or keygen)
- Analyzing the Mono/Unity managed layer before IL2CPP (note: after IL2CPP compilation it is native; use `reverse-engineering/` + seed-014)

If the target is a purely native binary (compiled C/C++/Go/Rust, with no CLR), use `reverse-engineering/`, `ida-reverse/`, or `radare2` instead.

## Core Principles

- **Identify before acting**: first confirm that it is a managed .NET program (PE CLR header + `#~` / `#Strings` streams + mscoree `_CorExeMain`), then choose dnSpy rather than IDA
- **IL before C#**: dnSpyEx's C# decompiler can lose/distort information (compiler-generated state machines, async/await, yield); key decisions and patches must use the **IL editor**, while the C# view is for quick browsing
- **de4dot first**: when an obfuscator is encountered, run one `de4dot` pass before static analysis; otherwise strings/control flow remain scrambled
- **MCP integration**: when dnSpy MCP (`dnspy_*` tools) is registered, prefer the MCP surface for decompilation / IL inspection to avoid switching GUIs
- **Evidence-based output**: save deobfuscated artifacts, extracted configuration/C2/key material, and patch diffs to disk

## Toolchain Mapping

| Capability | Preferred | Notes |
|------|------|------|
| Decompilation + debugging + patching | **dnSpyEx** | Best-in-class, the only GUI with an IL editor; legacy dnSpy is discontinued, use the Ex fork |
| Lightweight CLI / headless decompilation | **ILSpy** (`ilspycmd`) | Suitable for batch/scripted use and Linux/macOS |
| Deobfuscation | **de4dot** | Default solution for the ConfuserEx family, SmartAssembly, and other mainstream protectors |
| Obfuscator identification | **Detect It Easy (DIE)** / **file** | Determine the protector type before choosing de4dot parameters |
| Programmatic IL manipulation | **dnlib** | Write C# scripts to batch-edit metadata / string decryptors |
| Direct AI operation | **dnSpy MCP** | Tool surfaces such as `dnspy_decompile` / `dnspy_inspect_il` |

> Prerequisite: install dnSpyEx + de4dot on Windows (choco or release); use `ilspycmd` + `dotnet runtime` on Linux/macOS. See the installation matrix in `references/sharp-tools.md`.

## Six-Phase Workflow

### 1. Identify (.NET)

Confirm that the target is a managed program; do not analyze a native PE as .NET:

```powershell
# Windows
file target.exe                       # "PE32 executable ... for MS Windows" is insufficient
# Key check: determine whether CLR is present
powershell -c "[System.Reflection.AssemblyName]::GetAssemblyName('target.exe')"
# Or
dnSpyEx: drag it directly into dnSpyEx — if it opens, it is managed

# Generic
strings target.exe | grep -iE "mscoree|_CorExeMain|mscorlib|System\\."
```

**.NET identification markers:**
- PE header `Data Directory[14]` (CLR Runtime Header) is nonzero
- `mscoree.dll` import / `_CorExeMain` entry point
- `#~`, `#Strings`, `#US`, `#GUID`, `#Blob` metadata streams
- `mscorlib` / `System.Private.CoreLib` strings

**NativeAOT exception:** compiled as native, with no CLR header but with `System.Private.CoreLib` strings and reconstructed type metadata — use `reverse-engineering/` (IDA/r2) for these; this skill only provides identification guidance.

### 2. Detect (Obfuscator Detection)

```powershell
# Quick DIE identification
diec target.exe                        # Detect It Easy CLI
# Or drag it into dnSpyEx to check for many garbled class names / control-flow transformations
```

Common obfuscators → deobfuscation strategies (see `references/obfuscators.md`):

| Obfuscator | Characteristics | de4dot Handling |
|--------|------------|------------|
| ConfuserEx (1.0.0 / 2.x) | `<module>` anti-tamper, control-flow transformation, string encryption | `de4dot target.exe` usually detects it automatically |
| SmartAssembly | `circular`/`string encoding`, resource compression | `de4dot target.exe` |
| Babel.NET | Method-body encryption, control flow | `de4dot target.exe` |
| Eazfuscator.NET | String/resource encryption | `de4dot`, some versions require manual work |
| .NET Reactor | anti-tamper + necrobit | `de4dot`, newer versions may fail and require manual work |

### 3. Deobfuscate

```powershell
# de4dot automatically detects most protectors by default
de4dot target.exe -o target-clean.exe

# Specify the type (when automatic detection fails)
de4dot --type cfze target.exe          # ConfuserEx
de4dot --type sa target.exe            # SmartAssembly

# Multi-layer obfuscation / de4dot reports unknown
de4dot --detect target.exe             # See what it identifies
# You may need to patch anti-tamper before de4dot (see references/obfuscators.md)
```

Output: `target-clean.exe`; use it for subsequent analysis. **Preserve the original sample** for comparison.

### 4. Static Analysis

dnSpyEx loads the deobfuscated sample:

- **C# view**: quickly browse class structure, method signatures, and strings (for locating targets)
- **IL view**: inspect key decisions, encryption logic, and state machines in IL (right-click → Edit IL or IL view)
- Find entry points: `Main` / `Startup` / module initializer (`Module .cctor`)
- Find key logic: search for `flag`, `password`, `verify`, `check`, `encrypt`, `http`, `Config`

```text
Locate string → find cross-references → find the method using it → inspect decision logic in IL view
```

### 5. Dynamic Analysis

dnSpyEx debugger: attach to a process / start debugging, set breakpoints in key methods, and observe runtime behavior:
- Decrypted plaintext strings (many obfuscators decrypt strings only at runtime)
- C2 addresses and decrypted configuration results
- Exception-driven control flow (anti-debug commonly uses `try/catch` to hide the real path)

> .NET dynamic debugging is much friendlier than native debugging — object values and string contents are directly visible. Prefer dynamic analysis over struggling with static analysis.

### 6. Patch (as needed)

```text
dnSpyEx → right-click method → Edit Method (C#) or Edit IL
  - Change decision: ldc.i4.0 → ldc.i4.1 (false→true)
  - Change constant: edit strings/numbers directly
  - Remove validation: nop the entire block
File → Save Module → replace the original file
```

**IL patch reliability > C# patch**: C# recompilation may fail (missing references, invalid syntax), while IL editing rarely distorts behavior. See `references/common-workflow.md`.

## Trigger Routing

Enter this skill when the user says:
- ".NET / C# binary reverse engineering" / "decompile a C# program"
- "dnSpy analysis" / "dnSpyEx patch"
- "ConfuserEx / SmartAssembly / Babel deobfuscation / unpacking"
- "analyze Sharp* tools" (Rubeus / SharpHound / SharpShell)
- "reverse-engineer .NET malware / loader / info-stealer"
- "patch a C# program / keygen / change a decision"

## When to Switch Out

- Unity game compiled with IL2CPP → `reverse-engineering/` + `seed-014_unity-il2cpp-reverse.md` (IL2CPP is native; do not use dnSpy)
- NativeAOT artifact → `reverse-engineering/` (same, native)
- Pure native PE (no CLR) → `reverse-engineering/` / `ida-reverse/`
- Batch-migrate symbols/functions to another version → `binary-diff/`
- Draw an attack path / call-chain diagram → `diagram-generator/`

## Routing Context

**Upstream entry**: `skills/SKILL.md` (controller), `routing.md`
**Downstream exits**:
- IL2CPP / NativeAOT (native) → `reverse-engineering/`
- Deep native .so/.dll segment analysis → `ida-reverse/` / `radare2/`
- Direct AI operation of dnSpy → register and integrate dnSpy MCP (see `references/sharp-tools.md`)

**Related peer modules**:
- `reverse-engineering/languages-compiled.md` (.NET overview points to this module)
- `apk-reverse/` (Xamarin/MAUI Android reverse engineering can return here for the C# layer)

## Reference Documents

- [references/obfuscators.md](references/obfuscators.md) — detailed ConfuserEx / SmartAssembly / Babel / Eazfuscator / .NET Reactor deobfuscation + anti-tamper bypass
- [references/common-workflow.md](references/common-workflow.md) — complete workflow, IL patch reliability, string-decryptor extraction, and state-machine identification
- [references/sharp-tools.md](references/sharp-tools.md) — red-team Sharp* tool analysis, tool-installation matrix, dnSpy MCP integration, and community resource index

## Task Completion Checklist

- [ ] Did I confirm CLR / managed identity (or SWITCH out of this skill)?
- [ ] Was the obfuscated sample processed with de4dot / equivalent unpacking before deep analysis?
- [ ] Was key logic verified in the IL view rather than only in C# pseudocode?
- [ ] Were artifacts (clean sample / configuration / patch diff) saved and made reproducible?
- [ ] Did I provide a next-step menu or report exit?
