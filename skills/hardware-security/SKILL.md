---
name: hardware-security
description: Use for authorized hardware and embedded interface security research including UART/JTAG discovery, debug pad triage, secure boot overview, and offline firmware extraction support.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Hardware / Embedded Interface Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Confirm **authorization for physical access** and device ownership
2. `NOW`: ESD/power safety; default to read-only probing
3. `NEXT`: Coordinate with firmware-pentest for image analysis
4. `ACT`: Identify the enclosure and debug interfaces → consoles → extraction

## Applicable Scenarios

- UART / JTAG / SWD debug-port discovery
- Boot logs, root shells, and boot interruption
- Flash extraction during authorized teardown
- Feasibility assessment for secure boot/encrypted Flash (prioritize non-destructive methods)

## Workflow

```text
□ 拆解授权设备；拍照标注测试点
□ 万用表找 GND/VCC/TX/RX；逻辑电平 1.8/3.3/5V
□ USB-TTL 只读日志；记录波特率
□ JTAG：枚举 IDCODE；评估是否锁定
□ 提取镜像 → 交接 firmware-pentest / ghidra
```

## Toolchain

| Tool | Purpose |
|------|---------|
| USB-TTL / logic analyzer | UART |
| J-Link / CMSIS-DAP | Debugging |
| bus pirate / flipper (lab) | Multi-protocol |
| binwalk / flashrom | Extraction |

## References

- `references/debug-interface-triage.md`
- `../firmware-pentest/` `../ot-ics/`

## Routing Context

**Upstream**: MASTER R34
**MUST NOT**: Unauthorized teardown or damage to another person's device

## Completion Checklist

- [ ] Were interface voltage levels and the pinout recorded?
- [ ] Was the image hash preserved?
- [ ] Checklist complete?