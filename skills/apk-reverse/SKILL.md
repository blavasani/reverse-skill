---
name: apk-reverse
description: "Use for Android APK reverse engineering in a CLI environment: unpacking, Java decompilation, smali changes, repackaging, Frida hooks, and optional native .so analysis. Prefer locally installed jadx, apktool, Frida, adb, IDA, or radare2."
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

## ACTION REQUIRED (execute immediately after reading)

> Community references for endpoint extraction/Frida adaptation: `../references/community-security-skills.md`; dynamic analysis requires a scoped authorized device.

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm this skill's actions are authorized routine operations
2. `NOW`: Confirm that the current task matches this skill's scope
3. `NEXT`: Read `../tool-index.md` and verify tool availability and actual paths
4. `NEXT`: Invoke bootstrap when a tool is missing; do not guess paths
5. `ACT`: Enter and execute the first workflow step; do not stop in a confirmation state

# APK Reverse-Engineering CLI Practice

## Scope

Prefer this skill when the task involves:

- Analyzing APK Java business logic
- Locating login, signing, risk-control, certificate-validation, and root-detection logic
- Viewing and modifying `AndroidManifest.xml`
- Viewing and modifying smali
- Repackaging an APK
- Using Frida for Java/native dynamic hooks
- Switching to native analysis when the APK contains `.so` files

## CLI Tools Verified on the Current Machine

- `jadx` `1.5.5`
- `apktool` `3.0.2`
- `frida-ps` `17.9.6`
- `adb`
- `java`

## Scenarios Where Scripts Are Preferred

The following workflows are frequent and error-prone, so prefer the scripts included with this skill:

- Complete `jadx + apktool` in one pass, write outputs, and produce a summary: `scripts/decode.ps1`
- Check Frida devices, enumerate processes, and inject with spawn/attach: `scripts/frida-run.ps1`
- Rebuild, align, sign, and install an APK: `scripts/rebuild-sign-install.ps1`
- Quickly extract key Manifest components and permissions: `scripts/manifest-summary.ps1`

Keep the following one-line commands as direct invocations; do not wrap them separately:

- `adb devices`
- `adb logcat`
- `frida-ps -U`
- `jadx --version`
- `apktool --version`

## Included Scripts

### `scripts/decode.ps1`

Purpose:

- Run `jadx` and `apktool` through one workflow
- Create a task output directory beside the original APK by default
- Output summaries such as `package`, `java_files`, `smali_dirs`, and `so_files`
- Tolerate some `jadx` decompilation errors when usable artifacts are still produced

Examples:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\decode.ps1" -ApkPath "D:\DOWNLOAD\app.apk" -Clean
pwsh -File "<skill-root>\apk-reverse\scripts\decode.ps1" -ApkPath "D:\DOWNLOAD\app.apk" -Name demo -SkipJadx
```

### `scripts/frida-run.ps1`

Purpose:

- Provide unified Frida entry points for devices, processes, and spawn/attach
- Avoid confusing `-f`, `-n`, and `-U` when writing arguments manually

Examples:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -ListDevices
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -Usb -ListProcesses
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -Usb -Spawn -Package com.example.app -ScriptPath "D:\hooks\test.js"
```

### `scripts/rebuild-sign-install.ps1`

Purpose:

- Rebuild the APK with `apktool b`
- Align it with `zipalign`
- Sign and verify it with `apksigner`
- Optionally install it directly with `adb install`

Examples:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "C:\work\apktool_out" -Clean
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "C:\work\apktool_out" -Install -Reinstall -DeviceSerial "127.0.0.1:7555"
```

Notes:

- Generate and reuse a debug keystore by default
- Write output beside `ProjectDir` by default, keeping it near the original package and unpacked directory

### `scripts/manifest-summary.ps1`

Purpose:

- Extract the package name
- List permissions
- List activities, services, receivers, and providers
- Mark the main launcher activity

Example:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\manifest-summary.ps1" -ManifestPath "C:\work\apktool_out\AndroidManifest.xml"
```

To analyze `.so`, `lib/arm64-v8a/*.so`, or `lib/armeabi-v7a/*.so`, also use:

- `ida-reverse`
- `radare2`

## Tool Responsibilities

### `jadx`

Use it for:

- Reading decompiled Java
- Searching package, class, and method names
- Understanding the APK's high-level logic first

Common commands:

```bash
jadx -d jadx_out app.apk
jadx --single-class com.example.LoginActivity -d jadx_out app.apk
jadx --deobf -d jadx_out app.apk
```

### `JEB Pro` (optional commercial tool)

Use it for:

- Cross-validating and deeply decompiling Android DEX / APK / ARM
- Supplementing static analysis when JADX output is incomplete or heavily obfuscated
- Validating classes, methods, and call relationships for the same target with a second toolchain

Boundaries:

- JEB Pro is commercial software. The user must obtain and install a valid license; this package will not download, crack, or bypass licensing.
- Invoke it only after `tool-index` confirms that JEB is available locally; otherwise continue with `jadx`, `apktool`, Ghidra, IDA, or radare2.
- A third-party JEB MCP bridge is not a package dependency. Before installation, review its source, permissions, network behavior, and version according to `../ops/skill-supply-chain.md`, then obtain explicit user confirmation before registering it.

### `apktool`

Use it for:

- Unpacking APKs
- Viewing and modifying `AndroidManifest.xml`
- Viewing and modifying smali
- Rebuilding APKs

Common commands:

```bash
apktool d app.apk -o apktool_out
apktool b apktool_out -o rebuilt.apk
```

### `frida`

Use it for:

- Dynamically observing Java method calls
- Hooking native export functions
- Bypassing root detection, certificate validation, and debugger detection

Common commands:

```bash
frida-ps -U
frida -U -f com.example.app -l hook.js
frida-trace -U -f com.example.app -j '*!*certificate*'
```

### `adb`

Use it for:

- Connecting to devices
- Installing APKs
- Viewing logs
- Pulling files

Common commands:

```bash
adb devices
adb install -r app.apk
adb shell pm list packages
adb logcat
adb pull /data/local/tmp/file .
```

## Recommended Workflow

### 1. Triage

First determine the APK's general composition; do not rush to modify the package or add hooks.

Recommended actions:

1. Export Java code with `jadx -d jadx_out app.apk`
2. Export smali and resources with `apktool d app.apk -o apktool_out`
3. Check these first:
   - `AndroidManifest.xml`
   - The main `package`
   - `application`, `activity`, `service`, and `receiver`
   - Whether the `lib/` directory contains `.so` files
4. Issue #65 threat-pattern quick reference (authorized samples/devices; see `../reverse-engineering/references/nonpe-format-cookbook.md` §7–8):
   - Transparent/hidden icon (AU): `aapt dump badging` + manifest theme/label/icon → `E-android-hidden-icon-manifest`
   - Magisk/script device-reset indicators and remote curl|sh (AR/AS) → record indicators and URLs as evidence; **do not execute** destructive commands
   - Persistence paths (AT): `service.d` / `priv-app`, etc. → `E-android-persistence`

### 2. Observing Java Logic

Read these from `jadx_out` first:

- `MainActivity`
- `Application`
- Login, networking, encryption, and risk-control classes
- Third-party SDK initialization classes

Common keywords:

- `login`
- `sign`
- `encrypt`
- `cipher`
- `token`
- `root`
- `certificate`
- `trust`
- `okhttp`
- `retrofit`
- `webview`

If the Java code is readable, locate the business logic here first.

### 3. Confirming Smali and Resource Layers

When `jadx` output is incomplete, heavily obfuscated, or an actual patch is needed, switch to `apktool_out`:

- Inspect `smali*/`
- Inspect `res/values/strings.xml`
- Inspect `AndroidManifest.xml`

Prioritize patches to:

- `android:exported`
- Debug flags
- Root-detection return values
- Login validation logic
- Certificate-validation branches

### 4. Rebuilding and Installing

After making changes:

```bash
apktool b apktool_out -o rebuilt.apk
```

Or run the complete workflow directly with the script:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "apktool_out" -Install -Reinstall -DeviceSerial "127.0.0.1:7555"
```

Notes:

- This skill guarantees only the `apktool` rebuild path
- A signing workflow is usually also required for a subsequent production-style device installation
- If the task reaches signing/alignment, add `apksigner` / `zipalign`

### 5. Dynamic Hooking

Use Frida when static analysis is insufficient:

- Hook login functions
- Hook key points in `OkHttp` / `Retrofit` / `WebView`
- Hook `javax.crypto` and `MessageDigest`
- Hook root-detection functions
- Hook SSL-pinning logic

Principles:

- Hook the Java layer first, then determine whether a native hook is needed
- Print parameters and return values first, then decide whether to modify return values actively

Recommendations:

- Use `frida-*` directly for simple one-off commands
- Prefer `scripts/frida-run.ps1` for injection workflows that need reliable reuse

### 6. Native `.so` Branching

If the APK contains important `.so` files:

- Find `lib/**/*.so` with `apktool` or `jadx`
- Use `radare2` for exported symbols, strings, or quick triage
- Use `ida-reverse` for sustained deep analysis, decompilation, renaming, and type recovery

Switch to native analysis promptly when you see these signals:

- The Java layer is only a JNI wrapper
- Core signing logic is not in Java
- Key logic disappears after `System.loadLibrary()`
- Certificate validation or risk control is in `.so`

## Output Requirements

At minimum, state:

- Entry components and key classes
- Whether key logic is in Java, smali, or `.so`
- Confirmed sensitive points: login, signing, root, SSL, WebView, and JNI
- What changed if a patch was made
- Which class, method, or export function was hooked

## Prohibited Actions

- Do not blindly modify smali at the start
- Do not write hooks before inspecting the manifest and main entry point
- Do not equate incomplete Java decompilation with unanalyzable logic
- Do not keep focusing on the Java layer when `.so` files clearly carry core logic

## Quick Command Reference

```bash
# Decompile Java
jadx -d jadx_out app.apk

# Unpack APK
apktool d app.apk -o apktool_out

# Rebuild APK
apktool b apktool_out -o rebuilt.apk

# Devices and processes
adb devices
frida-ps -U

# Launch and inject
frida -U -f com.example.app -l hook.js
```

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (controller), `routing.md`
**Downstream exits**:
- Core logic in `.so` → `ida-reverse/` or `radare2/`
- Dynamic hooking/validation required → `reverse-engineering/tools-dynamic.md` (Frida section)
- General reverse-engineering methodology → `reverse-engineering/SKILL.md`

**Related sibling module**: `reverse-engineering/` (.so analysis and advanced Frida usage)

---

## On-Demand Bootstrap

The entry scripts for this skill are integrated with the unified bootstrap system. When a tool is missing, they attempt installation automatically instead of failing immediately.

### Automation Boundaries

| Tool | Auto-installable | Installation method | Notes |
|------|-----------|---------|------|
| jadx | ✓ | GitHub Release ZIP | Automatically download and extract to `%USERPROFILE%\Tools\jadx\` |
| apktool | ✓ | GitHub Release JAR + wrapper | Automatically download the jar and generate a bat file in `%USERPROFILE%\Tools\apktool\` |
| JEB Pro | ✗ | User installs it manually and provides a valid license | Optional Android / ARM cross-validation tool; audit the third-party MCP bridge separately |
| frida / frida-ps | ✓ | pip install frida-tools | Python must already be installed |
| adb | ✓ | winget / fallback path | Automatically install Android Platform-Tools |
| zipalign | ✗ | Manual Android Build-Tools installation required | `sdkmanager "build-tools;35.0.0"` |
| apksigner | ✗ | Manual Android Build-Tools installation required | Same as above |

### Bootstrap Trigger Points

- `scripts/decode.ps1`: automatically call `bootstrap-reverse.ps1` when jadx or apktool is missing
- `scripts/rebuild-sign-install.ps1`: automatically call bootstrap when adb or apktool is missing
- `scripts/frida-run.ps1`: still performs a manual check (frida is usually installed through pip)

### When Bootstrap Fails

If automatic installation fails, the script raises a clear error with a manual installation link. Common causes:
- Network unavailable (GitHub API / PyPI unreachable)
- winget unavailable (Windows version too old)
- Java not installed (apktool requires a JDK)

## Completion Self-Check (MUST pass before claiming completion)

- [ ] Did I execute every workflow step rather than only read it?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the Checklist items required by RULES?
- [ ] If hidden-icon/device-reset/persistence indicators were found, did I record E-android-* Evidence according to the U–AV cookbook within the authorized scope?
