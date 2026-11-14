# Kernel Driver Reverse Engineering Reference

> Covers Windows/Linux kernel driver reverse engineering, rootkit analysis, and C/C++ binary pattern recognition.

---

## Windows Driver Reverse Engineering

### Driver Types

| Type | Characteristics | Analysis Focus |
|------|------|---------|
| WDM (Windows Driver Model) | Legacy driver, manually manages IRPs | DriverEntry → device creation → Dispatch routines |
| KMDF (Kernel Mode Driver Framework) | Modern, event-driven framework | EvtDriverDeviceAdd → Queue → I/O callbacks |
| WDF (Windows Driver Foundation) | Collective term for KMDF + UMDF | Inspect WdfDriverCreate calls |
| Minifilter | File system filter driver | FltRegisterFilter → Pre/Post callbacks |

### WDM Driver Analysis Workflow

```text
1. 找 DriverEntry（入口点）
   - IDA 自动识别，或搜索 IoCreateDevice / IoCreateSymbolicLink

2. 找设备名和符号链接
   - IoCreateDevice → DeviceName（如 \Device\MyDriver）
   - IoCreateSymbolicLink → SymLink（如 \DosDevices\MyDriver）

3. 找 Dispatch 例程
   - DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DispatchIoctl
   - 这是用户态通过 DeviceIoControl 调用的入口

4. 分析 IOCTL 处理
   - switch(IoControlCode) 分发不同功能
   - IOCTL 编码：CTL_CODE(DeviceType, Function, Method, Access)
   - Method: METHOD_BUFFERED / METHOD_IN_DIRECT / METHOD_OUT_DIRECT / METHOD_NEITHER

5. 找漏洞
   - 用户可控缓冲区未验证长度 → 溢出
   - METHOD_NEITHER 直接使用用户指针 → 任意读写
   - 未检查 IOCTL 权限 → 非特权用户可调用
```

### IOCTL Encoding Analysis

```python
# 解析 IOCTL code
def decode_ioctl(code):
    device_type = (code >> 16) & 0xFFFF
    access = (code >> 14) & 0x3
    function = (code >> 2) & 0xFFF
    method = code & 0x3
    
    methods = {0: "BUFFERED", 1: "IN_DIRECT", 2: "OUT_DIRECT", 3: "NEITHER"}
    access_types = {0: "ANY", 1: "READ", 2: "WRITE", 3: "READ|WRITE"}
    
    return f"DevType=0x{device_type:X} Func=0x{function:X} Method={methods[method]} Access={access_types[access]}"

# 示例
decode_ioctl(0x80002034)
# DevType=0x8000 Func=0x80D Method=BUFFERED Access=ANY
```

### IDA Plugins

| Plugin | Purpose | Link |
|------|------|------|
| **Driver Buddy Reloaded** | Automatically identifies IOCTLs, Dispatch routines, and device names | https://github.com/VoidSec/DriverBuddyReloaded |
| **WinDbg + IDA** | Combines kernel debugging and static analysis | Built-in |
| **FLIRT/Lumina** | Identifies WDK library functions | Built into IDA |

### Reference Articles

- [Windows Drivers RE Methodology (VoidSec)](https://voidsec.com/windows-drivers-reverse-engineering-methodology/) — Comprehensive WDM driver reverse-engineering methodology
- [Driver Reversing 101](https://eversinc33.com/posts/driver-reversing.html) — WDM vs KMDF comparison
- [Methodology of Reversing Vulnerable Killer Drivers](https://whiteknightlabs.com/2025/10/28/methodology-of-reversing-vulnerable-killer-drivers/) — Vulnerable-driver analysis

---

## Linux Kernel Module Reverse Engineering

### LKM (Loadable Kernel Module) Structure

```text
关键函数：
- init_module / module_init → 模块加载时执行
- cleanup_module / module_exit → 模块卸载时执行

关键结构：
- struct file_operations → 字符设备的 open/read/write/ioctl
- struct net_device_ops → 网络设备操作
- struct block_device_operations → 块设备操作
```

### Analysis Workflow

```text
1. 确认是内核模块
   file module.ko → "ELF 64-bit ... relocatable"（注意是 relocatable 不是 executable）

2. 找 init/exit 函数
   readelf -s module.ko | grep -E "init_module|cleanup_module"
   或在 .modinfo section 找模块信息

3. 找 file_operations 结构
   搜索 register_chrdev / cdev_add / misc_register
   → 找到 fops 结构体 → 定位 ioctl/read/write 处理函数

4. 分析 ioctl 处理
   unlocked_ioctl / compat_ioctl 函数
   → switch(cmd) 分发

5. 找 Rootkit 行为
   - 修改 sys_call_table → syscall hook
   - 修改 /proc 文件系统 → 隐藏进程/文件
   - 注册 netfilter hook → 隐藏网络连接
   - 修改 VFS 层 → 隐藏文件
```

### Common Rootkit Techniques

| Technique | Characteristics | Detection Method |
|------|------|---------|
| syscall table hook | Modify `sys_call_table` entries | Compare the in-memory table with vmlinux on disk |
| VFS hook | Modify `file_operations` function pointers | Check whether fops pointers point outside the kernel code section |
| Netfilter hook | `nf_register_net_hook` | Traverse the netfilter hook list |
| kprobe/ftrace hook | Register a kprobe or ftrace callback | Check the ftrace registration list |
| eBPF rootkit | Load a malicious BPF program | `bpftool prog list` |
| DKOM | Directly modify kernel objects (the process list) | Traverse the task_struct list and compare it with /proc |

### Tools

| Tool | Purpose |
|------|------|
| `crash` | Kernel dump analysis |
| `volatility3` | Memory forensics (Linux profile) |
| `dmesg` / `journalctl` | Kernel logs |
| `lsmod` / `/proc/modules` | Loaded-module list |
| `modinfo` | Module metadata |
| `strace` | System-call tracing (user-mode perspective) |

---

## C/C++ Reverse-Engineering Pattern Recognition

### Common C Patterns

| Source Pattern | Disassembly Characteristics |
|---------|-----------|
| `if-else` | `cmp` + `jcc` (conditional jump) |
| `switch-case` | Jump table (`jmp [rax*8 + table]`) or consecutive `cmp` instructions |
| `for` loop | `cmp` + `jl/jle` + loop body + `inc/add` + `jmp` back |
| `while` loop | Condition checked at the top of the loop |
| `do-while` | Condition checked at the bottom of the loop |
| Function-pointer call | `call rax` or `call [reg+offset]` |
| `struct` access | `[reg+固定偏移]` (for example, `[rdi+0x10]`) |
| `malloc` + use | `call malloc` → store the return value in a register → access later using that register + offset |
| String comparison | `call strcmp` or `repe cmpsb` |

### C++-Specific Patterns

| Source Pattern | Disassembly Characteristics |
|---------|-----------|
| **Virtual-function call** | `mov rax, [rcx]` (load vtable) → `call [rax+offset]` (call virtual function) |
| **Constructor** | Allocate memory → write vtable pointer → initialize members |
| **Destructor** | Clean up members → possibly call `operator delete` |
| **this pointer** | The first argument (rcx/rdi) is the object pointer |
| **Inheritance** | The vtable contains parent virtual functions and child overrides |
| **Multiple inheritance** | The object contains multiple vtable pointers at different offsets |
| **RTTI** | A `type_info` pointer precedes the vtable |
| **Exception handling** | `__cxa_throw` / `_CxxThrowException` |
| **STL containers** | `std::vector`: `{begin, end, capacity}` three-pointer structure |
| **std::string** | Small-string optimization (SSO): short strings inline, long strings heap-allocated |

### vtable Reverse-Engineering Method

```text
1. 找 vtable
   - 搜索连续的函数指针数组（在 .rodata 或 .rdata 段）
   - 构造函数中 `mov [rcx], offset vtable` 写入 vtable 指针

2. 确定类层次
   - vtable 前 -8 偏移处通常是 RTTI 指针（如果未 strip）
   - 多个 vtable 共享前几个条目 → 继承关系

3. 标注虚函数
   - vtable[0] 通常是析构函数（或 deleting destructor）
   - 后续按偏移标注：vtable[1] = func1, vtable[2] = func2...

4. IDA 中操作
   - 在 vtable 地址创建 struct（每个字段是函数指针）
   - 对 `call [rax+offset]` 添加注释标明调用的虚函数
```

### Structure Recovery

```text
方法 1：从访问模式推断
  mov eax, [rdi+0x00]  → field_0: int/ptr (4/8 bytes)
  mov ecx, [rdi+0x08]  → field_8: int/ptr
  movss xmm0, [rdi+0x10] → field_10: float

方法 2：从 sizeof 推断
  call malloc(0x30) → 结构体大小 0x30 (48 bytes)
  
方法 3：从构造函数推断
  构造函数会初始化所有字段 → 字段类型和偏移一目了然

方法 4：用 IDA 的 "Create struct" 功能
  选中访问模式 → Edit → Struct → Create struct from selection
```

---

## Common Compiler Characteristics

| Compiler | Identification Characteristics |
|--------|---------|
| MSVC | `_security_cookie` checks, `__fastcall` calling convention, Rich Header |
| GCC | `__stack_chk_fail`, `-fstack-protector`, `.note.GNU-stack` |
| Clang/LLVM | Similar to GCC but with different optimization patterns, `__asan_*` (if sanitizer is enabled) |
| MinGW | GCC characteristics + Windows API calls |
| AOSP Clang | Android-specific `__android_log_print`, PGO markers |

### Optimization-Level Identification

| Optimization Level | Characteristics |
|---------|------|
| -O0 | Many redundant mov instructions, every variable on the stack, no function inlining |
| -O1 | Basic optimization, some variables in registers |
| -O2 | Loop unrolling, function inlining, tail-call optimization |
| -O3 / -Os | Aggressive inlining, vectorization (SIMD), difficult-to-read code |
| PGO | Hot-path optimization, cold code separated into `.text.cold` |
| LTO | Cross-module inlining, global dead-code elimination |

---

## Kernel Debugging Environment

### Windows

```text
调试器：WinDbg Preview
连接方式：网络调试（推荐）或串口

被调试机设置：
bcdedit /debug on
bcdedit /dbgsettings net hostip:192.168.x.x port:50000

调试机连接：
WinDbg → File → Attach to Kernel → Net → Port:50000 Key:xxx

常用命令：
!analyze -v          # 自动分析崩溃
lm                   # 列出已加载模块
!drvobj \Driver\xxx  # 查看驱动对象
dt nt!_DRIVER_OBJECT # 显示结构体
bp module!function   # 下断点
```

### Linux

```text
调试器：GDB + QEMU 或 kgdb

QEMU 内核调试：
qemu-system-x86_64 -kernel bzImage -s -S ...
gdb vmlinux -ex "target remote :1234"

常用命令：
info threads         # 内核线程
lx-symbols           # 加载内核符号（需要 scripts/gdb/）
p init_task          # 查看 init 进程
lx-dmesg             # 内核日志
```

---

## Agent Action Anchors (Issue #65 U–AV)

Aligned with `references/nonpe-format-cookbook.md` §5 (a short table, not a replacement for the workflow above):

| ID | Action | Evidence |
|----|------|----------|
| AG | Short `DriverEntry` → scan nonempty `MajorFunction` slots, prioritizing DEVICE_CONTROL/CREATE | `E-driver-irp-handlers` |
| AH | Build an IOCTL control-code → handler table and record METHOD_* | `E-driver-ioctl` |
| AI | Suspected BYOVD: compare public vulnerable-driver lists; record name/hash/signature and call intent; **do not write exploit steps** | `E-driver-byovd` |

## Reference Resources

| Resource | Description | Link |
|------|------|------|
| VoidSec Driver Reverse Engineering Methodology | Complete Windows WDM driver analysis workflow | https://voidsec.com/windows-drivers-reverse-engineering-methodology/ |
| Elastic Rootkit Series | Linux rootkit classification and detection | https://security-labs.elastic.co/security-labs/linux-rootkits-1-hooked-on-linux |
| Driver Buddy Reloaded | IDA driver-analysis plugin | https://github.com/VoidSec/DriverBuddyReloaded |
| LOLDrivers | Known vulnerable-driver list | https://www.loldrivers.io/ |
| Windows Driver Samples | Official Microsoft driver samples | https://github.com/microsoft/Windows-driver-samples |
| Linux Kernel Module Programming | Kernel-module development tutorial | https://sysprog21.github.io/lkmpg/ |
| Trail of Bits - Devirtualizing C++ | vtable reverse-engineering method | https://blog.trailofbits.com/2017/02/13/devirtualizing-c-with-binary-ninja/ |
