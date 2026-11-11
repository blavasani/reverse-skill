# Vendor Report Rules (vendor-report structure overlay)

> Issue #65, item 2.
> **Extract structure and writing rules only; do not reproduce vendor report text, charts, real IOC examples, or long passages.**
> This file is an **overlay**: it does not replace the task templates in `security-report-templates.md` or weaken §0 Evidence→Finding→Path.

Structural references (public examples, outline only):

| Flavor | Primary reference | Scenario |
|--------|-------------------|----------|
| `malware` | Huorong virus/technical analysis reports | Explicit commodity trojans, dual-use loaders, phishing delivery, malicious samples |
| `apt` | Kaspersky Securelist / APT campaign reports (such as MATA) | APTs, group campaigns, multi-stage infection chains, industry targeting |

Principle: **Prefer precise templates over many templates** — only two full vendor flavors (`malware` / `apt`) plus Base common elements and an **optional thin overlay** (such as `vuln` technical vulnerability analysis). Ordinary reverse engineering, penetration testing, CTF, and JS reports retain their task templates and are not disguised as malware reports by default; `vuln` is **not** a third default full flavor.

---

## 0. When to enable

When `docs-generator` generates a **security** report (reverse engineering / malware / penetration-test closeout / an explicit request for a “professional report” or “vendor style”), it **MUST** read this file. Select a vendor flavor only when supported by task evidence or explicitly requested by the user; otherwise use `flavor = null` and add only the common professional elements to the original task template.

| Signal | Flavor / Overlay |
|--------|------------------|
| APT / group / campaign / multi-stage C2 / industry targeting / ICS / spear-phishing campaign | `apt` |
| Explicit malicious sample, trojan, information stealer, dual-use loader, impersonation site | `malware` |
| User explicitly requests vulnerability/patch/CVE technical analysis, or task evidence concerns OS/component vulnerability research | `flavor = null` + **thin overlay `vuln`** (see §3b) |
| Ordinary APK/ELF/PE/Mach-O reverse engineering, algorithm analysis, firmware analysis, penetration testing, CTF, JS signatures | `flavor = null`; use the original task template and the minimum common professional elements |

An explicit user request for “Kaspersky/APT style,” “Huorong/virus-report style,” or “technical vulnerability analysis” overrides automatic selection.
**Do not** place ordinary malware/APT or ordinary reverse engineering into the `vuln` outline by default.

---

## 1. Common professional elements (Base)

Apply the following Base elements by report type. Items marked **MUST** cannot be omitted; elements tied to a specific flavor must not appear in unrelated tasks merely to fill the template. When no content applies, use `n/a` and explain why.

| # | Element | Requirement |
|---|---------|-------------|
| G1 | Executive summary / overview | **MUST**: 3–8 sentences covering what was analyzed, the most severe conclusion, impact, and recommended action |
| G2 | Scope and authorization | **MUST**: link to the case `scope.md` (see template §0.1) |
| G3 | Evidence→Finding→Path | **MUST**: see §0 of `security-report-templates.md` and `skills/ops/evidence-finding-path.md` |
| G4 | IOC table | **MUST** for `malware` / `apt`; include for other tasks only when relevant indicators exist |
| G5 | Recommendations / response | **MUST** for `malware` / `apt`: at least one actionable recommendation; follow the original task template for other tasks |
| G6 | Appendix metadata | **SHOULD**: tools and versions, sample hashes, complete reproduction commands |
| G7 | ATT&CK mapping | **MUST** under `apt` (use `n/a` plus a reason when no technique applies); **SHOULD** for other tasks |

### 1.1 Minimum IOC table columns

```markdown
| Type | Value | Context | First/last seen | Source evidence | Confidence |
|------|-------|---------|-----------------|-----------------|------------|
| file_sha256 / file_md5 / domain / ip:port / url / mutex / path / registry | … | Where found | YYYY-MM-DD / n/a | E-id | high/med/low |
```

### 1.2 Copyright and security boundaries

- Do not paste vendor PDF/webpage paragraphs or figure captions as if they were your own analysis.
- Use placeholders for real tokens, internal URLs, and customer identifiers.
- Do not output directly exploitable attack-step details for unauthorized targets (follow the case scope / RULES).

---

## 2. Flavor: `malware` (Huorong style · explicit selection)

**Narrative goal**: Let readers understand within five minutes “what it is → how it arrived → what the sample did → how to respond → which IOCs exist.”

### 2.1 Recommended section order

```markdown
# [标题：一句话威胁定性]

> 分析日期 / 分析方 / 样本标识（哈希）

## 1. 概述
（G1：发现渠道、伪装手法、核心技术点、产品侧可否查杀——若未知写 n/a）

## 2. 攻击 / 感染流程
（流程图：Mermaid 或分步列表；对应 Path `path_type=attack`）

## 3. 样本分析
### 3.1 样本溯源
### 3.2 静态分析
（**MUST** 纳入导入表 / 基础身份 Evidence：E-imports 或等价；见 radare2/ida/malware 硬门）
### 3.3 动态分析 / 行为
（无动态条件则 n/a + 原因）
### 3.4 核心发现（Findings 表或编号列表，挂 evidence_ids）

## 4. 应急处置方式
（仅在授权范围内执行：先确认 scope 并保全样本、内存、进程树、网络连接和日志等证据，再隔离主机；经负责人批准后再终止进程、隔离/清除文件、检查 hosts/启动项、全盘查杀并复核。不得在证据保全前直接删除文件。）

## 5. 总结说明
（给普通用户/运维的风险提醒与预防）

## 6. IOC 信息
（G4 表）

## 7. Evidence 链摘要
（§0：E / F / P / Timeline；可与 §3.4 合并但字段不省）

## 8. 附录
（工具版本、复现命令、脚本路径）
```

### 2.2 Style

- Lead with conclusions, then provide details.
- Organize static analysis by “component/stage” and avoid pasting long, unstructured logs.
- Response steps must be independently executable; do not pad them with empty “raise security awareness” statements.

---

## 3. Flavor: `apt` (Kaspersky Securelist style)

**Narrative goal**: Tell the campaign-level story clearly—who used which chain against whom and when, how the investigation progressed, how components were divided, and what defenders can use to detect it.

### 3.1 Recommended section order

```markdown
# [战役/集群名称]：[一句话影响]

> 日期 / 团队 / 行业与地区范围（若可知）

## 1. Executive summary
（G1：时间窗、受害者画像、入口、家族/集群归属、持续时长、最重要结论）

## 2. The infection chain
（分阶段：投递 → exploit/loader → 主马 → 后渗透/窃密；未知段明确 “limited visibility”
对应 Path；建议配链图）

## 3. Incident investigation
（调查叙事：关键转折、内网代理/C2 特征、如何扩大范围；挂 Timeline）

## 4. Interesting findings
（3–7 条非显而易见要点，每条尽量挂 E-id / F-id）

## 5. Technical analysis
### 5.1 组件总览表（loader / trojan / stealer / …）
### 5.2 分组件行为与配置
### 5.3 静态要点（含导入表/加壳/持久化 Evidence）
### 5.4 网络与 C2
（可附 ATT&CK 表 G7）

## 6. Detection and mitigation
（检测思路 / 狩猎线索 / 缓解优先级；非空泛口号）

## 7. IOC
（G4；按类型分组）

## 8. Evidence 链摘要
（§0 字段）

## 9. Appendix
（样本列表与哈希、工具版本、参考公开编号；不抄外部报告正文）
```

### 3.2 Style

- Describe the timeline and “visibility limitations” honestly.
- Interesting findings ≠ a repeated overview; write the genuinely important anomalies found during the investigation.
- Use a table for component analysis: role / persistence / C2 / dependencies, then expand on it.

---


## 3b. Thin overlay: `vuln` (technical vulnerability analysis · optional)

> Issue #65 supplement. The structure references public technical-analysis report outlines for operating-system/component vulnerabilities; **extract section skeletons only** and do not reproduce PoC messages, exploitation details, or unauthorized attack steps from screenshots or report text.
> **Not** a third default full vendor flavor; add it only for vulnerability-research tasks or when explicitly requested.

**Narrative goal**: Let readers quickly see “who is affected → how to confirm/reproduce (within authorization) → root cause and patch differences → how to mitigate.”

### Recommended section order

```markdown
## 1. 漏洞概述
### 1.1 影响范围（版本/组件/配置前提）
### 1.2 漏洞复现（授权环境；步骤可第三方重复；无武器化教程口吻）

## 2. 漏洞分析
### 2.1 崩溃 / 异常分析（Evidence：崩溃日志、触发条件）
### 2.2 补丁分析（diff/守卫条件/修复点 — 挂 E-*）
### 2.3 PoC 或触发器分析（仅授权范围内已有材料；协议/输入构造层次说明即可）

## 3. 防护建议
### 3.1 缓解措施（配置/缓解开关等）
### 3.2 官方补丁与验证

## 4. Evidence → Finding → Path（可并入各节或独立表）
```

### Hard constraints

- **MUST** enforce scope/authorization: reproduction and PoC expansion are prohibited for unauthorized targets.
- **MUST** enforce E/F/P: reproduction, crash, and patch conclusions all carry `evidence_ids`.
- **MUST NOT** treat `vuln` as the default malware/APT shell.
- **MUST NOT** reproduce exploit code or complete weaponized attack steps from external reports/screenshots.
- IOC table: include it only when network/file indicators exist; otherwise use `n/a` or omit it.

---
## 4. Integration with existing task templates

| Task template (`security-report-templates.md`) | Overlay behavior |
|-----------------------------------------------|------------------|
| 1. Reverse engineering report | Default `flavor = null`; retain the original “static/dynamic/reproduction” outline and hard-gate Evidence such as the imports table; use §2 only for an explicit malicious sample |
| 2. Penetration-test report | `flavor = null`; add applicable G1–G3 from Base, align the attack path with §0 Path, and do not require IOCs |
| 3. CTF writeup | `flavor = null`; retain the original challenge, solution approach, and reproduction structure; do not require IOC/ATT&CK |
| 4. JS/Web signature reverse engineering | `flavor = null`; use the original overview → localization → algorithm → reproduction outline, not malware |
| Malware / APT specialization | Explicitly select the `malware` or `apt` full outline |

**Conflict resolution**: §0 Evidence-chain fields and scope gates **always take precedence**; a flavor changes only narrative order and professional framing and must not remove E/F/P.

---

## 5. Selection pseudocode

```
if user_requests_kaspersky or apt or threat_campaign:
    flavor = apt
elif user_requests_huorong or vir_report or explicit_malware:
    flavor = malware
else:
    flavor = null  # 原任务模板 + Base 中适用的元素
overlay = null
if user_requests_vuln_tech_report or cve_patch_analysis:
    overlay = vuln  # thin only; never a third default full flavor
emit(base_report)
if flavor in (malware, apt):
    emit(report with flavor outline)
elif overlay == vuln:
    emit(report with vuln thin outline)
```

---

## 6. Completion checklist (self-check at the end of the report)

- [ ] A flavor or explicit “task template + minimum set” is selected
- [ ] G1 overview exists and is substantive
- [ ] §0 E/F/P fields are complete
- [ ] `malware` / `apt` reports include an IOC table (or `n/a` plus a reason)
- [ ] `malware` / `apt` reports include actionable recommendations/response steps
- [ ] Tasks without a flavor are not placed into malware/APT-specific sections
- [ ] `vuln` is enabled only for vulnerability tasks; it includes overview/analysis/mitigation skeletons and E/F/P, with no unauthorized PoC weaponization
- [ ] No vendor-text pasting and no placeholder/TODO
- [ ] Hard-gate Evidence such as the imports table is included in static/technical analysis when binary analysis was performed

---

## 7. Source register

- Kaspersky Securelist, “Updated MATA attacks industrial companies in Eastern Europe”: <https://securelist.com/updated-mata-attacks-industrial-companies-in-eastern-europe/110829> (structural reference; accessed 2026-08-11)
- Huorong public technical-article entry point: <https://www.huorong.cn/> (site entry point; accessed 2026-08-11. Record the specific article URL, title, and access date when actually citing it)
- ATT&CK technique IDs are only normalized mappings and must be supported by the current Evidence; do not automatically import IOCs from external reports into the current report.

---

## 8. Non-goals

- Do not maintain additional full templates for Mandiant/CrowdStrike/Qianxin or similar vendors (the two flavors plus optional thin overlay cover common needs).
- Do not promote `vuln` to a default full flavor parallel to malware/apt.
- Do not crawl vendor sites automatically to populate reports.
- Do not reduce the Evidence contract or authorization scope because of a flavor.
