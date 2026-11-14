# reverse-skill Package Security Audit (Executable Surface)

> 2026-09-03 review: The review was extended to Git objects, payload identity, symbolic links, the binary allowlist, GitHub Action pinning, and Gradle Wrapper verification. See [Repository security review — 2026-09-03](SECURITY-REVIEW-2026-09-03.md).

> Date: 2026-08-02
> Scope: Executable scripts and bootstrap manifests in `skills/**/scripts`, `skills/scripts`, `kali/scripts`, and `burp-mcp-full`
> **Excluded**: **Instructional payload documentation** such as `src-hunter` / payloader (its DROP/injection examples are methodology, not automatically executed)

## Overall Conclusion

| Category | Assessment |
|------|------|
| **Backdoors / deliberate database deletion / disk formatting** | **Not found** |
| **Piped download and execution (curl\|sh / IEX DownloadString)** | **Not found** |
| **Hardcoded cloud keys / private keys** | **Not found** (`sk-` / `BEGIN RSA` in documentation are detection examples) |
| **Residual supply-chain risk** | **Partially hardened (medium-low → low)**: pinned `@latest`; GitHub downloads support **manifest SHA256 + API digest** |

**Overall assessment: No implanted backdoors or "one-click database deletion" logic were found in the executable skill scripts at the time of the audit; potentially destructive deletions were confined to tool reinstallation temporary directories / case output directories.**

### 2026-07-18 Hardening (This Commit)

| Item | Action |
|----|------|
| jshookmcp | `@latest` → `@0.3.4` |
| pentestswarm | `@latest` / docker `:latest` → `@v0.1.0` / `:v0.1.0` |
| jadx | pin `v1.5.6` + `assetSha256` |
| apktool | pin `v3.0.2` + `assetSha256` |
| bootstrap PS/sh | Run `Assert-DownloadedFileIntegrity` / `verify_sha256` after downloading; prefer the manifest hash, then the GitHub `digest`; delete the file and abort on failure |
| Releases without pinned hashes | Can still be installed, but emit **WARN** and print the actual sha256 |

### 2026-08-02 Security Fixes

| Item | Fix |
|----|------|
| Kali quick setup | Resolve the sudo user's home with `getent`; remove `eval` |
| Frida process listing | Use a `frida-ps` argument array; remove inline Python code concatenation |
| Burp MCP token | Replace atomically using a restricted temporary file, with POSIX permissions fixed at `0600` |
| Burp MCP bridge | Parse MCP newline-delimited messages and reconnect as needed after Burp starts |
| Anything Analyzer MCP | Enable bearer auth by default during bootstrap and register credentials through an optional host adapter |
| IDA MCP startup | Terminate old processes individually to avoid multi-PID argument expansion errors |

## Scan Method

Search executable file types (`.ps1` / `.sh` / `.py` / `.js` / `.java`) for:

- `Invoke-Expression` / `IEX` / `FromBase64String` / `DownloadString`
- Piped execution via `curl|bash` / `wget|sh`
- `DROP DATABASE|TABLE`, `rm -rf /`, `Remove-Item ... C:\Windows`
- Reverse shell patterns (`/dev/tcp` abuse, `TcpClient` callbacks)
- Hidden window launches (review their purpose)

Second pass: manually review download and deletion paths in `bootstrap-reverse.ps1/.sh`, `mcp-bridge.js`, and diagram/cryptography Python scripts.

## Findings

### 1. Deletion Operations (Expected Cleanup, Not Database Deletion)

| Location | Behavior | Risk |
|------|------|------|
| `bootstrap-reverse.ps1` `Expand-ArchiveIntoDirectory` | Delete the target installation directory before reinstalling; delete `%TEMP%\reverse-bootstrap-*` | Tool installation paths only, not user production databases |
| `bootstrap-reverse.ps1` anything-analyzer | On failure, run `Remove-Item node_modules` followed by `pnpm install` | Confined to the cloned tool repository |
| `apk-reverse/scripts/decode.*` | Clean the task output directories jadx/apktool out | Confined to the task root |
| `case-init.ps1` | Clean temporary directories | Temporary |
| `bootstrap-reverse.sh` | Similar temp / installation target cleanup | Same as above |

**No executable logic was found** targeting `C:\` or system directories, or executing `DROP`/`TRUNCATE` against arbitrary database connection strings.

### 2. Network Behavior (Tool Bootstrap, Not C2)

| Location | Behavior | Notes |
|------|------|------|
| `bootstrap-reverse.ps1` | Fetch releases from `api.github.com`; download zip/jar files with `Invoke-WebRequest` | Repository names come from the **manifest allowlist** |
| `bootstrap-reverse.sh` | `curl` / `git clone` / `pipx` / `npm` | Same as above |
| `mcp-bridge.js` | HTTP to Burp via `127.0.0.1:9876` only | Local loopback |
| `ToolDiscovery.ps1` | Probe `http://host:port/mcp` | Health check |
| `kali/.../tool-discovery.sh` | `(echo >/dev/tcp/$host/$port)` | **Port probe**, not a reverse shell |

### 3. Hidden Windows

| Location | Purpose |
|------|------|
| `bootstrap-reverse.ps1` `Start-Process ... -WindowStyle Hidden` | Start `pnpm dev` in the background (anything-analyzer) |
| `ida-reverse/scripts/start.ps1` | Start IDA-related processes (must remain in the background) |

These are service startup patterns; no hidden malicious payload downloads were found.

### 4. "Dangerous Terms" in Documentation / Payloads (Not Automatically Executed)

**Markdown/JSON instructional materials** such as `pentest-tools/src-hunter` and `attack-chain` contain **red-team methodology** involving SQL injection, `DROP` examples, and log cleanup.
These **are not automatically executed by bootstrap or master-route**; execution depends on an AI or human selecting them within an **authorized scope**.

See the relevant constraints: `ops/scope-contract.md`, `ops/skill-supply-chain.md`, and `field-journal/precedent-*.md`.

### 5. Residual Supply-Chain Risk (Further Hardening Recommended, Not a Confirmed Backdoor)

| Item | Risk | Recommendation |
|----|------|------|
| `@jshookmcp/jshook@0.3.4`, `pentestswarm@v0.1.0` in `bootstrap-manifest.json` | Tag drift / supply-chain poisoning exposure | Pin versions + checksums |
| GitHub release zip **without SHA256 verification** | A replaced release may be difficult to detect promptly | Add `assetSha256` to the manifest and verify it during bootstrap |
| Default sources for `npm install -g` / `pip` | Risks inherent in dependency ecosystems | Install only manifest capabilities; use private sources / locking in production |

## Executable Script Inventory (Audit Baseline)

```
skills/scripts/*.ps1|*.sh + lib/ToolDiscovery.ps1
skills/apk-reverse/scripts/*
skills/radare2/scripts/*
skills/ida-reverse/scripts/*
skills/browser-automation/scripts/*
skills/diagram-generator/scripts/*.py
skills/case-review/scripts/*.py
kali/scripts/*
burp-mcp-full/mcp-bridge.js (+ Java 扩展源)
```

## Recommended Ongoing Checks

```powershell
# 可执行面快速体检（示例）
rg -n "Invoke-Expression|FromBase64String|DownloadString|rm -rf /|DROP DATABASE" skills/scripts skills/*/scripts kali/scripts burp-mcp-full -g "*.ps1" -g "*.sh" -g "*.py" -g "*.js"
```

Run this checklist again before merging **executable scripts** for a new skill; Markdown-only methodology changes do not require it.

## Sign-Off

- Audit performed: Local static scan of the repository + manual review of critical paths
- Result: No backdoors / no automatic database deletion; supply-chain hardening listed as a follow-up improvement
