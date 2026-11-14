# Go Binary Reverse Engineering Guide

> Go binaries present distinct challenges: static linking increases their size, they can contain tens of thousands of functions, strings use a special format, and stripped symbols are difficult to recover.
> This document covers tools, recovery techniques, and analysis workflows.

---

## Identifying Go Binaries

Quickly determine whether a binary was compiled with Go:

```bash
# 字符串特征
strings binary | grep -E "runtime\.|go\.buildid|GOROOT"

# rabin2 侦察
rabin2 -z binary | grep -i "runtime"

# 文件大小异常大（静态链接 runtime）
# 典型 Hello World: C ~20KB, Go ~2MB
```

Common indicators:
- Many functions with the `runtime.` prefix
- A `go.buildid` section
- `GOROOT` and `GOPATH` path strings
- 5000-50000+ functions, including the runtime and standard library

---

## Core Tools

### Symbol Recovery

| Tool | Purpose | Link |
|------|------|------|
| **GoReSym** | Mandiant tool for parsing Go symbol information (pclntab/moduledata) | https://github.com/mandiant/GoReSym |
| **GoResolver** | Volexity tool for automatically deobfuscating Garble binaries using CFG similarity | https://github.com/volexity/GoResolver |
| **redress** | Analyze stripped Go binaries and recover types, interfaces, and package structure | https://github.com/goretk/redress |
| **GoStringUngarbler** | Google tool for recovering Garble-obfuscated strings | https://github.com/mandiant/GoStringUngarbler |

### IDA Plugins

| Tool | Purpose | Link |
|------|------|------|
| **go_parser** | IDA plugin for parsing moduledata, pclntab, and type information | https://github.com/0xjiayu/go_parser |
| **IDAGolangHelper** | IDA script collection for parsing Go type information | https://github.com/sibears/IDAGolangHelper |
| **AlphaGolang** | SentinelLabs IDAPython script collection | https://github.com/SentineLabs/AlphaGolang |
| **IDA 9.2+ native support** | Official Hex-Rays Go decompilation improvements | https://hex-rays.com/blog/stop-guessing-and-start-going |

### Ghidra Plugins

| Tool | Purpose | Link |
|------|------|------|
| **Ghidra + GoReSym output** | Export symbols with GoReSym and import them into Ghidra | Used together |
| **golang_loader_assist** | Go loading assistance for Ghidra | Community script |

### Standalone Analysis Tools

| Tool | Purpose | Link |
|------|------|------|
| **gore** | Go reverse-engineering library underlying redress | https://github.com/goretk/gore |
| **garble** | Go obfuscation tool; understanding it helps counter it | https://github.com/burrowers/garble |

---

## Key Go Binary Structures

### pclntab (PC Line Table)

The most important structure in a Go binary contains:
- Function names and address mappings
- Source file paths
- Line numbers
- Stack frame sizes

Even when symbols are stripped, pclntab usually remains because the Go runtime depends on it.

```text
定位方法：
1. 搜索 magic bytes: 0xFFFFFFF0 (Go 1.16+) 或 0xFFFFFFFB (Go 1.18+)
2. 用 GoReSym 自动定位
3. 用 go_parser IDA 插件自动解析
```

### moduledata

Contains:
- pclntab pointer
- Type information table
- itab (interface table)
- Global variable information

### String Format

Go strings are not C-style null-terminated strings; they use a `(pointer, length)` structure:

```text
C 字符串:   "hello\0"
Go 字符串:  struct { ptr *byte; len int } → ptr 指向 "hello"（无 \0）
```

Consequently, default IDA and Ghidra string recognition misses many Go strings.

**Options**:
- Use `go_parser` to identify Go strings automatically
- Export a string list with GoReSym
- Manually locate `runtime.stringtable` or use cross-references

---

## Analysis Workflows

### Scenario 1: Unstripped Go Binary

```text
1. GoReSym -t -d -p binary > symbols.json
   → 导出所有函数名、类型、源文件路径
2. 加载到 IDA/Ghidra
3. 导入 GoReSym 的符号信息
4. 过滤掉 runtime.* 和标准库函数，聚焦用户代码
5. 从 main.main 开始分析
```

### Scenario 2: Stripped Go Binary

```text
1. GoReSym -t -d -p binary > symbols.json
   → 即使 strip 了，pclntab 通常还在
2. 如果 GoReSym 失败 → 用 redress
   redress -src binary    # 恢复源文件路径
   redress -pkg binary    # 恢复包结构
   redress -type binary   # 恢复类型信息
3. 加载到 IDA + go_parser 插件
4. 运行 go_parser 自动恢复
5. 从恢复的 main.main 开始
```

### Scenario 3: Garble-Obfuscated Go Binary

```text
Garble 会：
- 随机化函数名（main.main → main.a3f2b1c）
- 加密字符串
- 移除文件路径信息
- 混淆包名

对抗方法：
1. GoResolver（CFG 签名匹配）
   → 通过控制流图相似度恢复标准库函数名
2. GoStringUngarbler（字符串解密）
   → 自动识别 Garble 的字符串加密模式并解密
3. 动态分析（Frida/dlv）
   → Hook runtime 函数观察实际行为
4. 对比分析
   → 编译同版本 Go 的 Hello World，用 binary-diff 对比 runtime 部分
```

### Scenario 4: CGo Mixed Compilation

```text
1. 识别 CGo 边界（_cgo_* 函数）
2. Go 部分用 go_parser 恢复
3. C 部分用常规 IDA 分析
4. 关注 _cgo_topofstack、crosscall2 等桥接函数
```

---

## Common Commands

```bash
# GoReSym：导出符号
GoReSym -t -d -p binary > symbols.json
GoReSym -t -d -p binary -o ida_script.py  # 生成 IDA 脚本

# redress：分析 stripped 二进制
redress -src binary          # 源文件路径
redress -pkg binary          # 包结构
redress -type binary         # 类型信息
redress -interface binary    # 接口信息
redress -filepath binary     # 完整文件路径

# GoResolver：去混淆 Garble
GoResolver -binary binary -output resolved.json

# GoStringUngarbler：解密 Garble 字符串
GoStringUngarbler -i binary -o deobfuscated_binary

# 快速判断 Go 版本
strings binary | grep "go1\."
GoReSym -p binary | grep "Version"
```

---

## Go Analysis in IDA

```text
1. 加载二进制（选择正确的架构）
2. 等待自动分析完成
3. 运行 go_parser 插件：
   - File → Script File → go_parser.py
   - 或 Edit → Plugins → Go Parser
4. 插件会自动：
   - 解析 pclntab
   - 恢复函数名
   - 标记 Go 字符串
   - 解析类型信息
5. 过滤视图：
   - 隐藏 runtime.* 函数
   - 聚焦 main.* 和第三方包
6. 从 main.main 开始逆向
```

---

## Common Pitfalls

| Pitfall | Description | Mitigation |
|------|------|------|
| Too many functions | Go static linking produces 5000-50000 functions | Filter by package name and focus on main.* and application packages |
| Incomplete string recognition | Go strings are not null-terminated | Recover strings with go_parser or GoReSym |
| Difficult decompilation output | Go defer, goroutine, and interface constructs complicate pseudocode | Use IDA 9.2+ improvements or supplement with dynamic analysis |
| Garble obfuscation | Function names and strings are randomized | Use GoResolver + GoStringUngarbler |
| Version differences | pclntab format varies by Go version | GoReSym supports Go 1.2-1.23+ |
| CGo boundary | Go and C code are mixed | Identify _cgo_* functions as boundaries |

---

## Integration With Other Skills

| Need | Tool |
|------|--------|
| In-depth Go binary analysis in IDA | `ida-reverse/` + go_parser plugin |
| Ghidra analysis (free) | Ghidra + GoReSym symbol import |
| Quick reconnaissance | `radare2/` — inspect strings with `rabin2 -z` |
| Dynamic hooking | Frida (hook runtime functions) or dlv (native Go debugger) |
| Cross-version comparison | `binary-diff/` — migrate symbols from an older version to a newer version |
| Garble deobfuscation | GoResolver + GoStringUngarbler |
