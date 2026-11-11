# Cybersecurity Skills Router — Kali Linux Edition

> This directory is the optimized Kali Linux 2026.1 adaptation layer. It is specifically optimized for Kali 2026.1, released in March 2026 (kernel 6.18).
> The core knowledge base (`skills/`, `CTF-Sandbox-Orchestrator/`) is shared with the Windows edition. The Kali-specific README and Bash entry points cover the Windows core capability names and additionally provide native Kali tool/MCP capabilities.  
> The root [README-kali.md](../README-kali.md) is only a short redirect; **this file is authoritative**.

---

## AI: Execute Immediately After Reading

```text
1. 检测包根目录（含 skills/ 与 kali/ 的仓库根）
2. 读取 kali/RULES-kali.md → 全局注入与工具扫描
3. bash kali/scripts/refresh-tool-index.sh
4. 与主包共用作战链：
   - skills/MASTER-ROUTING.md（或 pwsh skills/scripts/master-route.ps1）
   - skills/scripts/case-init.ps1 → work/<case>/scope.md
   - auth.status=granted + network_profile 后才对目标 ACT
   - skills/ops/（证据链 / 角色 / 时间线 / IDENTITY）
5. 向用户报告配置结果
```

For general Agent guidance, see the repository root [README_AI.md](../README_AI.md) (read this file when Kali is detected).

---

## 0. Relationship to the Windows Edition (Capability Alignment)

```text
项目根目录/
├── skills/                    # 共享：SKILL、routing、MASTER-ROUTING、ops、scripts、field-journal
├── CTF-Sandbox-Orchestrator/  # 共享：40+ CTF 子技能
├── kali/                      # ← 你在这里
│   ├── scripts/
│   │   ├── bootstrap-reverse.sh
│   │   ├── refresh-tool-index.sh
│   │   ├── bootstrap-manifest.json
│   │   └── lib/
│   │       └── tool-discovery.sh
│   ├── RULES-kali.md
│   └── README-kali.md
├── RULES.md                   # Windows 版规则
└── Readme.md                  # Windows 版说明
```


### 0.1 Alignment Principles

The Kali-specific entry point is not a simple copy of the Windows README; it is **the same set of core capability names plus additional Kali capabilities**:

- Windows：`skills/scripts/bootstrap-reverse.ps1`
- Kali：`kali/scripts/bootstrap-reverse.sh`
- Standard Linux/macOS: `skills/scripts/bootstrap-reverse.sh`

JEB Pro is a commercial tool that users must license and install themselves. Reqable MCP uses the official pinned version of `reqable-mcp-server`, but still requires the separate Reqable desktop client.

Kali scripts should cover the core capability names in the Windows manifest, such as `jadx`, `apktool`, `frida`, `jshookmcp`, `xquik-mcp`, `anything-analyzer`, `idapro`, `r2`, `adb`, `ghidra-mcp`, `seclists`, `burpsuite-mcp`, `nmap`, and `pentestswarm`; they may additionally support native Kali tools such as `mcp-kali-server`, `metasploitmcp`, `hexstrike-ai`, `sstimap`, `xsstrike`, and `netexec`.

**Shared components** (no changes needed):
- All `SKILL.md`, `routing.md`, and `MASTER-ROUTING.md` files
- `skills/ops/` operational contract (scope / evidence chain / roles / timeline)
- All `references/` knowledge base content
- `field-journal/` self-evolution mechanism
- Everything in `CTF-Sandbox-Orchestrator/`
- `docs-generator/`、`diagram-generator/`
- `skills/scripts/case-init.ps1`, `master-route.ps1` (can be invoked with pwsh)

**Kali-specific components**:
- All scripts are Bash (`.sh`)
- Package management uses `apt`
- Paths follow Linux conventions (`/opt/`, `~/tools/`, `/usr/bin/`)
- Many tools are preinstalled on Kali, greatly simplifying bootstrap logic

---

## 1. Kali Native Advantages

The following tools are **ready to use out of the box** on Kali 2026.1 (no bootstrap required):

### Classic Preinstalled Tools

| Tool | Kali Package | Status |
|------|----------|------|
| nmap | nmap | Preinstalled |
| sqlmap | sqlmap | Preinstalled |
| hashcat | hashcat | Preinstalled |
| john | john | Preinstalled |
| hydra | hydra | Preinstalled |
| metasploit | metasploit-framework | Preinstalled |
| gobuster | gobuster | Preinstalled |
| ffuf | ffuf | Preinstalled |
| radare2 | radare2 | Preinstalled |
| binwalk | binwalk | Preinstalled |
| frida | python3-frida-tools | Preinstalled or pip |
| burpsuite | burpsuite | Preinstalled |
| wireshark | wireshark | Preinstalled |
| nikto | nikto | Preinstalled |
| wfuzz | wfuzz | Preinstalled |
| impacket | impacket-scripts | Preinstalled |
| netexec | netexec | Preinstalled |
| responder | responder | Preinstalled |
| aircrack-ng | aircrack-ng | Preinstalled |
| bloodhound | bloodhound | Available through apt |
| ghidra | ghidra | Available through apt |

### New Kali 2026.1 Tools (March 2026)

| Tool | Package | Purpose |
|------|------|------|
| AdaptixC2 | adaptixc2 | Post-exploitation and adversary-emulation framework |
| Atomic-Operator | atomic-operator | Cross-platform Atomic Red Team test execution |
| Fluxion | fluxion | WiFi security auditing and social engineering |
| GEF | gef | Modern enhanced debugging framework for GDB |
| MetasploitMCP | metasploitmcp | Metasploit MCP server interface |
| SSTImap | sstimap | Automated server-side template injection detection and exploitation |
| WPProbe | wpprobe | Rapid WordPress plugin enumeration |
| XSStrike | xsstrike | Advanced XSS scanner |

### New Kali 2025.4 Tools (December 2025)

| Tool | Package | Purpose |
|------|------|------|
| evil-winrm-py | evil-winrm-py | Python WinRM remote command execution |
| hexstrike-ai | hexstrike-ai | AI MCP security automation platform (150+ tools) |
| bpf-linker | bpf-linker | BPF static linker |

### Native Kali MCP Tools (Priority Optimization)

| Tool | Package | Purpose | Installation |
|------|------|------|------|
| mcp-kali-server | mcp-kali-server | Official Kali MCP; direct AI access to terminal tools | `apt install mcp-kali-server` |
| MetasploitMCP | metasploitmcp | Metasploit MCP interface | `apt install metasploitmcp` |
| HexStrike AI | hexstrike-ai | MCP automation for 150+ security tools | `apt install hexstrike-ai` |

> **The Kali edition has the greatest advantage over the Windows edition**: the three MCP tools can be installed directly with apt, without manual GitHub/npm/Docker configuration.

This means `bootstrap-reverse.sh` requires much less work on Kali than on Windows.

---

## 2. Quick Start

### 2.0 One-Click Initialization (Recommended for New Systems)

```bash
# 全新 Kali 2026.1 系统一键配置（需要 root）
sudo bash kali/scripts/quick-setup.sh

# 跳过系统更新（网络慢时）
sudo bash kali/scripts/quick-setup.sh --skip-update

# 最小安装（不装 AD/内网工具）
sudo bash kali/scripts/quick-setup.sh --minimal
```

This script automatically performs: system updates → installs new 2026.1 tools → configures native MCP → installs reverse-engineering tools → refreshes the index → prints a report.

### 2.1 Initial Configuration

```bash
# 1. 进入项目根目录
cd /path/to/cybersecurity-skills-router

# 2. 给脚本加执行权限
chmod +x kali/scripts/*.sh kali/scripts/lib/*.sh

# 3. 刷新工具索引（检测本机工具状态）
bash kali/scripts/refresh-tool-index.sh

# 4. 查看结果
cat skills/tool-index.md
```

### 2.2 Set Up All Native Kali MCP Tools in One Step (Strongly Recommended)

```bash
# 安装 Kali 官方 MCP 三件套
bash kali/scripts/bootstrap-reverse.sh mcp-kali-server metasploitmcp hexstrike-ai

# 安装后 MCP 配置自动写入 ~/.claude/mcp.json
# 如果用 Kiro，手动复制到 ~/.kiro/settings/mcp.json
```

### 2.3 Install the New 2026.1 Tools

```bash
# 全部新工具一键安装
bash kali/scripts/bootstrap-reverse.sh adaptixc2 atomic-operator sstimap xsstrike wpprobe fluxion gef

# AD/内网渗透套件
bash kali/scripts/bootstrap-reverse.sh coercer evil-winrm-py netexec responder bloodhound certipy
```

### 2.4 Install Missing Tools

```bash
# 安装单个工具
bash kali/scripts/bootstrap-reverse.sh jadx

# 安装多个工具
bash kali/scripts/bootstrap-reverse.sh jadx apktool frida jshookmcp

# 安装并启动服务
bash kali/scripts/bootstrap-reverse.sh idapro --start-services
```

### 2.5 Enable Automatic Routing for the AI Client

Tell your AI client to read `kali/RULES-kali.md`; it will automatically perform the global injection.

---

## 3. Path Conventions

| Purpose | Kali Path |
|------|----------|
| Tool installation directory | `~/tools/` or `/opt/` |
| jadx | `/opt/jadx/` or `~/tools/jadx/` |
| apktool | `/usr/local/bin/apktool` (apt) or `~/tools/apktool/` |
| Ghidra | `/opt/ghidra/` or `~/tools/ghidra/` |
| IDA Pro | `/opt/idapro/` (if the Linux edition is available) |
| Android SDK | `~/Android/Sdk/` |
| SecLists | `/usr/share/seclists/` (apt) or `~/tools/SecLists/` |
| Node.js | `/usr/bin/node`（apt/nvm） |
| Python | `/usr/bin/python3` (system-provided) |
| MCP configuration | `~/.claude/mcp.json` or `~/.kiro/settings/mcp.json` |

---

## 4. Summary of Differences from the Windows Edition

| Dimension | Windows Edition | Kali Edition |
|------|-----------|---------|
| Script language | PowerShell (.ps1) | Bash (.sh) |
| Package management | winget / GitHub Release ZIP | apt / pip / npm / GitHub Release tar.gz |
| Path separator | `\` | `/` |
| Environment variables | `%USERPROFILE%` | `$HOME` |
| Preinstalled tools | Almost none | Many security tools preinstalled |
| IDA startup | `start.ps1` | Start the Linux edition of IDA manually; the script only registers/checks MCP unless a local launcher has been added |
| MCP configuration path | `%USERPROFILE%\.claude\mcp.json` | `~/.claude/mcp.json` |
| Port detection | `TcpClient` | `nc -z` or `ss` |

---

## 5. Verification Checklist

```bash
# ─── 基础命令 ───
java -version
python3 --version
pip3 --version
node -v
npx -v

# ─── 逆向工具 ───
jadx --version
apktool --version
adb version
frida --version
r2 -v
gdb --version          # GEF 自动加载

# ─── 渗透工具（Kali 预装） ───
nmap --version
sqlmap --version
hashcat --version
hydra -h | head -1
msfconsole --version
gobuster version
ffuf -V
nuclei -version

# ─── Kali 2026.1 新工具 ───
sstimap -h 2>&1 | head -3
xsstrike -h 2>&1 | head -3
wpprobe --help 2>&1 | head -3
coercer -h 2>&1 | head -3
evil-winrm-py -h 2>&1 | head -3

# ─── AD/内网工具 ───
netexec --help 2>&1 | head -3
responder -h 2>&1 | head -3
certipy --version 2>&1 | head -1

# ─── Kali 原生 MCP ───
which kali-server-mcp && echo "mcp-kali-server OK"
which metasploitmcp && echo "metasploitmcp OK"
which hexstrike-ai && echo "hexstrike-ai OK"

# ─── 刷新工具索引 ───
bash kali/scripts/refresh-tool-index.sh

# ─── 检查 MCP 服务（如果已配置） ───
nc -z 127.0.0.1 5000 && echo "mcp-kali-server OK" || echo "mcp-kali-server offline"
nc -z 127.0.0.1 8085 && echo "metasploitmcp OK" || echo "metasploitmcp offline"
nc -z 127.0.0.1 13337 && echo "IDA MCP OK" || echo "IDA MCP offline"
nc -z 127.0.0.1 23816 && echo "anything-analyzer OK" || echo "anything-analyzer offline"
```

---

## 6. Frequently Asked Questions

### Q: What if the radare2 version included with Kali is too old?

```bash
# 用官方源安装最新版
bash kali/scripts/bootstrap-reverse.sh r2
# Kali 版默认优先 apt 安装/补齐 radare2；如需最新版可按平台文档改用 GitHub/source
```

### Q: Can I use this with Parrot OS / BlackArch?

Yes. The script checks whether commands exist and is not tied to a specific distribution. However, `apt`-based automatic installation may need to be changed to `pacman` (BlackArch).

### Q: How do I configure the Linux edition of IDA Pro?

Install IDA in `/opt/idapro/`, then edit the `startScript` path for `idapro` in `kali/scripts/bootstrap-manifest.json`.

### Q: I want to use this system on both Windows and Kali

No problem. The `skills/` directory is synchronized through Git, and the `field-journal` experience is shared between both sides. When running scripts, Windows uses `skills/scripts/*.ps1`, while Kali uses `kali/scripts/*.sh`.

