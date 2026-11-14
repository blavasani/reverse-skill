# Obfuscator-LLVM Deobfuscation

> OLLVM deobfuscation workflow for APK .so files, ELF binaries, and control-flow flattening.
> Tool and variant information is based on research into active community projects in 2026, not training memory.
> Applies to Android NDK hardening, CTF reverse engineering, packed .so analysis, and commercial obfuscators.

---

## 0. Quick Decision: Which Tool Should I Use?

Choose directly based on your environment and assessment of the target's obfuscation type:

| Your situation | First choice | Alternative | Notes |
|---------|---------|------|------|
| IDA Pro 7.5-7.7 + Hex-Rays available; one-click deflattening wanted | **obpo-plugin** | d810-ng | obpo uses microcode + data flow + hybrid execution and has the strongest results, but it is a cloud plugin (network required, core closed-source) |
| IDA Pro (any recent version) available; local all-in-one deobfuscation wanted | **d810-ng** | Original D-810 | Local, open source, integrates Z3, and supports OLLVM/Tigress/Hodur/Approov variants |
| Binary Ninja available | **ollvm-breaker** | — | Designed for practical Android .so analysis (hardened samples such as libvdog) |
| No IDA/BN; scripting only; x86/x64 target | **ollvm-unflattener** (Miasm) | angr deflat | Miasm-based symbolic execution with multi-layer BFS processing |
| No IDA/BN; scripting only; x86/x64 target | **ollvm-unflattener** (Miasm) | angr deflat | Miasm-based symbolic execution with multi-layer BFS processing |
| Pure Python symbolic execution in a CTF scenario | **angr** Deobfuscator | Triton | Scriptable without a GUI dependency |
| ARM64 .so target without IDA | **deollvm** (Unicorn) | angr | Unicorn-based ARM64 deflat |
| BR obfuscation (indirect branches) encountered | **DeObfBR** | Make the data segment read-only | Goron/Arkari-style BR obfuscation can be countered simply by making the data segment read-only |
| Tigress obfuscation encountered | d810-ng `UnflattenerSwitchCase`/`UnflattenerTigressIndirect` | — | d810-ng includes a dedicated Tigress unflattener |

> **Key recommendation:** Prefer **d810-ng** (local, actively maintained, broad variant coverage). When cloud services are available, **obpo-plugin** has the strongest results. If both fail, use **angr/Miasm** symbolic execution for customized processing.

---

## 1. Modern OLLVM Variant Ecosystem (2026 Community Survey)

OLLVM is no longer just the original 2017 repository. The following obfuscator branches are currently active; **identify the target variant before deobfuscation**, because countermeasures differ substantially between variants:

### 1.1 Obfuscator Branch Lineage

| Variant | Base LLVM | Features added over original OLLVM | Countermeasure notes |
|------|----------|----------------------|---------|
| **Obfuscator** (original) | 3.3~4.0 | sub + bcf + fla (three core passes) | Standard tools can handle it |
| **Hikari** | 6~8 | Anti Class Dump, Function Call Obfuscate, Function Wrapper, Indirect Branching, Split BB, String Encryption | Decrypt strings and repair indirect jumps first |
| **Hikari-LLVM15** | 15~19 | + Anti Debugging, Anti Hook, Constant Encryption | Closed-source; Constant Encryption makes static analysis harder |
| **goron** | 7~10 | Indirect Branch/Call/GlobalVariable | ⚠️ Goron-style indirect obfuscation can be countered simply by making the data segment read-only |
| **Arkari** (komimoe/Hikari) | 14~latest | Based on goron, continuously maintained | As with goron, a read-only data segment provides a partial countermeasure |
| **Pluto** | 14 | MBA Obfuscation, Random CF, Split BB, **Trap Angr** (designed specifically to thwart angr) | ⚠️ The Trap Angr pass breaks angr symbolic execution; switch tools or bypass the trap |
| **Polaris** (formerly Pluto) | 16 | Alias Access, Indirect Branch/Call, String Encryption, Merge Function, Linear MBA, Dirty Bytes Insertion, Function Splitting, Junk Insertion | Combines Hikari+Pluto; the most difficult and requires layered processing |
| **O-MVLL** | open-obfuscator | Python-driven pass manager; Anti Hooking, Arithmetic(MBA), BB Duplicate, CF Breaking, Function Outline, Indirect Branch/Call, Opaque Constants | Common in modern Android hardening; Python configuration is easy to customize |
| **amice** (Rust) | Rust implementation | Full set + VM Flatten, Instruction Virtualization, Delayed Offset Loading, Parameter Aggregation | Includes virtualization; restore VM handlers rather than only deflattening |
| **VMP family** (SmallVmp/VMPilot/xVMP/VMPacker) | — | Instruction virtualization | **Outside OLLVM scope**; requires VM reverse engineering and dedicated VM tools |

### 1.2 Key Identification Clues

- **Trap Angr** (Pluto/Polaris): If angr crashes during execution or suffers path explosion, suspect the target uses the Trap Angr pass → switch to d810-ng or a dynamic Unicorn method
- **Goron/Arkari indirect jumps**: If the dispatcher uses indirect jumps (BR x8 instead of switch), first try making the relevant data segment read-only; indirect jump targets often become statically solvable
- **Constant Encryption** (Hikari-LLVM15/Polaris/O-MVLL): Constants are decrypted at runtime and their real values are invisible to purely static analysis → dynamically execute the decryption stub with Unicorn
- **VM Flatten** (amice): Control flow becomes a VM dispatch loop; **do not treat it as ordinary fla**; identify the VM handler table first

---

## 2. Detecting OLLVM Obfuscation Types

Signatures of OLLVM's three core passes:

### 2.1 Control Flow Flattening (`fla`)

**IDA view characteristics:**
- The function entry first jumps to a unique dispatcher block
- The main logic is split into multiple basic blocks, each of which jumps back to the dispatcher at the end
- The dispatcher uses a **state variable** to choose the next block to execute
- A large `switch` structure whose cases have no logical relationship

```
Original:             OLLVM flattened:
  block_A               entry -> dispatcher
  block_B                 ↓
  block_C              state_machine:
                         switch(state):
                           0 → block_A
                           1 → block_B
                           2 → block_C
```

**Variant forms (multiple dispatchers recognized by d810-ng):**
- O-LLVM: switch / if-chain + state variable
- Tigress: `m_jtbl` (switch-case) or `m_ijmp` (indirect jump, requiring `goto_table_info` configuration)
- Hodur (PlugX): nested `while(1)` state machine, `jnz state, #CONST`, with **no switch dispatcher**
- Approov: `while(v8 != C)`, with state constants concentrated in `0xF6000–0xF6FFF`

### 2.2 Bogus Control Flow (`bcf`)

- **Unreachable fake branches** are inserted between real branches
- Fake branches are protected by **opaque predicates** (conditions that are always true or false but cannot be directly proven by static analysis)
- Large amounts of dead code inflate the function body

```c
// 经典不透明谓词：x(x+1) 必为偶数，编译器证明不了
if (x * (x + 1) % 2 == 0) {
    // 真实逻辑
} else {
    // 不可达垃圾代码
}
```

### 2.3 Instruction Substitution (`sub`) → MBA

- Simple arithmetic and bitwise operations are replaced with equivalent complex expressions (MBA, Mixed Boolean-Arithmetic)

```
a + b  →  (a ^ b) + 2*(a & b)
a ^ b  →  (a | b) - (a & b)
a - b  →  a + (~b) + 1
```

### 2.4 Quick Classification Table

| Obfuscation type | IDA characteristics | Primary countermeasures |
|---------|---------|------------|
| fla (flattening) | Large switch + dispatcher | obpo / d810-ng / deflat |
| bcf (bogus control flow) | Unreachable branches + dead code | d810-ng opaque predicate removal / symbolic execution |
| sub/MBA | Complex arithmetic expressions | d810-ng MBA simplifier / SiMBA (Z3) |
| fla + bcf + sub | All techniques combined, with extreme expansion | **Layered deobfuscation (bcf, then fla, then sub)** |

---

## 3. Detailed Guide to Major Tools (Active Community Projects)

### 3.1 obpo-plugin — Strongest Results, Cloud Plugin

> [obpo-project/obpo-plugin](https://github.com/obpo-project/obpo-plugin) · 629⭐ · Active in 2026-06

A Hex-Rays **microcode** pseudocode optimizer that reconstructs flattened control flow using **data-flow tracing + program slicing + hybrid execution (concolic)**. Its results are recognized by the community as among the strongest.

**Key features:**
- Operates at the microcode layer and directly optimizes decompiler output (does not modify ASM)
- Supports IDA 7.5.0 / 7.6.0 / 7.7.0 + Hex-Rays
- Architectures: ARM, ARM64, x86, x86_64, PowerPC, PowerPC64, MIPS (7.6/7.5)
- **Cloud plugin**: The target function binary is uploaded to obpo-server for processing (core closed-source, plugin freely open-source)
- Self-funded server maintenance, 600s timeout, **multithreading/malicious calls prohibited**

**Installation and usage:**
```text
1. 下载 obpo_plugin.py 和 obpoplugin 目录
2. 复制到 IDA plugins 路径
3. 重启 IDA，打开目标二进制
4. 在 CFG 中定位分发块（dispatcher），通常长这样：
   [截图参考仓库 assets/dispatchblock.png]
5. 右键 → OBPO → Mark and process function
6. 处理完成后刷新反编译器
7. 可根据反编译变化继续标记新的分发块（迭代处理嵌套 fla）
```

**Use cases and limitations:**
- ✅ Standard and nested fla, with good results
- ⚠️ Network required; use caution with sensitive samples (undisclosed internal vulnerabilities, trade secrets) because the binary is uploaded
- ⚠️ The server may be unavailable; depends on the author's maintenance
- ❌ Does not solve every obfuscation (as explicitly stated by the author)

### 3.2 d810-ng — Local All-in-One First Choice

> [w00tzenheimer/d810-ng](https://github.com/w00tzenheimer/d810-ng) · 223⭐ · Updated 2026-06-26

A modern maintained/refactored version of D-810 (Next Generation). Runs locally, is open source, integrates the **Z3 SMT** solver, and has the broadest variant coverage.

**Core capabilities (organized from the d810-ng README):**

*Instruction-level optimization:*
| Category | Description |
|------|------|
| MBA simplification | `(a+b)-2*(a&b) => a^b`, Z3-verified DSL rules |
| Hacker's Delight | Bitwise equivalences (from the book Hacker's Delight) |
| O-LLVM patterns | Obfuscator-LLVM-specific MBA patterns |
| Constant folding | 22 constant simplification rules |
| Predicate simplification | Opaque predicate removal (setz/setnz/lnot/smod) |
| Z3 rules | SMT solving when template matching fails |
| Hodur-specific | MBA patterns from PlugX (Hodur) malware |

*Control-flow unflatteners (classified by target obfuscation):*
| Unflattener | Target | Description |
|------------|------|------|
| `Unflattener` | O-LLVM | Standard switch/if-chain + state variable |
| `UnflattenerSwitchCase` | Tigress | Tigress switch-case dispatcher (`m_jtbl`) |
| `UnflattenerTigressIndirect` | Tigress | Tigress indirect jump (`m_ijmp`), requiring `goto_table_info` configuration |
| `HodurUnflattener` | Hodur (PlugX) | Nested `while(1)` + `jnz state, #CONST`, no switch |
| `BadWhileLoop` | Approov | `while(v8 != C)`, state constants at 0xF6000–0xF6FFF |
| `UnflattenerFakeJump` | Generic | Removes always-true/always-false conditional jumps |
| `SingleIterationLoopUnflattener` | Residual | Cleans single-iteration loops where `INIT == CHECK` and `UPDATE != CHECK` |
| `UnflattenControlFlowRule` (experimental) | Generic | CFG unflattener based on path emulation |

**Installation and usage:**
```text
1. clone d810-ng
2. 安装依赖（含 Z3）
3. 复制到 IDA plugins 目录
4. IDA 中按 Ctrl-Shift-D 加载插件
5. 在 GUI 中勾选要应用的规则集
6. 对目标函数应用
```

**Why choose d810-ng over original D-810:**
- Original D-810 is less actively maintained
- d810-ng has CI tests, refactored code, and dedicated Tigress/Hodur/Approov unflatteners
- Integrates Z3 and falls back to SMT solving when template matching fails, improving the success rate

### 3.3 ollvm-unflattener — Miasm Symbolic Execution, Script Only

> [cdong1012/ollvm-unflattener](https://github.com/cdong1012/ollvm-unflattener) · 265⭐ · Active in 2026-06

Based on the **Miasm** symbolic execution engine, with no IDA/BN dependency and a pure Python command-line interface.

**Features:**
- Uses Miasm symbolic execution to recover original control flow (unlike MODeflattener's purely static method)
- **Multi-layer BFS processing**: Automatically follows calls from the target function and recursively deobfuscates
- Supports Windows/Linux x86/x64
- Outputs a new deobfuscated binary

**Installation and usage:**
```bash
git clone https://github.com/cdong1012/ollvm-unflattener.git
cd ollvm-unflattener
pip install -r requirements.txt   # miasm, graphviz, keystone-engine

# 基本用法
python unflattener -i <input.bin> -o <output.bin> -t <function_addr> -a
# -a: 自动跟随调用做多层处理
```

**Use case:** No IDA, x86/x64 target, and a need for batch scripting.

### 3.4 ollvm-breaker — Binary Ninja in Practice

> [amimo/ollvm-breaker](https://github.com/amimo/ollvm-breaker) · 441⭐

Uses **Binary Ninja** to deflatten binaries. The repository includes the hardened Android sample `libvdog.so` as a test case and has fixed functions including JNI_OnLoad, crazy::GetPackageName, and prevent_attach_one.

**Use case:** Binary Ninja users and practical Android .so analysis.

### 3.5 deollvm — ARM64 Unicorn

> [GeT1t/deollvm](https://github.com/GeT1t/deollvm) · 34⭐ · 2026-04

ARM64 OLLVM deflat based on **Unicorn**. An alternative for processing ARM64 .so files without IDA.

### 3.6 DeObfBR — BR Obfuscation Specialist

> [Mrack/DeObfBR](https://github.com/Mrack/DeObfBR) · 96⭐ · 2026-06-25

Dedicated to removing **BR obfuscation** (indirect-branch obfuscation in the Goron/Arkari style).

**⚠️ Simple countermeasure (from awesome-ollvm):** Goron/Arkari-style indirect obfuscation can be countered simply by **making the data segment read-only**—indirect jump targets often depend on a writable data segment at runtime, and making it read-only makes them statically solvable.

### 3.7 angr — General Symbolic Execution Framework

```python
import angr

proj = angr.Project("target.so", auto_load_libs=False)
cfg = proj.analyses.CFGFast()
func = proj.kb.functions[0x12345]

# 内置 Deobfuscator
deob = proj.analyses.Deobfuscator(func=func)
deob.normalize()
```

**⚠️ Pluto/Polaris Trap Angr pass:** These two variants specifically add traps to thwart angr symbolic execution. If angr suffers path explosion or errors, suspect Trap Angr → switch to d810-ng or a dynamic Unicorn method.

---

## 4. Complete Deobfuscation Workflow (By Scenario)

### 4.1 General Decision Tree

```
目标二进制
  ↓
1. 识别 OLLVM 变种（看 1.2 节线索）
  ├── 原始 OLLVM / Hikari / O-MVLL  → 标准 fla/bcf/sub
  ├── Pluto / Polaris                → 注意 Trap Angr，避开 angr
  ├── Goron / Arkari                 → 先试数据段只读，再处理 BR
  ├── Tigress                        → d810-ng Tigress unflattener
  ├── Hodur (PlugX)                  → d810-ng HodurUnflattener
  └── amice (含 VM)                  → 不是单纯 fla，需 VM handler 还原
  ↓
2. 选择工具（看第 0 节决策表）
  ├── 有 IDA + 可联网 + 非敏感样本 → obpo-plugin
  ├── 有 IDA + 本地              → d810-ng
  ├── 有 Binary Ninja            → ollvm-breaker
  ├── 无 GUI + x86/x64           → ollvm-unflattener (Miasm)
  ├── 无 GUI + ARM64             → deollvm (Unicorn) / angr
  └── 纯符号执行 / CTF           → angr
  ↓
3. 分层去混淆（顺序很重要）
  a) 先去除不透明谓词 (bcf)   → d810-ng opaque predicate removal
  b) 再去除控制流平坦化 (fla) → unflattener
  c) 最后简化 MBA (sub)       → d810-ng MBA simplifier / SiMBA
  ↓
4. 验证
  ├── 函数体积显著减小？
  ├── CFG 从星形/放射状变为链形/树形？
  └── Frida hook 关键函数验证逻辑正确？
```

### 4.2 Android NDK .so Deobfuscation

OLLVM-hardened .so files compiled with Android NDK are the most common APK reverse-engineering scenario.

**Step 1 — Extract the .so:**
```bash
adb pull /data/app/~~/lib/arm64/libnative.so
# 或从 APK 直接解压：unzip target.apk -d out/ ; find out -name "*.so"
```

**Step 2 — Identify OLLVM and its variant:**
```bash
readelf -a libnative.so | grep -E "Size|text"   # .text 异常大但函数少 → 大概率 OLLVM
# IDA 打开看函数特征：
#   巨大 switch → fla
#   不可达分支 → bcf
#   复杂算术 → sub/MBA
#   间接跳转 BR x8 → Goron/Arkari，试数据段只读
#   while(1) + jnz state → Hodur，用 d810-ng HodurUnflattener
```

**Step 3 — Deobfuscate in layers:**
```
a) bcf: d810-ng opaque predicate removal  (或 obpo 自动处理)
b) fla: d810-ng Unflattener / obpo-plugin / deollvm(ARM64)
c) sub: d810-ng MBA simplifier
```

**Step 4 — Dynamic validation with Frida:**
```javascript
// Trace OLLVM 状态变量，辅助 deflat 确定状态变量地址
const target = Module.findBaseAddress("libnative.so");
console.log("[+] libnative.so @", target);

// 在分发器入口下 hook，观察 state 变化序列
Interceptor.attach(target.add(0x1234), {  // dispatcher offset
    onEnter(args) {
        // 读取状态变量（需根据反编译确定寄存器/栈位置）
        console.log("[state]", this.context.x8);  // 假设 state 在 x8
    }
});
```

### 4.3 Quick Deobfuscation for CTF Scenarios

CTFs are usually time-constrained, so use the fastest path first:

```python
#!/usr/bin/env python3
"""CTF OLLVM quick deflat with angr"""
import angr

proj = angr.Project("challenge", auto_load_libs=False)
cfg = proj.analyses.CFGFast()

# 找最大的几个函数（最可能是被混淆的）
funcs = sorted(cfg.functions.values(), key=lambda f: f.size, reverse=True)[:5]
for func in funcs:
    print(f"[*] {func.name} @ {hex(func.addr)} size={hex(func.size)}")
    try:
        deob = proj.analyses.Deobfuscator(func=func)
        deob.normalize()
        print(f"    [+] deobfuscated")
    except Exception as e:
        print(f"    [-] failed: {e}")
        # angr 失败 → 怀疑 Trap Angr → 换 d810-ng / Unicorn
```

---

## 5. MBA Expression Simplification

### 5.1 Common OLLVM MBA Patterns

```python
# 这些等式是 OLLVM sub pass 生成表达式的化简目标
"(a | b) + (a & b)"        # → a + b
"(a | b) - (a & b)"        # → a ^ b
"(a ^ b) + 2*(a & b)"      # → a + b
"(a | b) & ~(a & b)"       # → a ^ b
"~(~a & ~b)"               # → a | b (De Morgan)
```

### 5.2 Tool Selection

| Tool | Method | Use case |
|------|------|------|
| **d810-ng MBA simplifier** | Batch processing inside IDA, Z3-verified | First choice, integrated into the decompilation workflow |
| **SiMBA** (`pip install simba-simplifier`) | Command line/library | Pure expression simplification and batch processing |
| **Arybo** | Symbolic bit vectors | Large numbers of MBA expressions |
| **Direct Z3 solving** | SMT | Most general; use when template matching fails completely |

```python
# SiMBA 示例
from simba import simplify_mba
exprs = ["(a | b) + (a & b)", "(a ^ b) + 2*(a & b)"]
for e in exprs:
    print(f"{e}  →  {simplify_mba(e)}")
```

---

## 6. Complete Deobfuscation Case Script

```bash
#!/bin/bash
# OLLVM deobfuscation pipeline (2026 community tools)
# 适用标准 OLLVM / Hikari / O-MVLL 加固的 ELF/.so

BINARY=$1

echo "[*] Stage 0: 基本分析与变种识别"
file $BINARY
readelf -h $BINARY 2>/dev/null | head -5
echo "    → 在 IDA 中确认变种（参考第 1 节）"

echo "[*] Stage 1: d810-ng 本地反混淆（首选）"
echo "    IDA → Ctrl-Shift-D 加载 d810-ng"
echo "    勾选: MBA + Opaque predicate + Unflattener"
echo "    Apply to target functions"
echo "    保存 IDB"

echo "[*] Stage 2: obpo-plugin（如 d810-ng 效果不足且可联网）"
echo "    IDA → 右键 dispatcher → OBPO → Mark and process"
echo "    ⚠️ 敏感样本勿用（二进制上传云服务）"

echo "[*] Stage 3: 无 IDA 备选（x86/x64）"
echo "    python unflattener -i $BINARY -o deobf.bin -t <func_addr> -a"

echo "[*] Stage 4: ARM64 .so 无 IDA 备选"
echo "    deollvm (Unicorn) 或 angr Deobfuscator"

echo "[+] Done. 在 IDA 中重新分析验证。"
```

---

## 7. Common Pitfalls (Community Practice Summary)

| Problem | Cause | Solution |
|------|------|---------|
| angr path explosion/abnormal exit | Pluto/Polaris **Trap Angr** pass | Switch to d810-ng or a dynamic Unicorn method |
| obpo-plugin cannot connect | Self-funded server maintenance; may be unavailable | Switch to local d810-ng; file an issue in the obpo repository |
| Goron/Arkari indirect-jump deflat fails | Dispatcher uses BR x8 rather than switch | Make the data segment read-only first, then use DeObfBR |
| Function remains chaotic after d810-ng processing | OLLVM customized pass parameters/seed | Use symbolic execution to remove opaque predicates, then unflatten |
| Nested fla (multi-layer flattening) not fully cleared in one pass | obpo/d810-ng clears only one layer per run | **Iterate**: mark each newly appearing dispatcher |
| ARM64 .so deflat reports an error | Old deflat scripts support x86 only | Use d810-ng / obpo (ARM64 supported) / deollvm |
| Hikari strings are not visible | String Encryption pass | Emulate the decryption stub with Unicorn and dump decrypted strings |
| amice target deflat is completely ineffective | Contains VM Flatten / Instruction Virtualization | **Not OLLVM fla**; restore VM handlers (see VM reverse engineering) |
| Hodur(PugX) sample has no switch dispatcher | Nested while(1) + jnz state | Use d810-ng **HodurUnflattener**, not the ordinary Unflattener |
| Approov sample state constants show no pattern | Constants concentrated at 0xF6000–0xF6FFF | Use d810-ng **BadWhileLoop** unflattener |
| obpo used accidentally on a sensitive sample | Binary uploaded to cloud service | For confidential/undisclosed-vulnerability samples, **use local tools only** (d810-ng/angr) |
| Frida hook of an OLLVM function hangs | State variable modification causes an infinite loop | Add a conditional breakpoint at the dispatcher entry to limit execution count |

---

## 8. Tool Quick Reference (2026 Community Activity)

| Tool | Platform | Method | Stars/Price | Last update | Open source | Notes |
|------|------|------|---------|---------|------|------|
| **obpo-plugin** | IDA | microcode+concolic (cloud) | 629 | 2026-06 | Plugin open-source/core closed-source | Strongest results, network required |
| **ollvm-breaker** | Binary Ninja | BN API | 441 | 2026-06 | ✅ | Practical Android .so analysis |
| **ollvm-unflattener** | CLI | Miasm symbolic execution | 265 | 2026-06 | ✅ | x86/x64, multi-layer BFS |
| **d810-ng** | IDA | microcode+Z3 | 223 | 2026-06 | ✅ | **Local first choice**, broad variant coverage |
| **DeObfBR** | — | BR obfuscation specialist | 96 | 2026-06 | ✅ | Goron/Arkari indirect branches |
| **IDA_Ollvm-unflattener** | IDA | Miasm plugin version | 90 | 2026-04 | ✅ | IDA plugin wrapper for ollvm-unflattener |
| **deollvm** | CLI | Unicorn | 34 | 2026-04 | ✅ | ARM64 specialist |
| **angr** | CLI | Symbolic execution | — | Active | ✅ | General-purpose, constrained by Trap Angr |
| **SiMBA** | CLI/library | MBA simplification | — | — | ✅ | Expression simplification |
| **Triton** | CLI | Symbolic execution+taint | — | Active | ✅ | Dynamic symbolic execution |

---

## 9. Reference Links

**Obfuscators (for understanding the targets of countermeasures):**
- [obfuscator-llvm/obfuscator](https://github.com/obfuscator-llvm/obfuscator) — Original OLLVM
- [HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari) — Hikari
- [komimoe/Hikari](https://github.com/komimoe/Hikari) — Arkari (based on goron, LLVM 14+)
- [amimo/goron](https://github.com/amimo/goron) — goron
- [bluesadi/Pluto](https://github.com/bluesadi/Pluto) — Pluto
- [za233/Polaris-Obfuscator](https://github.com/za233/Polaris-Obfuscator) — Polaris (formerly Pluto)
- [open-obfuscator/o-mvll](https://github.com/open-obfuscator/o-mvll) — O-MVLL
- [fuqiuluo/amice](https://github.com/fuqiuluo/amice) — Rust implementation of OLLVM passes
- [lich4/awesome-ollvm](https://github.com/lich4/awesome-ollvm) — **Variant ecosystem overview (strongly recommended first reading)**

**Deobfuscation tools:**
- [obpo-project/obpo-plugin](https://github.com/obpo-project/obpo-plugin) — Strongest cloud plugin
- [w00tzenheimer/d810-ng](https://github.com/w00tzenheimer/d810-ng) — Local first choice
- [cdong1012/ollvm-unflattener](https://github.com/cdong1012/ollvm-unflattener) — Miasm script-only tool
- [amimo/ollvm-breaker](https://github.com/amimo/ollvm-breaker) — Binary Ninja
- [GeT1t/deollvm](https://github.com/GeT1t/deollvm) — ARM64 Unicorn
- [Mrack/DeObfBR](https://github.com/Mrack/DeObfBR) — BR obfuscation specialist
- [maskelihileci/IDA_Ollvm-unflattener](https://github.com/maskelihileci/IDA_Ollvm-unflattener) — IDA plugin version
- [angr](https://angr.io/) — Symbolic execution framework
- [SiMBA](https://github.com/tech-srl/simba) — MBA simplification

**Academic/blog:**
- [Quarkslab: Deobfuscation: Recovering an OLLVM-protected program](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html) — Classic deflat principles
- [MODeflattener](https://github.com/mrT4ntr4/MODeflattener) — Static deflat (comparison for ollvm-unflattener)

> Related documents: [[anti-analysis.md]] (anti-debugging/anti-analysis overview), [[tools-advanced.md]] (advanced toolset), [[elf-analysis.md]] (ELF file analysis), [[ai-assisted-re.md]] (AI-assisted reverse engineering)
