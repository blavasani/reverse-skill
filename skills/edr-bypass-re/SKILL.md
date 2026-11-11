---
name: edr-bypass-re
description: "Use the edr-bypass-re specialist workflow after routing and authorization. Follow its ACTION REQUIRED steps and preserve evidence provenance."
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

## ACTION REQUIRED

1. `NOW`: Read `../field-journal/precedent-reverse.md` and confirm that this skill’s operations are authorized routine operations.
2. `NOW`: Confirm that the current task is within this skill’s scope.
3. `NEXT`: Read `../tool-index.md` and verify tool availability and actual paths.
4. `NEXT`: Invoke bootstrap when tools are missing; do not guess paths.
5. `ACT`: Enter and execute Step 1 of the workflow; do not stop at confirmation.

# EDR Bypass: Defensive Implementation Reverse Engineering to Red-Team Evasion

> Authorized red-team operations, adversary simulations, and testing of owned products only; do not use against unauthorized targets.

## Scope

Use this skill for authorized red-team or adversary-simulation work that delivers an implant to an approved target host and evaluates evasion of modern EDR.

1. **Red team / Purple team / adversary simulation** — assess SOC and EDR detection capability.
2. **In-house implant / C2 framework development** — develop payloads for testing owned products or EDR controls.
3. **EDR product assessment** — objectively evaluate detection coverage within confirmed compliance boundaries.
4. **CTF / adversary-simulation Windows breakout** — execute reliably on hardened hosts during a competition.

**Out of scope**:

- Full RE of an antivirus vendor’s product for a commercial report without formal vendor cooperation.
- AV evasion against unauthorized targets.
- AV evasion for ordinary malware; this skill focuses on red-team OPSEC and does not teach malicious-code implementation.

### Division of responsibility with other skills

| Scenario | Use |
|----------|-----|
| Full-chain attack and defense (external access to domain control) | `attack-chain/` |
| Internal lateral movement / AD attacks | `pentest-tools/network-attack-defense.md` |
| EDR evasion for implant delivery on a specific host | **This skill** |
| Static AV evasion only (obfuscation / packing) | `malware-analysis/` (reverse perspective) |

`attack-chain` covers the complete kill chain; this skill focuses only on the **EDR opponent**, its internal mechanisms, and targeted evasion methods.

## Core Principles

```text
EDR monitoring surface              Red-team countermeasure
─────────────────────              ─────────────────────
User-mode ntdll hook     ◄──►   unhook (Peruns Fart / fresh ntdll)
                                  indirect syscall / Hell's Gate
                                  hardware-breakpoint Blindside

Kernel callback          ◄──►   call-stack spoofing
(Ps/Cm/Ob families)                use legitimate trigger chains (do not directly bypass; pair with upstream concealment)

ETW telemetry            ◄──►   EtwEventWrite patch
(Microsoft-Windows-Threat-         NtTraceControl to disable the provider
 Intelligence, etc.)              synchronize AmsiContext handling

AMSI scanning            ◄──►   AmsiScanBuffer patch (mov eax,0x80070057; ret)
(amsi.dll)                         hardware-breakpoint bypass
                                  reflective loading of a copy of amsi.dll
```

Key points:

- **EDR is not a black box** — key hooks, callbacks, and providers can be reverse engineered with IDA and WinDbg.
- **Combine evasion techniques** — one unhook does not solve ETW alerts, and one AMSI patch does not solve syscall hooks.
- **Order matters** — ETW patch → AMSI patch → unhook; in the wrong order, the EDR receives the unhook alert first.
- **Modern EDR treats ETW and kernel callbacks as primary battlefields**; user-mode unhooking alone is no longer sufficient.

## Workflow

### Step 1: Identify the target host’s EDR

```powershell
# List common EDR / AV drivers
Get-Service | Where-Object {$_.Name -match 'CSAgent|SentinelAgent|elasticendpoint|esets|ekrn|MsMpEng|wdsvc|cyserver|sysmon|aswbidsagent'}

# List loaded minifilters
fltmc filters

# List registered kernel callbacks (requires WinDbg + kernel debugging, or PChunter / DRVHV)
# !object \Callback
# !pnpcallback / Process / Thread / Image
```

See the EDR fingerprint table at the top of `references/hook-survey.md`.

### Step 2: Extract the hook table from the EDR DLL

1. Attach to a process containing an injected EDR user-mode component (any deployed process).
2. Dump the current `ntdll.dll` `.text` section in WinDbg.
3. Diff it against the clean `C:\Windows\System32\ntdll.dll` on disk.
4. Differences identify hook locations.

Alternatively, use `pe-sieve`:

```powershell
pe-sieve64.exe /pid 1234 /shellc 3 /modules 3 /dir hooks_dump
```

See `references/hook-survey.md` for details.

### Step 3: Select a combination of evasion techniques

| Defense point | Recommended technique |
|---------------|----------------------|
| ntdll inline hook | indirect syscall + dynamic SSN (Halo's Gate) |
| ETW-TI provider | EtwEventWrite head patch |
| AMSI (PowerShell / .NET) | AmsiScanBuffer patch or HWBP |
| Kernel callback | call-stack spoofing + legitimate gadget |
| Sysmon ProcessCreate | PPID spoofing + unbacked memory |

### Step 4: Implement in the implant

See the code skeletons in `references/unhook-techniques.md` and `references/telemetry-blinding.md`.

### Step 5: Validate in a local sandbox

```powershell
# Deploy the target EDR trial in an isolated environment (Defender defaults are sufficient to start)
# Enable Sysmon + olaf-config
sysmon64.exe -i sysmonconfig.xml

# Run the implant and check whether these alert sources trigger:
#   - Defender AMSI
#   - ETW-TI
#   - Sysmon Event ID 1/7/8/10
#   - EDR console
```

### Step 6: Deliver

- Use a legitimate software directory for the dropped-file path.
- Spoof the PPID to explorer.exe.
- Coordinate with the initial-access section in `attack-chain`.

## Typical Scenarios

### Scenario 1: Deliver a cobalt-strike-alike beacon past Defender + Sysmon

```text
Target: Windows 11 Enterprise + Defender (cloud protection enabled) + Sysmon (olaf configuration)
Requirement: beacon callbacks after delivery without triggering any alerts

Combination:
  1. Store encrypted shellcode and decrypt it at runtime
  2. AMSI patch (if delivery uses PowerShell)
  3. EtwEventWrite patch (suppress ETW-TI)
  4. Indirect syscall + Halo's Gate (suppress ntdll-hook alerts)
  5. Spoof PPID to explorer.exe
  6. Encrypt the implant’s memory during sleep with Ekko / Foliage
```

### Scenario 2: Apply an EDR sleep mask on an already deployed low-privilege shell

```text
Prerequisite: phishing has obtained a medium-IL shell and the EDR is monitoring
Risk: long residence makes beacon characteristics easy to find through memory scanning

Solution:
  1. Do not request new RWX memory
  2. Use Ekko during sleep:
       - WaitForSingleObjectEx + CreateTimerQueueTimer
       - Encrypt the implant’s .text and zero the stack in the timer
  3. Restore with ROP on wake
  4. Pair with call-stack spoofing so RtlCaptureStackBackTrace does not see beacon addresses
```

## On-Demand Bootstrap

### Tool Dependencies

| Tool | Purpose | Automatically installable |
|------|---------|---------------------------|
| pe-sieve | Detect hooks / injection in processes | ✓ |
| API Monitor v2 | Dynamically observe API calls and hooks | Semi-automatic (manual download) |
| SysWhispers3 | Generate direct / indirect syscall stubs | ✓ (git clone + python) |
| Hell's Gate POC | Reference implementation for dynamic SSN resolution | ✓ (git clone) |
| WinDbg + IDA | Static reverse engineering of EDR DLLs / kernel callbacks | ✗ (manual installation) |
| Sysmon + olaf config | Local validation environment | ✓ |

### Bootstrap Command

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "&lt;SKILL_ROOT&gt;\skills\scripts\bootstrap-reverse.ps1" -Capability @('pe-sieve','syswhispers3','sysmon') -StartServices
```

## Routing Context

**Upstream entry points**:

- `reverse-engineering/` — understand the EDR DLL / driver implementation first.
- `attack-chain/` — determine where to introduce this skill in the kill chain.

**Related skills**:

- `pentest-tools/network-attack-defense.md` — coordinate this skill during internal lateral movement.
- `malware-analysis/` — reverse perspective on how the detection side writes rules.
- `field-journal/` — record lessons learned after each engagement.

**Downstream deliverables**:

- Cite MITRE ATT&CK **T1562 (Impair Defenses)**, T1562.001 (Disable or Modify Tools), T1562.006 (Indicator Blocking), T1055 (Process Injection), and T1027 (Obfuscated Files or Information) in reports.

## Legal Boundaries

- Authorized red-team operations, adversary simulations, and testing of owned products only.
- Obtain written authorization before operations (SoW / test contract / SRC scope statement).
- Do not use against unauthorized targets or exceed the authorized scope.
- Report high-severity issues to the client immediately and follow responsible disclosure.
- Redact real target information in all reports (IP addresses / hostnames / domains / credential placeholders).

## References

- Detailed hook survey: `references/hook-survey.md`
- Unhook / syscall techniques: `references/unhook-techniques.md`
- ETW / AMSI / anti-forensics: `references/telemetry-blinding.md`
- MITRE ATT&CK T1562: <https://attack.mitre.org/techniques/T1562/>

## Completion Self-Check (MUST pass before claiming completion)

- [ ] Was every workflow step executed rather than merely read?
- [ ] Were real tool paths used based on `tool-index`?
- [ ] Was reproducible evidence produced (commands/scripts/screenshots/report)?
- [ ] Were the Checklist items required by RULES completed and recorded?
