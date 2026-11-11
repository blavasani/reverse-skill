# Automatic Routing Rules for Reverse Engineering / Penetration / Security Tasks (Kali Linux Edition)

> **This file is the Kali path adaptation layer, not a second behavior chain.** Behavior and authorization follow the repository-root `RULES.md`.
> The core knowledge base (`skills/config/routing.json`, SKILL.md, and references) is shared with the Windows edition.
> **Do not** write this file to `~/.claude/CLAUDE.md` or any other client-global configuration. Core scripts must not write client-global files.

Hot path (same as `RULES.md`): `skills/scripts/master-route.sh` → `case-init.sh` (do not ACT on targets before `auth.status=granted`) → PRIMARY `SKILL.md`. Identity: `skills/ops/IDENTITY.md`. Scripts use `kali/scripts/*.sh` in this directory.

---

## Trigger Keywords (Identical to the Windows Edition)

- APK、Android 逆向、反编译、smali、jadx、apktool、Frida、Hook
- 二进制分析、IDA、radare2、r2、反汇编、逆向工程、RE、还原源码、源码还原、逆向还原
- 前端签名、加密参数、JS 逆向、jshookmcp、CDP、SourceMap
- 抓包、HTTP 捕获、请求重放、anything-analyzer
- CTF、Pwn、Web 渗透、漏洞利用、提权
- MCP 逆向工具、idalib-mcp
- 重打包、签名、证书校验、root 检测、反调试
- so 分析、native hook、JNI
- 渗透测试、红队、安全评估、蓝队、应急响应
- 写报告、写文档、出报告、writeup、技术文档、渗透报告、逆向报告
- 浏览器自动化、打开网页、填表、爬取、截图、自动化登录、Playwright、agent-browser、headless
- 符号迁移、bindiff、跨版本、PDB 缺失、函数偏移迁移、symbol migration、版本对比、旧版符号
- N-day、Nday、补丁差分、patch diff、patch tuesday、1day、CVE 复现、漏洞还原、ghidriff、Diaphora、DeepDiff、补丁分析
- pwn、栈溢出、堆溢出、ROP、ret2libc、ret2csu、one_gadget、libc-database、tcache、fastbin、kernel pwn、SMEP、SMAP、KASLR、modprobe_path、commit_creds、pwntools、GEF、pwndbg
- firmware、IoT、binwalk、unblob、squashfs、UBI、JFFS2、Firmadyne、FAT、QEMU 全系统仿真、EMBA、固件渗透、路由器固件、嵌入式漏洞利用、AFL++、boofuzz、UART、JTAG
- BurpSuite、Burp MCP、Intruder、Repeater、Collaborator、代理历史分析
- LLM 安全、AI 安全测试、Prompt 注入、jailbreak、Agent 安全、garak、PyRIT
- API 安全测试、GraphQL 安全、JWT 攻击、供应链安全、SBOM、Trivy
- iOS 逆向、Objection、YARA、恶意软件分析、AI 反编译、LLM4Decompile
- Agent 不干活、AI 懒、跳过步骤、Prompt 工程、Agent 服从性
- EDR 绕过、AV bypass、unhook、direct syscall、indirect syscall、Hell's Gate、SysWhispers、ETW patch、AMSI patch、call stack spoofing、MITRE T1562、CrowdStrike 绕过、Defender 绕过、SentinelOne 绕过、pe-sieve
- 端口扫描、Nmap、漏洞扫描、Nuclei、SQL 注入、SQLMap、目录爆破、FFUF、密码破解、Hashcat、Hydra、Metasploit、Impacket、pentestMCP
- SRC、Bug Bounty、众测、漏洞赏金、HackerOne、WAF bypass、绕过 WAF、IDOR、越权、任意账号
- 画图、流程图、架构图、攻击路径图、时序图、状态图、数据流图、Mermaid、Graphviz、PlantUML、diagram
- 恶意软件分析、病毒分析、样本分析、沙箱、YARA、IOC
- 内核驱动、Rootkit、LKM、IOCTL、DeviceIoControl
- 密码学、加解密、AES、RSA、哈希碰撞、签名验证
- 协议逆向、自定义协议、Protobuf、序列化
- 固件逆向、IoT、binwalk、ARM、MIPS、嵌入式
- WASM、WebAssembly、Python 字节码、pyc、.NET、dnSpy、IL
- macOS、iOS、Mach-O、ObjC、Swift、Frida iOS
- Go 逆向、Rust 逆向、stripped binary、GoReSym
- 内存转储、memory dump、取证、forensic、隐写、steganography
- 云安全、容器逃逸、K8s、Docker、AWS、Azure
- Prompt 注入、AI 安全、Agent 安全、LLM 攻击
- 内网渗透、横向移动、Pass-the-Hash、域渗透、AD 攻击、BloodHound
- 权限提升、提权、SUID、Potato、UAC bypass
- 凭证提取、Mimikatz、Kerberoasting、DCSync、LSASS
- C2、远控、持久化、后门、Cobalt Strike、反弹 shell
- 蓝队、检测、防御、应急响应、SIEM、EDR、威胁狩猎、IOC
- 移动安全测试、OWASP MASTG、APP 安全、脱壳、加固分析
- SSTI、模板注入、SSTImap、XSS、XSStrike、跨站脚本
- WordPress、WPScan、WPProbe、CMS 渗透
- AdaptixC2、C2 框架、对抗模拟、红队模拟、Atomic Red Team
- WiFi 攻击、无线渗透、Fluxion、aircrack-ng、deauth
- NTLM relay、Coercer、认证强制、PetitPotam
- WinRM、evil-winrm、Windows 远程执行
- NetExec、nxc、CrackMapExec、SMB 枚举
- AI 自动渗透、HexStrike、MetasploitMCP、mcp-kali-server
- Pentest Swarm、pentestswarm、群体渗透、Swarm AI、自主扫描、stigmergy
- Bug Bounty 自动化、攻击面管理、ASM、持续监控
- GEF、GDB 增强、调试框架
- Wireshark、tshark、PCAP 分析、抓包分析
- BurpSuite、Web 代理、拦截请求、Intruder
- Responder、LLMNR 投毒、NBT-NS、MDNS
- BloodHound、AD 路径、攻击图、SharpHound
- Certipy、AD CS、证书攻击、ESC1、ESC8
- wfuzz、参数模糊、Web Fuzz
- objdump、strings、file、静态分析
- ProxyCat、代理池、IP 轮换
- 红队、HW、攻防演练、打点、初始突破、边界突破
- 完整渗透、全流程渗透、从外网打到内网、从外打到域控
- 攻击面评估、攻击路径规划、攻击链、kill chain
- 拿到 shell 下一步、后渗透、据点扩展、纵深渗透
- 近源渗透、BadUSB、Rubber Ducky、WiFi Pineapple、Proxmark3、RFID 克隆
- EDR 绕过、免杀、AV bypass、Shellcode 加载器、无文件攻击
- 钓鱼邮件、社会工程、OAuth 钓鱼、HTML 走私
- 供应链攻击、组件投毒、第三方渗透
- 痕迹清理、反取证、日志清除、时间戳修改
- Cobalt Strike、Sliver、Havoc、Mythic、C2 框架

---

## Routing Entry Point

> **Detection method**: the parent directory of the directory containing this file (`RULES-kali.md`) is the package root.

Hot path (same as `RULES.md` / `routing.json`):

1. `skills/scripts/master-route.sh -Hint "<task>"` — PRIMARY
2. `skills/scripts/case-init.sh` — `scope.md`; do not ACT on targets before `auth.status=granted`
3. PRIMARY `SKILL.md` ACTION REQUIRED
4. `skills/tool-index.md` — canonical paths; if missing, use `kali/scripts/bootstrap-reverse.sh`

---

## Operating Principles (Same as the Windows Edition, Different Commands Only)

### Tool Usage
- **Never guess tool paths**; read `tool-index.md` first.
- When a tool is missing, call `bootstrap-reverse.sh` to install it automatically.
- Kali has many tools preinstalled, so bootstrap failures are much less likely than on Windows.
- After automatic installation of the same tool fails twice, stop retrying and provide manual steps.
- If MCP service ports do not match, ask for the actual port and help update the configuration.

### Routing Decisions
- If routing misses, **do not force the task into an existing skill**; propose a new one.
- If one path fails, switch paths: static to dynamic, Java layer to native libraries, IDA to r2.
- For cross-module tasks, combine multiple skills according to the "path intersections" section of `routing.md`.

### Reusing Experience
- **Always check** `field-journal/_index.md` before entering routing.
- When similar experience exists, read the corresponding log first and reuse the verified approach.
- If a historical approach is unsuitable, explain why in the new log.

### Safety Boundaries
- All operations must remain within the user’s authorized scope.
- Penetration testing requires confirmation of lawful authorization (SRC/Bug Bounty/owned systems/CTF).
- Do not expand the attack surface proactively or exceed the user-specified target scope.
- Report high-severity vulnerabilities immediately and wait for instructions before continuing.
- Do not retain unsanitized sensitive information in reports or logs.

### Output Quality
- Provide reproducible commands for key operations (do not only describe steps).
- Reverse-engineering analyses must identify addresses, offsets, and function names (do not say only "some function").
- Penetration tests must include a complete PoC (curl command/script/screenshot path).
- Mark uncertain conclusions with a confidence level.

---

## Complete Behavior Chain

```
1. 识别任务属于安全/逆向类
2. 包根 = 本文件父目录
3. master-route.sh → PRIMARY（routing.json）
4. case-init.sh / scope.md — auth.status=granted 前禁止对目标 ACT
5. 打开 PRIMARY SKILL.md
6. 缺工具 → kali/scripts/bootstrap-reverse.sh
7. 不要写入客户端全局配置
```

---

## Bootstrap Commands (Kali Edition)

```bash
bash "<本包根目录>/kali/scripts/bootstrap-reverse.sh" <capability1> [capability2] ... [--start-services]
```

### Common Combinations

```bash
# 一键配齐 Kali 原生 MCP（推荐首次使用时执行）
bash kali/scripts/bootstrap-reverse.sh mcp-kali-server metasploitmcp hexstrike-ai

# 安装 2026.1 全部新工具
bash kali/scripts/bootstrap-reverse.sh adaptixc2 atomic-operator sstimap xsstrike wpprobe fluxion gef

# AD/内网渗透工具链
bash kali/scripts/bootstrap-reverse.sh coercer evil-winrm-py netexec responder bloodhound certipy

# 逆向分析工具链
bash kali/scripts/bootstrap-reverse.sh jadx frida gef ghidra-mcp

# Web 渗透工具链
bash kali/scripts/bootstrap-reverse.sh sstimap xsstrike wpprobe nuclei
```

All supported capability names:jadx、apktool、frida、idalib-mcp、jshookmcp、xquik-mcp、anything-analyzer、idapro、r2、rabin2、adb、agent-browser、ghidra-mcp、nmap、sqlmap、hashcat、hydra、gobuster、ffuf、msfconsole、nuclei、seclists、proxycat、mcp-kali-server、metasploitmcp、hexstrike-ai、pentestswarm、adaptixc2、atomic-operator、sstimap、xsstrike、wpprobe、fluxion、gef、evil-winrm-py、coercer、netexec、responder、crackmapexec、bloodhound、certipy、wfuzz、aircrack-ng

## Refresh the Tool Index

```bash
bash "<本包根目录>/kali/scripts/refresh-tool-index.sh"
```

---

## MCP Service Management

### Native Kali MCP (direct apt installation; no additional configuration)

| Service | Package | Port | Purpose | Startup |
|------|------|------|------|---------|
| mcp-kali-server | mcp-kali-server | 5000 | Official Kali MCP; direct AI access to terminal tools | `kali-server-mcp --port 5000` |
| MetasploitMCP | metasploitmcp | 8085/stdio | Metasploit Framework MCP interface | `metasploitmcp --transport stdio` |
| HexStrike AI | hexstrike-ai | — | MCP automation platform for 150+ security tools | `hexstrike-ai` |

### Third-Party MCP Services

| Service | Port | Purpose | Startup |
|------|------|------|---------|
| Pentest Swarm AI | stdio | Swarm-intelligence autonomous penetration (recon→classify→exploit→report) | `pentestswarm mcp serve` |
| idapro | 13337-13350 | IDA Pro reverse-engineering tool | `bash kali/scripts/ida-start.sh` |
| anything-analyzer | 23816 | Browser automation + HTTP capture | `cd ~/tools/anything-analyzer && pnpm dev` |
| jshookmcp | — | JS Hook/CDP/Network/AST | `npx -y @jshookmcp/jshook@0.3.4`（stdio） |
| ghidra | 8765 | Free Ghidra decompiler | Ghidra GUI 启动后自动监听 |
| burpsuite | 9876 | BurpSuite Web proxy | BurpSuite 扩展启动 |

### MCP Priority Recommendations (Kali 2026.1)

For penetration-testing scenarios, the recommended MCP priority is:

1. **pentestswarm** — 全自动群体渗透，适合大规模目标（1000+ 子域名）和 Bug Bounty 持续监控
2. **mcp-kali-server** — 最通用，可以调用 Kali 上任何终端工具
3. **metasploitmcp** — Metasploit 专用，exploit/payload/session 管理
4. **hexstrike-ai** — 自动化编排，适合多工具联动场景
5. **jshookmcp** — Web/JS 逆向专用

Set up all penetration-testing MCP tools in one step:
```bash
bash kali/scripts/bootstrap-reverse.sh mcp-kali-server metasploitmcp hexstrike-ai pentestswarm
```

---

## Error-Handling Strategy

| Scenario | What the AI should do |
|----------|------------------------|
| Bootstrap succeeds | Continue the task |
| `apt install` fails | Check the network/source, run `apt update`, then retry once |
| `pip install` fails | Try adding `--break-system-packages`, or recommend a venv |
| GitHub download fails | Check the network/proxy and provide a manual download link |
| Service ports do not match | Ask for the actual port and help update the MCP configuration |
| The same tool fails twice | Provide complete manual steps and do not retry |

---

## Native Kali Advantages

The AI should know the following in a Kali 2026.1 environment:

1. **Many tools are preinstalled** — nmap/sqlmap/hashcat/hydra/metasploit/gobuster/ffuf/radare2/binwalk/burpsuite/wireshark/nikto/impacket/netexec/responder/bloodhound and others need no installation.
2. **Native MCP support** — `mcp-kali-server`, `metasploitmcp`, and `hexstrike-ai` are in the official Kali repository and can be installed with `apt install`.
3. **New in 2026.1** — AdaptixC2 (C2 framework), Atomic-Operator (red-team testing), SSTImap (SSTI detection), XSStrike (XSS scanning), WPProbe (WP enumeration), Fluxion (WiFi social engineering), and GEF (GDB enhancement).
4. **New in 2025.4** — evil-winrm-py (WinRM remote execution), hexstrike-ai (AI security automation), and bpf-linker.
5. **Kernel 6.18** — supports current hardware and NetHunter wireless-injection patches (QCACLD-3.0).
6. **Full Wayland support** — GNOME 49 + KDE Plasma 6.5; Wayland is also supported in VMs.
7. **Rich apt sources** — commands such as `apt install ghidra`, `apt install seclists`, and `apt install coercer` complete setup in one line.
8. **Complete Python environment** — python3/pip3 are preinstalled, and frida-tools can be installed directly with pip.
9. **No permission restrictions** — root by default or passwordless sudo.
10. **Complete network toolkit** — nc/curl/wget/socat/proxychains/chisel and others are preinstalled.
11. **SecLists path** — `/usr/share/seclists/` after apt installation.
12. **Wordlists** — common dictionaries such as rockyou are under `/usr/share/wordlists/`.
13. **LLM integration** — the official Kali blog has a local LLM integration guide for Claude Desktop + Ollama + 5ire.
14. **BackTrack mode** — `kali-undercover --backtrack` switches to the classic BackTrack 5 appearance (for social-engineering scenarios).

---

## Prohibited Actions (Same as the Windows Edition)

- ❌ Do not start reverse-engineering or penetration operations without reading `routing.md`.
- ❌ Do not guess tool paths; obtain them from `tool-index`.
- ❌ Do not skip the field-journal query before starting a task.
- ❌ Do not skip the Checklist after completing a task.
- ❌ Do not retain unsanitized real-target information in reports.
- ❌ Do not expand the penetration scope without user authorization.
- ❌ Do not repeatedly retry an automatic installation that has already failed twice.
- ❌ Do not stay silent — report problems immediately.
- ❌ Do not invent tool versions or feature descriptions.

---

## Mandatory Post-Task Checklist (Cannot Be Skipped)

When the task is complete (the vulnerability is verified, reverse engineering is complete, or the flag is captured), the AI **must** perform each item:

```text
□ 1. 生成正式报告（docs-generator skill）
     - 使用对应模板（逆向报告/渗透报告/CTF writeup/签名报告）
     - 必须包含：目标概述、完整步骤、关键证据、复现命令
     - 输出到用户项目目录（不是 skill 包内）

□ 2. 生成图表（diagram-generator skill）
     - 至少 1 张流程图嵌入报告
     - 类型选择：渗透→攻击路径图 / 逆向→调用关系图 / JS→时序图 / CTF→解题流程

□ 3. 回写 field-journal（已脱敏）
     - 按 field-journal/_template.md 格式
     - 必须包含：踩坑记录、可复用模式、工具链发现、环境信息
     - 脱敏检查：无真实域名/IP/Token/用户名

□ 4. 沉淀搜索到的知识（如果本次任务中联网搜索过）
     - 将搜索到的有价值内容写入对应 skill 的 references/
     - 标注来源 URL 和日期
     - 如果发现了新工具 → 更新 bootstrap-manifest.json
     - 如果发现了新场景 → 更新 routing.md + RULES-kali.md 关键词

□ 5. 询问社区贡献
     - "是否将本次经验贡献到社区主仓库？数据已脱敏，只提交 field-journal 文件。"
     - 用户同意 → 按 CONTRIBUTE-BACK.md 流程创建 PR
     - 用户拒绝 → 跳过

□ 6. 更新系统索引
     - 更新 field-journal/_index.md（新增条目）
     - 检查是否需要更新：routing.md / bootstrap-manifest / tool-index
     - 如果发现新工具或新场景 → 执行对应更新
```

If the AI does not execute the checklist after completing a task, the user may say: "You forgot to write the report and update the experience journal"; the AI must complete it immediately.

---

## Multi-Task and Interruption Handling

- If the user changes topics during a task, first save the current progress to field-journal (mark it as "incomplete").
- When the user returns, restore the context from field-journal.
- If the user provides multiple security tasks at once, execute them one at a time by priority rather than in parallel (to avoid tool conflicts).
- Report progress periodically for long-running tasks (such as large-file IDA analysis) so the user does not think the task is stalled.

---

## Supplementing Knowledge Online (Required When Search Is Available)

When the AI has online search capability, it **must search proactively in the following situations**:

| Scenario | What to search | What to do afterward |
|----------|----------------|----------------------|
| Unknown packer/protection/obfuscation | Search for unpacking methods and tools | Write the method to the relevant skill’s references/ |
| Unknown framework/protocol | Search for reverse-engineering/penetration methods for the framework | Write to references/ or propose a new skill |
| Tool error/incompatibility | Search the error message + version compatibility | Record the pitfall in field-journal |
| New CVE/vulnerability | Search for a PoC and exploitation method | Write to pentest-tools/references/ |
| Routing miss (new scenario) | Search the field’s methodology and tools | Propose a new skill with the gathered material |
| Specific Frida script needed | Search existing scripts on GitHub/CodeShare | Write to apk-reverse/references/ or use directly |
| Specific payload needed | Search PayloadsAllTheThings/HackTricks | Write to pentest-tools/payloads/ |
| Tool version too old | Search for the latest version and breaking changes | Update the bootstrap manifest and documentation |

### Knowledge-Recording Process After Search

```text
1. 搜索获取信息
2. 验证信息可靠性（优先官方文档 > GitHub > 博客 > 论坛）
3. 提取可操作的内容（命令/脚本/配置/步骤）
4. 写入本包对应位置：
   - 通用方法论 → 对应 skill 的 references/*.md
   - 特定工具用法 → 对应 skill 的 references/ 或 SKILL.md
   - 踩坑经验 → field-journal/
   - 新工具发现 → kali/scripts/bootstrap-manifest.json + tool-discovery.sh
   - 新场景发现 → routing.md + RULES-kali.md 关键词
5. 标注来源（URL + 日期），便于后续验证时效性
6. 如果信息量足够大（新领域），提议新增独立 skill
```

### Search Quality Requirements

- **Do not search and give the user only a link** — extract the key content and write it into this package.
- **Do not blindly trust search results** — verify against official documentation and mark confidence.
- **Prefer Chinese resources** (when the user communicates in Chinese), but use English official documentation for technical details.
- **Mark timeliness** — the security field changes quickly; record the search date and mark stale content `[possibly outdated]`.

---

## Adding a Skill

When the routing matrix cannot cover the current task type, add a skill according to the `CONTRIBUTING.md` process.

Path: `<package root>/skills/CONTRIBUTING.md`

After adding it, also update: `routing.md`, `kali/scripts/bootstrap-manifest.json`, `kali/scripts/lib/tool-discovery.sh`, and `kali/scripts/refresh-tool-index.sh`.
