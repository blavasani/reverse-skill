---
name: macos-reverse
description: Use for authorized macOS and Mach-O reverse engineering including codesign, Objective-C/Swift recovery, endpoint security surfaces, and Apple platform malware analysis.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# macOS / Mach-O Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md`
2. `NOW`: Confirm the target is a macOS/Mach-O/App bundle (iOS IPA → `mobile-reverse/`)
3. `NEXT`: Read tool-index; use jtool2/lldb and related tools
4. `ACT`: Signature and loading information → static → dynamic (lldb/Frida)

## Applicable scenarios

- Mach-O executables / dylibs / frameworks
- .app bundles, LaunchAgents/Daemons
- Objective-C / Swift symbols and runtime
- Notarization/signing, Hardened Runtime, and TCC-related behavior analysis
- Static/dynamic analysis of macOS malware (in conjunction with malware-analysis)

## Workflow

### 1. Bundle and signing

```bash
file target
codesign -dv --verbose=4 target
spctl -a -vv target 2>&1
otool -L target
```

### 2. Static analysis

```text
□ class-dump / swift-demangle / Hopper / Ghidra / IDA
□ 字符串与 XPC 服务名、TCC 敏感 API
□ LC_LOAD_dylib 依赖与 rpath
```

### 3. Dynamic analysis

```text
□ lldb / Frida
□ fs_usage / log stream 观察
□ 网络：联合 protocol-reverse 或代理
```

## Toolchain

| Tool | Purpose |
|------|------|
| otool / nm / codesign | system-provided |
| Hopper / Ghidra / IDA | decompilation |
| class-dump / dsdump | Objective-C |
| Frida / lldb | dynamic |
| jtool2 | Mach-O |

## References

- `references/macho-triage.md`
- `../mobile-reverse/` (iOS) `../ghidra-reverse/` `../malware-analysis/`

## Routing context

**Upstream**: MASTER R31  
**Downstream**: iOS → mobile-reverse; generic sample → malware-analysis

## Completion self-check

- [ ] Did you record signing/Hardened Runtime status?
- [ ] Do you have address-level/symbol-level conclusions?
- [ ] Checklist?
