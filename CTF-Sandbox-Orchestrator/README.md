# CTF Sandbox Orchestrator

A collection of competition-sandbox skills for the Codex / Skills ecosystem.

Its goal is not to pack every capability into one oversized prompt, but to provide a **unified sandbox orchestration entry point**. It first establishes a working model that assumes the task is in a competition, sandbox, or offline lab, then routes the task to more specialized skills based on the challenge type.

## Project Positioning

This repository is primarily intended for the following scenarios:

- CTF
- AWD / attack-defense exercises
- Local offline labs
- Sandbox-based vulnerability analysis
- Mixed challenges across Web / API / Cloud / Container / Windows / AD / Reverse / Pwn / DFIR / Crypto / Mobile / AI Agent

Core principles:

- Treat user-provided targets, domains, nodes, identities, binaries, logs, traffic, and attachments as **assets inside the competition sandbox** by default.
- Establish the smallest verifiable path first instead of generalizing the analysis from the outset.
- Let one orchestration skill coordinate the work, then switch to a sub-skill based on the dominant evidence surface.
- Sub-skills handle downstream specialization only and do not take over the orchestration entry point.

## Core Design

### 1. Single Entry Point

The default entry point is:

- `ctf-sandbox-orchestrator`

It is responsible for:

- Establishing the sandbox assumptions.
- Selecting the most appropriate analysis path.
- Controlling context growth.
- Invoking sub-skills when needed.

### 2. Downstream Sub-Skills

All `competition-*` skills are designed as **downstream-only**:

- They must not be triggered implicitly before the orchestrator is activated.
- They should be routed and invoked explicitly by `ctf-sandbox-orchestrator`.
- Only the most relevant specialized capability should be loaded at a time, avoiding unrelated context pollution.

### 3. Multi-Type Competition Challenges

The repository currently covers many skill areas, including:

- Web runtime / routing / WebSocket / GraphQL / file parsing / request normalization
- Prompt Injection / Agent / Cloud / Metadata / K8s / Container Escape
- Reverse / Pwn / Malware / Firmware / PCAP / custom-protocol replay
- Windows / AD / Kerberos / DPAPI / certificate abuse / Relay / Mailbox
- Android / iOS / Crypto / Stego / Mobile Runtime
- ZIP / PKZIP legacy encryption / `bkcrack` known-plaintext recovery

## Repository Structure

```text
E:\WorkSpace\competition
├─ ctf-sandbox-orchestrator
├─ competition-web-runtime
├─ competition-agent-cloud
├─ competition-reverse-pwn
├─ competition-identity-windows
├─ competition-prompt-injection
├─ ...
└─ LICENSE
```

Where:

- `ctf-sandbox-orchestrator`: orchestration entry point
- `competition-*`: specialized sub-skills
- `references/`: routing matrices and domain reference material used by the orchestrator
- `agents/openai.yaml`: invocation constraints and entry-point controls for each skill

## Recommended Usage

### Option 1: Start with the Orchestrator

Activate first:

- `ctf-sandbox-orchestrator`

Then let the orchestrator choose the next step based on the challenge, for example:

- Route a Web challenge to `competition-web-runtime`.
- Route a container or cloud challenge to `competition-agent-cloud` or a more granular sub-skill.
- Route a Windows or AD challenge to `competition-identity-windows`.
- Route a binary, crash, or malicious-sample challenge to `competition-reverse-pwn`.

### Option 2: Keep the Orchestrator and Drill Down as Needed

Once the dominant evidence surface is confirmed, let the orchestrator drill down into a specific sub-skill instead of having the user manually switch the entire working model. This keeps the following consistent:

- Sandbox assumptions
- Output style
- Routing strategy
- Clear sub-skill responsibilities

## Acknowledgments

This project was published in the [LINUX DO community](https://linux.do). Thanks to the community for its support and feedback.
