---
name: ghidra-reverse
description: Use for free/open reverse engineering with Ghidra (headless or GUI), including decompile, cross-refs, and optional Ghidra MCP workflows when IDA is unavailable.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Ghidra Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md`
2. `NOW`: Confirm that **Ghidra** is required (no IDA / open-source preference / batch headless)
3. `NEXT`: Read `../tool-index.md` to find the ghidra / ghidra-mcp paths
4. `NEXT`: If tools are missing, bootstrap `ghidra-mcp` (if supported by the manifest) or install Ghidra using the manual steps
5. `ACT`: Import the sample → run automatic analysis → export decompilation for key functions

## Applicable Scenarios

- Primary reverse-engineering entry point when no IDA license is available
- Batch headless analysis / decompilation in CI
- Automation with Ghidra scripts (Java/Python Jython/PyGhidra)
- ghidriff integration with `binary-diff` / `patch-diff-exploit`

## Division of Work with IDA

| Requirement | Priority |
|------|------|
| Deep analysis with an existing IDA MCP | `ida-reverse/` |
| Open source / batch / educational use | **this skill** |
| CLI-only quick reconnaissance | `radare2/` |

## Workflow

### 1. Project and Automatic Analysis

```text
□ 新建 Project → Import 文件 → Analyze（默认分析器）
□ 记录语言/编译器识别结果与基址
□ 标记入口、导出表、字符串 xref
```

### 2. Key Functions

```text
□ 从字符串 / 导入 API 反查
□ Decompile 窗口还原算法
□ 重命名函数/变量；写 Plate comment
□ 需要动态时交接 Frida/GDB（reverse-engineering 动态章）
```

### 3. Headless (Batch)

```bash
# 示例：analyzeHeadless 路径因安装而异，MUST 从 tool-index 取
analyzeHeadless /path/to/project Proj -import sample.bin -postScript ExportDecomp.py
```

### 4. MCP (If Configured)

```text
□ 确认 ghidra MCP 端口（常见 8765，以 tool-index 为准）
□ 用 MCP 工具拉反编译 / xrefs，禁止猜端口
```

## Toolchain

| Tool | Purpose | Bootstrap |
|------|------|------|
| Ghidra | Primary decompilation tool | Manual release / package manager |
| ghidra-mcp | AI bridge | Bootstrap capability name `ghidra-mcp` |
| ghidriff | Patch diffing | See `patch-diff-exploit` |

## References

- `references/ghidra-cheatsheet.md`
- `../ida-reverse/` `../radare2/` `../binary-diff/`

## Routing Context

**Upstream**: MASTER R22
**Downstream**: Dynamic validation → Frida/GDB; exploitation → `pwn-chain`
**Peer**: `ida-reverse` (commercial deep analysis)

## Task Completion Checklist

- [ ] Is the path based on the actual Ghidra/tool-index entry?
- [ ] Are function addresses and renames documented?
- [ ] Are the steps reproducible?
- [ ] Checklist / journal?