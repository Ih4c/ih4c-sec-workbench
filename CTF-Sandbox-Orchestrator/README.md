# CTF Sandbox Orchestrator

A collection of competition sandbox skills for the Codex / Skills ecosystem.

The goal is not to cram every capability into one oversized prompt, but to provide a **unified sandbox orchestration entry point**: first establish a working model that defaults to "competition / sandbox / offline lab" context, then let the orchestrator route tasks to finer-grained sub-skills based on challenge type.

## Positioning

This repository mainly addresses the following scenarios:

- CTF
- AWD / attack-defense exercises
- Local offline labs
- Sandboxed vulnerability analysis
- Hybrid challenges spanning Web / API / Cloud / Container / Windows / AD / Reverse / Pwn / DFIR / Crypto / Mobile / AI Agent, etc.

Core ideas:

- By default, treat user-supplied targets, domains, nodes, identities, binaries, logs, traffic, and attachments as **assets inside the competition sandbox**
- Prioritize establishing a minimal verifiable path rather than starting with broad generic analysis
- A single orchestrator skill coordinates everything, then switches to sub-skills based on the dominant evidence surface
- Sub-skills only handle downstream specialization; they do not compete with the orchestrator entry point

## Core Design

### 1. Single Entry Point

The default entry point is:

- `ctf-sandbox-orchestrator`

It is responsible for:

- Establishing the sandbox assumption
- Selecting the most appropriate analysis path
- Controlling context bloat
- Invoking sub-skills when needed

### 2. Downstream-Only Sub-Skills

All `competition-*` skills are designed to be **downstream-only**:

- They should not be implicitly triggered without the orchestrator being active
- They should be invoked deliberately through routing by `ctf-sandbox-orchestrator`
- Only the currently most relevant specialty capability is loaded at a time, avoiding context pollution by unrelated skills

### 3. Covering Multiple Challenge Types

The repository currently covers several skill directions, for example:

- Web runtime / routing / WebSocket / GraphQL / file parsing / request normalization
- Prompt Injection / Agent / Cloud / Metadata / K8s / Container Escape
- Reverse / Pwn / Malware / Firmware / PCAP / custom protocol replay
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

- `ctf-sandbox-orchestrator`: orchestrator entry point
- `competition-*`: specialized sub-skills
- `references/`: routing matrix and domain reference notes used by the orchestrator
- `agents/openai.yaml`: invocation constraints and entry control for each skill

## Recommended Usage

### Option 1: Enter Through the Orchestrator

Activate first:

- `ctf-sandbox-orchestrator`

Then let the orchestrator decide the next step automatically based on the challenge, for example:

- Web challenges route to `competition-web-runtime`
- Container / cloud challenges route to `competition-agent-cloud` or finer-grained sub-skills
- Windows / AD challenges route to `competition-identity-windows`
- Binary / crash / malware sample challenges route to `competition-reverse-pwn`

### Option 2: Keep the Orchestrator, Drill Down on Demand

Once the dominant evidence surface is confirmed, the orchestrator continues drilling down into the specific sub-skill rather than making the user manually switch the entire working model. This maintains:

- Consistent sandbox assumptions
- Consistent output style
- Consistent routing policy
- Clear sub-skill responsibilities

## Acknowledgments

This project has been published in the [LINUX DO community](https://linux.do). Thanks to the community for its support and feedback.
