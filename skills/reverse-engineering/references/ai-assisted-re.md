# AI-Assisted Reverse Engineering

> LLM-driven decompilation / multi-agent validation / neural semantic recovery
> The biggest paradigm shift of 2025-2026

## Core Tools and Models

### LLM4Decompile
- First open-source framework to use LLMs for binary-to-source decompilation
- Supports x86/ARM/MIPS architectures
- Input: assembly code → output: C source
- Training data: millions of source-assembly pairs

### Decaf (2026)
- **Compiler-feedback validation**: compile LLM-generated source and compare it with the original binary
- Result: decompilation rate 26% → 83.9% (ExeBench Real -O2)
- Key insight: the feedback loop is more effective than a larger model

### Constraint-Guided Multi-Agent (2026)
- Three-stage validation pipeline:
  1. Syntax correctness (parsing)
  2. Compilability (GCC)
  3. Behavioral equivalence (LLM-generated test cases)
- 84-97% reproducibility, at only $0.03-0.05 per run

### REMEND (2026)
- Specialization: extracting mathematical equations from binaries
- 89.8-92.4% accuracy (across 3 ISAs × 3 optimization levels × 2 languages)
- Speed: 0.132s/function, only 12M parameters

### Glaurung
- Open-source Ghidra alternative with a Rust core + Python bindings
- **AI-native architecture**: an LLM agent embedded in each analysis layer
- Evidence artifacts: plain/rich/JSON/JSONL output formats for LLM consumption
- Supports ELF/PE/Mach-O, x86/ARM/RISC-V, IOC detection, and entropy analysis

## Workflow: AI-Enhanced Binary Analysis

### 1. LLM-Assisted Rapid Reconnaissance

```text
□ Extract strings → LLM semantic classification (URL/key/path/protocol)
□ Analyze imports → LLM infers functionality (crypto = OpenSSL? network = libcurl?)
□ Inspect disassembly snippets → LLM identifies patterns (cryptography, anti-debugging, VM detection)
□ Review error messages → LLM infers context ("Invalid license" → authorization logic location)
```

### 2. Neural Decompilation

```bash
# LLM4Decompile
python llm4decompile.py --binary target.so --arch arm64 --output target.c

# Verify the result (recompile and compare)
gcc -O2 -o target_recompiled target.c -fPIC -shared
# → 验证输出行为等价性
```

### 3. Multi-Agent Validation

```text
Agent 1 (syntax): Check whether the generated C code parses
  ↓ failure → send the error to the LLM and retry
Agent 2 (compilation): Compile with GCC → check warnings/errors
  ↓ failure → send the compiler error to the LLM
Agent 3 (behavior): LLM generates inputs → run the original and recompiled versions → compare outputs
  ↓ mismatch → send the difference to the LLM → iterate on the fix
```

### 4. LLM-Assisted Static Analysis

```text
□ 函数重命名: 输入反编译伪代码 → LLM 建议语义化名称
□ Type recovery: analyze context → LLM infers struct/class definitions
□ Algorithm identification: assembly snippets → LLM identifies cryptographic algorithms (AES/TEA/RC4/custom)
□ Protocol reverse engineering: network packet sequences → LLM infers the protocol format
□ Comment generation: decompiled code → LLM generates clear English comments
```

### 5. macOS/iOS Private-Framework Reverse Engineering (MOTIF)

```text
Problem: macOS private frameworks lack documentation and type information
Approach: LLM analyzes usage patterns → infers method signatures and parameter types
Result: ObjC signature recovery improves from 15% to 86% (vs. static analysis)
```

## LLM Prompt Templates

### Function Semantic Analysis

```
You are a reverse engineering expert. Analyze this decompiled function:

[伪代码]

1. What does this function do? (one sentence)
2. Suggest a meaningful function name.
3. What are the input parameters and their likely types?
4. What is the return value?
5. What external APIs/functions does it depend on?
6. Any security-relevant operations (crypto, auth, network, file I/O)?
```

### Algorithm Identification

```
Analyze this assembly/disassembly for cryptographic operations:

[汇编代码]

1. Is this a known cryptographic algorithm? (AES/DES/RC4/TEA/ChaCha20/custom?)
2. Identify the key schedule and round structure.
3. What is the key size?
4. Are there any hardcoded constants that identify the algorithm?
```

### Protocol Format Inference

```
Given this network packet sequence, infer the protocol structure:

[hex dump]

1. Identify magic bytes and length fields.
2. Propose a struct definition for the packet header.
3. What field(s) appear to be checksums/CRCs?
4. Is this a known protocol or custom?
```

## Tool Selection

| Scenario | Recommended Tool | Cost |
|------|---------|------|
| Rapid decompilation | LLM4Decompile | Free (local GPU) |
| High-accuracy decompilation | Constraint-Guided Multi-Agent | ~$0.05/binary |
| Mathematical-function extraction | REMEND | Free |
| Cross-platform RE | Glaurung (Rust) | Free and open source |
| LLM interaction | Claude API / GPT-4 / DeepSeek | ~$0.01-0.10/request |

## Limitations

- **Complex control flow**: virtualization/obfuscated code remains difficult (control-flow flattening, VMProtect)
- **Indirect calls**: virtual tables and function pointers are difficult to recover
- **Inline functions**: boundaries become unclear after compiler inlining
- **Floating-point operations**: semantic recovery for vectorized instructions needs improvement
- **Context window**: large functions (>1000 lines) exceed LLM context limits

Source: Decaf (2026), REMEND (2026), Constraint-Guided Multi-Agent Decompilation (2026), LLM4Decompile, Glaurung
