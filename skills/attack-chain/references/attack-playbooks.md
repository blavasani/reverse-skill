# Attack Chain Playbook Quick Reference

> Select a playbook by target type. Each playbook defines a standard path from initial access to the objective.

---

## Playbook 1: Internet-Facing Web Application → Domain Controller

```
1. 子域名枚举 + 端口扫描
2. Web 指纹识别 → 找到已知漏洞组件
3. 漏洞利用获取 Webshell / RCE
4. 内网信息收集（ipconfig/ifconfig, arp, net user）
5. 搭建隧道（frp/chisel/ssh）
6. 内网扫描（存活主机、开放端口）
7. 凭据获取（mimikatz/hashdump/配置文件）
8. 横向移动（PTH/WMI/PsExec）
9. 域信息收集（BloodHound）
10. 域提权（Kerberoasting/DCSync/约束委派）
11. 获取域控权限
```

**Key tool chain**: subfinder → httpx → nuclei → sqlmap/sstimap → frp → nmap → mimikatz → crackmapexec → bloodhound → certipy

---

## Playbook 2: Phishing → Internal-Network Penetration

```
1. 目标员工信息收集（LinkedIn/脉脉）
2. 构造钓鱼邮件（伪造发件人/合法主题）
3. 制作载荷（宏文档/LNK/ISO/HTML走私）
4. 发送钓鱼邮件
5. 等待上线（C2 beacon）
6. 本地信息收集 + 提权
7. 凭据提取
8. 横向移动
9. 持久化
10. 目标达成
```

**Key tool chain**: theHarvester → gophish → msfvenom/cobalt-strike → mimikatz → bloodhound

---

## Playbook 3: Close-Access Penetration → Internal Network

```
1. 物理踩点（WiFi 信号、门禁类型、USB 口）
2. WiFi 攻击（Fluxion 伪造热点 / WPA 破解）
   或 BadUSB 植入（Rubber Ducky 键盘注入）
   或 网络植入（Raspberry Pi / LAN Turtle）
3. 获取内网接入点
4. 内网扫描
5. 后续同 Playbook 1 的步骤 5-11
```

**Key tool chain**: fluxion/aircrack-ng → rubber-ducky → frp → nmap → crackmapexec

---

## Playbook 4: Cloud-Environment Penetration

```
1. 云资产发现（子域名 → CNAME → 云服务商）
2. 存储桶枚举（S3/OSS/Blob 公开访问）
3. SSRF → 云元数据（169.254.169.254）
4. 获取临时凭据（AK/SK/Token）
5. 云 API 枚举（IAM/EC2/Lambda/RDS）
6. 权限提升（PassRole/AssumeRole）
7. 横向移动（跨账户/跨区域）
8. 数据获取
```

**Key tool chain**: subfinder → nuclei(ssrf) → aws-cli → pacu → ScoutSuite

---

## Playbook 5: Bug Bounty / SRC Rapid Assessment

```
1. 资产收集（子域名 + 端口 + JS 文件）
2. 指纹识别 → 已知漏洞快速验证（nuclei）
3. 参数发现（arjun/paramspider）
4. 逐类测试：
   - IDOR/越权（改 ID/改角色）
   - SSRF（内网探测/云元数据）
   - SQL 注入（sqlmap）
   - XSS（xsstrike）
   - 文件上传（绕过检测）
   - 逻辑漏洞（支付/验证码/密码重置）
5. 编写 PoC + 提交报告
```

**Key tool chain**: subfinder → httpx → nuclei → arjun → sqlmap → xsstrike → burpsuite

---

## Playbook 6: AD CS Certificate Attack

```
1. 发现 AD CS 服务（certipy find）
2. 识别易受攻击的模板（ESC1-ESC8）
3. 请求恶意证书
4. 使用证书认证为目标用户
5. 获取 NTLM Hash 或 TGT
6. DCSync 导出所有凭据
```

**Key tool chain**: certipy → rubeus → mimikatz → secretsdump

---

## General Decision Matrix

| Current state | Next priority |
|---------|-------------|
| Only the target domain is known | Enumerate subdomains → scan ports → identify Web fingerprints |
| A Web vulnerability exists | Obtain a shell → collect internal-network information |
| A low-privilege shell is available | Escalate privileges → extract credentials |
| An internal-network machine is available | Set up a tunnel → scan the internal network → move laterally |
| Domain-user credentials are available | BloodHound → find attack paths |
| A domain-admin hash is available | DCSync → Golden Ticket |
| Cloud AK/SK is available | Enumerate permissions → escalate privileges → obtain data |
| A phishing session is active | Escalate local privileges → obtain credentials → move laterally |
| Close-access entry is available | Scan the internal network → same as above |
