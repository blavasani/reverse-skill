---
name: wifi-wireless
description: Use for authorized wireless security assessment including Wi-Fi capture, WPA handshake analysis, rogue AP detection research, and lab-only deauth testing.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Wi-Fi / Wireless Security

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read precedent-pentest; **wireless attacks carry significant legal risk**, so written authorization and physical scope are required
2. `NOW`: Record the target SSID/BSSID/site in scope; scanning neighboring networks is prohibited
3. `NEXT`: Confirm the adapter's monitor-mode capability
4. `ACT`: Reconnaissance -> collection -> analysis (laboratory first)

## Applicable scenarios

- Authorized Wi-Fi security assessment
- WPA/WPA2 handshake capture and offline assessment
- Rogue AP / phishing hotspot detection research
- Enterprise wireless isolation and captive-portal security

## Workflow

```text
□ iwconfig / airmon-ng 进入 monitor（合法环境）
□ airodump-ng 锁定目标 BSSID 频道
□ 握手或 PMKID 采集（仅目标）
□ hashcat/aircrack 离线评估口令策略
□ 报告：加密类型、隔离、门户绕过、建议
```

## Toolchain

| Tool | Purpose |
|------|---------|
| aircrack-ng suite | Capture/assessment |
| hcxdumptool / hcxtools | PMKID |
| hashcat | Password assessment |
| Wireshark | Management frame analysis |

## References

- `references/wireless-lab-rules.md`
- `../pentest-tools/` `../attack-chain/` (close-access sections)

## Routing context

**Upstream**: MASTER R29
**MUST NOT**: Perform unauthorized deauth or operate on non-target client networks

## Task completion checklist

- [ ] Is activity strictly restricted to the target BSSID?
- [ ] Does the report include hardening recommendations?
- [ ] Checklist?