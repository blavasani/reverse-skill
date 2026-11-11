---
name: digital-forensics
description: Use for authorized digital forensics including memory dumps, disk timelines, PCAP investigation, artifact triage, and IR evidence preservation.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Digital Forensics & IR Artifacts

## ACTION REQUIRED

1. `NOW`: Read `../field-journal/precedent-pentest.md` or the organization’s IR authorization statement.
2. `NOW`: Confirm that this is **forensics/attribution**, not offensive scanning.
3. `NOW`: Create a case; prioritize read-only evidence copies and write-protect original media.
4. `NEXT`: Read tool-index; Volatility and similar tools are often manual.
5. `ACT`: Preserve hashes → build a timeline → analyze key artifacts.

## Use Cases

- Memory-dump analysis (Volatility 2/3)
- Disk / E01 / file timeline analysis
- PCAP attribution and protocol reconstruction (can be combined with `protocol-reverse/`)
- Host artifacts: Prefetch, Shimcache, Event Log, and browser history
- Incident-response IOC extraction (combined with `malware-analysis/` / `threat-hunting/`)

## 工作流

### 1. Preservation

```text
□ Compute SHA256; record the time zone and collection command
□ Work on copies; keep originals read-only
□ Record chain-of-custody notes in the timeline
```

### 2. 内存

```bash
vol -f mem.dmp windows.info
vol -f mem.dmp windows.pslist
vol -f mem.dmp windows.netscan
vol -f mem.dmp windows.cmdline
```

### 3. Host Artifacts

```text
□ Event logs: Security / PowerShell / Sysmon
□ Persistence: Run keys, services, scheduled tasks, WMI
□ Execution traces: Amcache, Prefetch, BAM
```

### 4. Network

```text
□ Use tshark to summarize sessions and DNS
□ Export suspicious streams → protocol-reverse or malware C2 analysis
```

## Toolchain

| Tool | Purpose |
|------|---------|
| Volatility 3 | Memory |
| Timeline Explorer / Plaso | Super timeline |
| tshark | PCAP |
| Eric Zimmerman tools | Windows artifacts |
| Autopsy / FTK Imager | Disk |

## References

- `references/forensics-triage.md`
- `../malware-analysis/` `../threat-hunting/` `../protocol-reverse/`

## Routing Context

**Upstream**: MASTER R25
**Downstream**: deep malware-sample analysis → malware-analysis; rules → threat-hunting

## Completion Self-Check

- [ ] Were hashes and the copy strategy preserved?
- [ ] Is the timeline reproducible?
- [ ] Are IOCs redacted and classified?
- [ ] Checklist completed?
