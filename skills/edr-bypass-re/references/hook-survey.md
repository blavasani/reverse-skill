# EDR Hook Survey Quick Reference

> Authorized red-team operations, adversary simulations, and testing of owned products only; do not use against unauthorized targets.

This document summarizes monitoring points used by major EDR / AV products in user mode and kernel mode, helping red teams quickly identify what requires analysis during reconnaissance.

## 1. Major EDR Fingerprints and Hook Patterns

| Vendor / product | User-mode components | Kernel drivers | Primary monitoring surface |
|------------|-----------|---------|-----------|
| CrowdStrike Falcon | `CSFalconService.exe`, `CSAgent.sys` injected into target processes | `CSAgent.sys`, `CSBoot.sys` | Kernel-heavy callbacks + ETW-TI; fewer user-mode hooks (cloud scanning) |
| Microsoft Defender for Endpoint (MDE) | `MsMpEng.exe`, `MpClient.dll` | `WdFilter.sys`, `WdBoot.sys`, `WdNisDrv.sys` | Comprehensive AMSI + ETW-TI + ntdll inline hooks + kernel callbacks |
| SentinelOne | `SentinelAgent.exe`, `SentinelHelperService.exe` | `SentinelMonitor.sys`, `SentinelDeviceControl.sys` | Heavy ntdll user-mode hooks + kernel callbacks + proprietary ETW provider |
| Elastic Defend (formerly Endpoint Security) | `elastic-endpoint.exe` | `elastic-endpoint-driver.sys` | Primarily ETW + limited ntdll hooks, with uploads through Elastic Agent |
| ESET | `ekrn.exe`, `eamsi.dll` | `eamonm.sys`, `epfwwfp.sys` | Many user-mode hooks (NtCreateFile / NtOpenProcess, etc.) |
| Sophos Intercept X | `SophosFileScanner.exe`, `SophosNtpService.exe` | `SophosED.sys`, `hmpalert.sys` | ntdll hooks + HMPA memory protection + kernel callbacks |
| Kaspersky | `avp.exe`, `klif.sys` | `klif.sys`, `klhk.sys` | Heavy user-mode hooks + KLIF proprietary minifilter + network filter driver |
| Trend Micro Apex One | `TmListen.exe`, `TmCCSF.dll` | `tmcomm.sys`, `tmactmon.sys` | User-mode hooks + behavioral monitoring drivers |
| Carbon Black | `RepMgr.exe`, `RepWAV.exe` | `ParityDriver.sys` | Kernel-focused callbacks + ETW |

### 快速指纹脚本

```powershell
$edrSigs = @{
    'CSAgent'           = 'CrowdStrike Falcon'
    'SentinelAgent'     = 'SentinelOne'
    'elastic-endpoint'  = 'Elastic Defend'
    'ekrn'              = 'ESET'
    'MsMpEng'           = 'Microsoft Defender'
    'SophosFileScanner' = 'Sophos Intercept X'
    'avp'               = 'Kaspersky'
    'TmListen'          = 'Trend Micro Apex One'
    'cb'                = 'Carbon Black'
}

Get-Process | ForEach-Object {
    foreach ($k in $edrSigs.Keys) {
        if ($_.ProcessName -match $k) {
            "[+] $($edrSigs[$k]) detected: $($_.ProcessName) (PID $($_.Id))"
        }
    }
}

Get-ChildItem 'C:\Windows\System32\drivers\*.sys' |
    Where-Object { $_.Name -match 'CSAgent|Sentinel|elastic|eam|WdFilter|Sophos|klif|tmcomm|Parity' } |
    Select-Object Name, VersionInfo
```

## 2. Key User-Mode ntdll Hook Targets

EDRs almost always hook these `ntdll.dll` exports, grouped by ATT&CK behavior:

| Function | Monitored behavior | ATT&CK |
|----------|-------------------|--------|
| `NtCreateThreadEx` | Remote-thread injection, QueueUserAPC injection | T1055.002 / T1055.004 |
| `NtAllocateVirtualMemory` | Allocate RWX memory for shellcode | T1055 |
| `NtAllocateVirtualMemoryEx` | Cross-process memory allocation (new API on Win10+) | T1055 |
| `NtProtectVirtualMemory` | Change page permissions RW→RX | T1055 |
| `NtWriteVirtualMemory` | Write shellcode across processes | T1055.012 |
| `NtMapViewOfSection` | Section-based injection (Process Doppelganging / Ghosting) | T1055.013 |
| `NtCreateSection` | Used with MapViewOfSection | T1055.013 |
| `NtOpenProcess` | Open the target process and obtain a handle | T1057 |
| `NtQueueApcThread` / `NtQueueApcThreadEx` | APC injection | T1055.004 |
| `NtCreateProcess` / `NtCreateProcessEx` / `NtCreateUserProcess` | Create child processes (including PPID spoofing) | T1106 |
| `NtSetContextThread` | Change thread context (thread-hijacking injection) | T1055.003 |
| `NtResumeThread` | Resume a thread after injection | T1055 |
| `NtQuerySystemInformation` | Enumerate processes / drivers / handles | T1057 / T1082 |
| `NtAdjustPrivilegesToken` | Elevate privileges to obtain SeDebugPrivilege, etc. | T1134 |
| `NtLoadDriver` | Load a kernel driver (BYOVD) | T1543.003 |

### 验证 hook 是否存在

```powershell
# Simple: disassemble-diff the disk ntdll against the current process ntdll
# 1. Obtain the disk ntdll
copy C:\Windows\System32\ntdll.dll C:\temp\ntdll_clean.dll

# 2. Attach to any process in WinDbg and export the current ntdll .text section
# .writemem c:\temp\ntdll_live.bin ntdll!.text L?<size>

# 3. Disassemble NtAllocateVirtualMemory with IDA / radare2; it should normally be:
#    mov r10, rcx
#    mov eax, <SSN>
#    test byte ptr [...]
#    jne ...
#    syscall
#    ret
# If the first instruction becomes jmp <address>, it is a hook
```

## 3. Kernel Callback Monitoring Points

Common kernel callbacks registered by EDRs (all can be unregistered through the BYOVD route in `attack-chain`, but at high cost):

| API | Callback timing | Defensive use |
|-----|-----------------|---------------|
| `PsSetCreateProcessNotifyRoutineEx` | Process creation / exit | Block suspicious child processes |
| `PsSetCreateThreadNotifyRoutine` | Thread creation / exit | Detect remote-thread injection |
| `PsSetLoadImageNotifyRoutine` | DLL / EXE loaded into any process | Module integrity / unsigned-image blocking |
| `CmRegisterCallback` / `CmRegisterCallbackEx` | Registry operations | Persistence detection |
| `ObRegisterCallbacks` | `OpenProcess` / `OpenThread` handle requests | Prevent LSASS handle acquisition (T1003.001) |
| `MmRegisterPhysicalMemoryCallback` | Physical-memory mapping | Prevent DMA / memory forensics |
| `IoRegisterFsRegistrationChange` | File-system registration | Minifilter coordination |
| `KeRegisterNmiCallback` | NMI (rarely used by EDRs) | Anomaly monitoring |
| `EtwRegister` (kernel side) | Kernel ETW reporting | Works with ETW-TI |

### 用 windbg 枚举已注册 callback

```text
0: kd> dx -r1 nt!PspCreateProcessNotifyRoutine
0: kd> dx -r1 nt!PspCreateThreadNotifyRoutine
0: kd> dx -r1 nt!PspLoadImageNotifyRoutine

0: kd> !object \Callback
0: kd> !object \Callback\ProcessObject
```

或用 PChunter / DRVHV 这类工具，普通用户可视化看 callback 列表。

## 4. 静态 dump hook 表（IDA + windbg 流程）

### 流程 A：单进程比对

```text
1. 找一个已被 EDR 注入用户态组件的进程（任意已存活进程）
2. windbg attach (-pn target.exe)
3. lm m ntdll  → 拿到模块基址
4. .writemem c:\temp\ntdll_live.bin ntdll+0x0 L?<image size>
5. 把 C:\Windows\System32\ntdll.dll 复制为 c:\temp\ntdll_disk.dll
6. 在 IDA 里加载两个文件，跳到 NtAllocateVirtualMemory：
     - disk：标准 prologue
     - live：第一条 jmp <0x7FFE000000xx>
7. 跟着 jmp 目标地址 → 那就是 EDR 的 trampoline，dump 出来
8. 进 trampoline 看它最终落到哪个 DLL，确认 EDR 模块名
```

### 流程 B：批量 hook 表生成

用 `HookHunter` 或自写脚本：

```powershell
# pseudo workflow，详见 references 提到的脚本
$disk = Get-Content C:\Windows\System32\ntdll.dll -Encoding Byte
$live = # 通过 OpenProcess + ReadProcessMemory 拿
# 对比 .text 段每个 export 的前 16 字节
```

## 5. pe-sieve 自动检测

`pe-sieve` 是侦察 EDR hook 与 implant 自检的首选：

```powershell
# 基本扫描
pe-sieve64.exe /pid 1234

# 推荐组合（含 shellcode 与 hook 检测）
pe-sieve64.exe /pid 1234 /shellc 3 /modules 3 /imp 3 /data 3 /dir hooks_dump

# 关键参数：
#   /shellc N    shellcode 扫描等级 (0-3)
#   /modules N   模块完整性检查 (0-3)
#   /imp N       IAT hook 检查
#   /data N      数据段扫描
#   /dir <path>  dump 输出目录
```

输出会在 `hooks_dump/<pid>.<name>/` 下产生 `*.tag` 文件，列出 hook 地址：

```text
modified_modules.tag 示例：
71f10000;ntdll.dll
71f1a3b0;hook;jmp_far
71f1c020;hook;jmp_near
```

可直接喂给 IDA 跳到对应 RVA 做后续分析。

### 在 implant 中嵌入 pe-sieve（自检）

实战中常把 `pe-sieve` 编译为 lib (`libpe-sieve`)，让 implant 启动时先自检：如果 ntdll 有 hook，就触发 unhook 流程；如果发现自己被 hook 反而要小心，可能在沙箱里。

## 6. API Monitor v2 动态观察

API Monitor v2（Rohitab）适合在 lab 里看 EDR 在何时何处插入 hook：

```text
1. 启动 API Monitor v2（管理员）
2. API Filter 勾选：
     - NT Native API → Memory Management
     - NT Native API → Process and Thread
     - Windows Defender / AMSI（如果可见）
3. Monitor New Process → 选择 implant 测试样本
4. 观察：
     - NtAllocateVirtualMemory 调用顺序
     - 是否被 EDR DLL 中转
5. 在 Modules tab 看哪些 EDR DLL 被 LoadLibrary 注入
```

## 7. 常见 EDR DLL（用户态）速查

| DLL | 厂商 | 备注 |
|-----|------|------|
| `umppc*.dll` | Microsoft Defender | MpClient userland |
| `mpoav.dll` | Microsoft Defender | AMSI provider |
| `aswAMSI.dll` | Avast | AMSI provider |
| `eamsi.dll` | ESET | AMSI provider |
| `IDPMServiceClient.dll` | Sophos | HMPA 注入 |
| `klsihk64.dll` | Kaspersky | 注入到目标进程 |
| `CrowdStrike.Sensor.dll` | CrowdStrike | 旧版本，新版主要靠内核 |
| `SentinelInjection64.dll` | SentinelOne | 用户态注入 |
| `TmUmEvt64.dll` | Trend Micro | 行为监控 |

确认目标 EDR 后，再决定逆向哪个 DLL 取 hook 表。

## 参考链接

- pe-sieve：<https://github.com/hasherezade/pe-sieve>
- HollowsHunter：<https://github.com/hasherezade/hollows_hunter>
- API Monitor v2：<http://www.rohitab.com/apimonitor>
- MITRE ATT&CK T1562：<https://attack.mitre.org/techniques/T1562/>
- MITRE ATT&CK T1055：<https://attack.mitre.org/techniques/T1055/>
- ired.team EDR notes：<https://www.ired.team/offensive-security/defense-evasion>

## 路由回调

完成 hook 调研后，回到 `SKILL.md` 的 Step 3 选择绕过技术组合，然后按 `references/unhook-techniques.md` 与 `references/telemetry-blinding.md` 执行。
