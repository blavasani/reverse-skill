---
name: protocol-reverse
description: Use for authorized reverse engineering of custom binary protocols, Protobuf/gRPC, WebSocket frames, and PCAP-driven protocol recovery.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Protocol Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` to confirm authorization and routine operational boundaries.
2. `NOW`: Confirm the task involves **protocol, traffic, or serialization format** reverse engineering; route pure Web parameter signing to `js-reverse/`.
3. `NOW`: For target network interaction, initialize scope with `../scripts/case-init.ps1`; do not ACT on the target unless `auth` is granted.
4. `NEXT`: Read `../tool-index.md`; bootstrap missing tools (tshark/wireshark and others may require manual installation).
5. `ACT`: Begin workflow Phase 1 and draft a frame layout or message dictionary.

## Applicable Scenarios

- Custom TCP/UDP binary protocols
- Protobuf / gRPC / FlatBuffers / MessagePack
- WebSocket / MQTT / private RPC
- Recovering fields and state machines from PCAP / PCAPNG
- Client-server validation, sequence numbers, and encrypted frame headers

## Routing Exclusions

| Scenario | Route |
|------|------|
| HTTP parameter signing / JS encryption only | `js-reverse/` |
| TLS certificate issues only | `pentest-tools/` or a browser proxy |
| Deep analysis and emulation of a firmware protocol stack | `firmware-pentest/`, then return to this skill |

## Workflow

### Phase 1 — Collection and Triage

```text
□ 拿到样本：PCAP / 代理导出 / 客户端日志 / 二进制
□ 标记方向：C→S / S→C；是否有握手、心跳、重连
□ 固定头？魔数？长度字段？TLV？定长？
□ 是否压缩（zlib/gzip/lz4）或加密（AES/ChaCha 帧内）
□ tshark -r cap.pcap -T fields -e frame.number -e ip.src -e tcp.payload
```

### Phase 2 — Frame Layout Recovery

```text
□ 对齐多个同类消息，找不变字节 / 自增序列号
□ 长度字段：大端/小端、含头/不含头
□ 校验：CRC16/32、checksum、HMAC 位置
□ 画出状态机：Connect → Auth → Ready → Request/Response → Close
□ 工具：Wireshark 自定义 dissector 草稿 / ImHex / 010 Editor 模板 / Kaitai Struct
```

### Phase 3 — Serialization and Encryption

```text
□ Protobuf：.proto 恢复（blackboxprotobuf / pbtk / protoc --decode_raw）
□ gRPC：HTTP/2 headers + protobuf body
□ 加密：找密钥派生（客户端 so/dll/JS）→ 联合 ida-reverse / js-reverse / apk-reverse
□ 重放：仅在授权 scope 内；先无害字段再敏感操作
```

### Phase 4 — Deliverables

```text
MUST 产出：
- 消息类型表（name / opcode / fields）
- 至少 1 条可复现的解码命令或脚本
- Evidence：原始 hex 摘录 + 解码结果（脱敏）
```

## Toolchain

| Tool | Required | Purpose | Bootstrap |
|------|------|------|------|
| tshark / Wireshark | Strongly recommended | PCAP parsing | Manual / winget |
| Python3 | Yes | Decoding scripts | System |
| blackboxprotobuf | Optional | Unknown protobuf | pip |
| ImHex / 010 | Optional | Structure templates | Manual |
| IDA / r2 / Ghidra | As needed | Client serialization functions | See the corresponding skill |

## References

- `references/protocol-workflow.md` — Frame layout and Protobuf quick reference
- Related: `../ida-reverse/` `../js-reverse/` `../firmware-pentest/` `../pentest-tools/`

## Routing Context

**Upstream**: `MASTER-ROUTING` R21 · `routing.md`
**Downstream**: Client algorithm required → `ida-reverse`/`js-reverse`; replay exploitation required → `pentest-tools`/`api-security`
**Peers**: `malware-analysis` (C2 protocols), `digital-forensics` (traffic forensics)

## Completion Checklist

- [ ] Did you recover a message layout or state machine rather than only pasting hex?
- [ ] Is there a reproducible decoding command?
- [ ] Did you comply with scope and redaction requirements?
- [ ] Did you update the field journal and report checklist?