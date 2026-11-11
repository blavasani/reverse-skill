---
name: browser-extension-reverse
description: Use for authorized reverse engineering of browser extensions (Chrome/Firefox) including manifest analysis, background workers, and extension-based credential or traffic logic recovery.
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Browser Extension Reverse Engineering

## ACTION REQUIRED(execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md`
2. `NOW`: Confirm that the target is a **browser extension** (crx/xpi/extracted directory), not ordinary web JS (ordinary web scan → `js-reverse/`)
3. `NEXT`: Extract the extension and read its manifest
4. `ACT`: Permission surface → background scripts → network/storage hooks

## Applicable scenarios

- Chrome/Edge MV2/MV3 extension analysis
- Firefox extension
- Malicious-extension IOCs and extension supply-chain poisoning investigations
- Recover extension-implemented signing, encryption, or proxy logic

## Workflow

### 1. Package

```text
□ Extract crx / obtain the extension directory from the profile
□ manifest.json:permissions,host_permissions,background,content_scripts
□ Assess excessive permissions(<all_urls>,webRequest,debugger)
```

### 2. Logic

```text
□ service_worker / background entry point
□ content_script injection points and world(isolated)
□ chrome.storage / IndexedDB keys
□ As with `js-reverse`: observe network and message passing (`runtime.sendMessage`)
```

### 3. Dynamic analysis

```text
□ Load the extracted directory in developer mode
□ chrome://extensions Check for errors
□ DevTools Attach to service worker
□ When necessary, use Frida/browser CDP (`jshookmcp`)
```

## Toolchain

| Tool | Purpose |
|------|------|
| Extraction/jq | manifest |
| Chrome DevTools | worker debugging |
| js-reverse Toolchain | deep JS analysis |
| YARA | Malicious-extension rules |

## References

- `references/extension-analysis.md`
- Relevant extension-recovery entries in field-journal
- `../js-reverse/` `../malware-analysis/`

## Routing context

**Upstream**: MASTER R30
**Downstream**: heavily obfuscated JS → `js-reverse`;poisoning investigation → supply-chain / malware

## Completion self-check

- [ ] Is the permission surface and entry-point scripts listed?
- [ ] Is the key data flow recovered?
- [ ] Checklist?
