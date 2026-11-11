---
name: browser-automation
description: "Use the browser-automation specialist workflow after routing and authorization. Follow its ACTION REQUIRED steps and preserve evidence provenance."
---

## Agent summary

Use this specialist workflow only after the master router and case scope gate are satisfied. Prefer the tools and references named in this file. Follow ACTION REQUIRED in order, keep all activity within the approved scope, and report conclusions as Evidence -> Finding -> Path.

# Automation (Desktop & Browser Automation)

## ACTION REQUIRED(execute immediately after reading)

1. `NOW`:Confirm that the current task falls within this skill's scope
2. `NOW`:Read `../tool-index.md`,verify tool availability and actual paths
3. `NEXT`:If a tool is missing, invoke bootstrap; do not guess paths
4. `ACT`:Enter and execute the first workflow step; do not stop at confirmation

## Scope

Use this skill when the task involves any of the following:

### Browser scenarios(Playwright / agent-browser)
- Open web pages and operate page elements (click, fill forms, submit)
- Scrape page content or capture screenshots
- Automate login flows
- Interact with web pages during penetration testing (submit payloads, trigger XSS)
- Automate CAPTCHA pages
- Submit forms in batches

### Desktop application scenarios(OpenReverse)
- Operate Windows desktop applications (IDA Pro, x64dbg, Wireshark, and others)
- Use vision-driven interaction (CUA mode)
- Use structured UI operations (UIA mode)
- Observe desktop application network traffic (built-in mitmproxy)
- Automate reverse-engineering tool GUI operations
- Black-box test desktop software

### Division of responsibilities with other tools

| Scenario | Tool |
|------|--------|
| Operate web pages (in a browser) | **Playwright / agent-browser** |
| Operate desktop applications (Windows GUI) | **OpenReverse** |
| Packet capture analysis and HTTP request capture | anything-analyzer or the OpenReverse network lane |
| JS breakpoints, hooks, and CDP debugging | jshookmcp |
| Locate signature algorithms and reproduce them with a simulated environment | js-reverse |

Simple rule:
- The target is a web page → Playwright
- The target is a Windows desktop application → OpenReverse
- Both are needed → Use them together

---

## Part 1: Browser Automation (Playwright / agent-browser)

### Core workflow

```bash
# 1. Open the page
agent-browser open <url>

# 2. Get interactive elements (returns @e1, @e2... references)
agent-browser snapshot -i

# 3. Operate elements using references
agent-browser click @e1
agent-browser fill @e2 "text"

# 4. Close when finished
agent-browser close
```

### Command reference

```bash
# Navigation
agent-browser open <url>
agent-browser close

# Page snapshots
agent-browser snapshot        # Full accessibility tree
agent-browser snapshot -i     # Interactive elements only (recommended)

# Interaction
agent-browser click @e1
agent-browser fill @e2 "text"
agent-browser type @e2 "text"
agent-browser press Enter
agent-browser scroll down 500

# Get information
agent-browser get text @e1
agent-browser get title
agent-browser get url

# Waiting
agent-browser wait @e1
agent-browser wait 2000
agent-browser wait --load networkidle
```

### Notes
- must be executed `agent-browser close`; otherwise, the process leaks
- Run snapshot before operating; do not guess element references
- After submitting a form, use `wait --load networkidle` and wait for the page to stabilize

---

## Part 2: Desktop application automation(OpenReverse)

### Overview

[OpenReverse](https://github.com/zhexulong/openreverse) is a desktop interaction and evidence-collection framework for AI agents that supports:
- **UIA mode**:Windows UI Automation,structured desktop-control operation
- **CUA mode**:vision-driven interaction (Computer Use Agent), suitable for complex GUIs
- **Network observation**:built-in mitmproxy proxy plus local capture

### Interaction mode selection

| Mode | Suitable scenarios | Underlying technology |
|------|---------|------|
| UIA | The target application has standard Windows controls (buttons, text boxes, lists) | Windows UI Automation API |
| CUA | The target application has a complex UI or non-standard controls (IDA disassembly views, custom-rendered interfaces) | Vision recognition plus mouse and keyboard |

### Network observationMode

| Mode | Suitable scenarios |
|------|---------|
| Proxy Lane | The target application can use a configured proxy (recommended) |
| Local Lane | The target application cannot use a proxy and requires local capture |

### Installation and configuration

```bash
# 1. Clone the project
git clone https://github.com/zhexulong/openreverse.git
cd openreverse

# 2. Install dependencies
npm install

# 3. Integrate with the agent host
npm run init:agents -- --target=all /path/to/project

# 4. Install the CUA runtime (if vision-driven mode is needed)
npm run install:cua-runtime
npm run doctor:cua-runtime

# 5. Install network-observation dependencies(if traffic capture is needed)
npm run install:mitmproxy
npm run doctor:network
```

### Common combinations

| Requirement | Configuration |
|------|------|
| Operate only a desktop application | UIA or CUA, without a network lane |
| Operate a desktop application + capture traffic | UIA/CUA + proxy lane |
| Operate a desktop application + local capture | UIA/CUA + local lane |

### Reverse-engineering examples

```text
Scenario: automate IDA Pro for batch analysis

1. Open IDA Pro with OpenReverse CUA mode
2. Automatically load the target binary
3. Wait for analysis to complete
4. Use the UI to export the function list
5. Use the network lane to observe IDA network behavior (such as Lumina requests)
```

```text
Scenario: automate x64dbg debugging

1. Start x64dbg with OpenReverse UIA mode
2. Load the target program
3. Set breakpoints
4. Run and observe register/memory changes
5. Capture screenshots as evidence
```

---

## On-Demand Bootstrap

### Automation capability boundaries

| Tool | Can install automatically | Installation method | Description |
|------|-----------|---------|------|
| Playwright | ✓ | npm + npx playwright install | Browser automation engine |
| agent-browser CLI | ✓ | npm install -g agent-browser | browser-operation CLI |
| Node.js | ✓ | winget | Prerequisite |
| OpenReverse | ✗ | manual clone + npm install | Experimental, with heavy dependencies |
| mitmproxy | ✗ | Manual installation | OpenReverse network-observation dependency |

### Bootstrap triggers

- Browser operation requires missing Playwright → bootstrap automatically
- Desktop operation requires OpenReverse → guide the user through manual installation (provide complete steps)

### OpenReverse manual installation guide

If AI detects that desktop automation is required but OpenReverse is not installed:

```markdown
⚠️ **OpenReverse is required for desktop application automation**

**Installation steps**:
1. `git clone https://github.com/zhexulong/openreverse.git`
2. `cd openreverse && npm install`
3. `npm run init:agents -- --target=all <your project path>`
4. For vision mode:`npm run install:cua-runtime`
5. For network observation:`npm run install:mitmproxy`

**Verify**:`npm run doctor:cua-runtime` and `npm run doctor:network`
```

---

## Routing context

**Upstream entry**: `skills/SKILL.md`(master controller),`routing.md`
**Applicable scenarios**: Any task requiring automated operation of a browser or desktop application
**Downstream exits**:
- Captured requests require analysis → `anything-analyzer` or `js-reverse`
- JS debugging/hooks are required → `jshookmcp`
- Signature algorithm recovery is required → `js-reverse`
- The desktop application is a reverse-engineering tool → `ida-reverse/`

**Related modules**: `js-reverse`(May require JS analysis after browser operation),`ida-reverse`(OpenReverse can automate the IDA GUI)


## Completion self-check (MUST pass before claiming completion)

- [ ] Did I execute every workflow step rather than only read it?
- [ ] Did I use `tool-index` real tool paths?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/report)?
- [ ] Did I complete and write back the checklist items required by RULES?
