---
name: browser-automation
description: |
  Unified automation entry point (统一自动化入口). Covers browser automation (Playwright) and Windows desktop app automation (OpenReverse).
  Browser scenarios: open web pages, click, fill forms, crawl, screenshot, automated login, pentest page interaction.
  Desktop scenarios: operate GUI tools such as IDA/x64dbg, Windows UI Automation, vision-driven interaction, desktop app network capture.
  Trigger keywords (触发关键词): 浏览器自动化、桌面自动化、打开网页、填表、爬取、截图、自动化登录、Playwright、agent-browser、headless、OpenReverse、UIA、CUA、桌面操作、Windows 自动化.
---

# Automation Operations (Desktop & Browser Automation)

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Confirm whether the current task falls within this skill's scope of application
2. `NOW`: Read `../tool-index.md`, verify tool availability and actual paths
3. `NEXT`: If a tool is missing, invoke bootstrap; never guess paths
4. `ACT`: Enter step one of the "Workflow" below and execute — do not stop at an acknowledgment state

## Scope of Application

Use this skill when the task matches one of the following scenarios:

### Browser scenarios (Playwright / agent-browser)
- Open web pages and operate page elements (click, fill forms, submit)
- Crawl page content or take screenshots
- Automated login flows
- Interacting with web pages during penetration testing (submitting payloads, triggering XSS)
- Automated handling of captcha pages
- Batch form submission

### Desktop app scenarios (OpenReverse)
- Operate Windows desktop apps (IDA Pro, x64dbg, Wireshark, etc.)
- Vision-driven interaction needed (CUA mode)
- Structured UI operations needed (UIA mode)
- Network traffic observation of desktop apps (built-in mitmproxy)
- Automated GUI operations of reverse-engineering tools
- Black-box testing of desktop software

### Division of Labor with Other Tools

| Scenario | What to use |
|------|--------|
| Operating a web page (inside a browser) | **Playwright / agent-browser** |
| Operating a desktop app (Windows GUI) | **OpenReverse** |
| Packet capture analysis, HTTP request capture | anything-analyzer or the OpenReverse network lane |
| JS breakpoints, hooking, CDP debugging | jshookmcp |
| Locating signing algorithms, environment patching for reproduction | js-reverse |

Quick rule of thumb:
- Target is a web page → Playwright
- Target is a Windows desktop app → OpenReverse
- Both needed → combine them

---

## Part 1: Browser Automation (Playwright / agent-browser)

### Core Workflow

```bash
# 1. Open the page
agent-browser open <url>

# 2. Get interactive elements (returns @e1, @e2... references)
agent-browser snapshot -i

# 3. Operate elements with the references
agent-browser click @e1
agent-browser fill @e2 "text"

# 4. Close when done
agent-browser close
```

### Command Reference

```bash
# Navigation
agent-browser open <url>
agent-browser close

# Page snapshots
agent-browser snapshot        # full accessibility tree
agent-browser snapshot -i     # interactive elements only (recommended)

# Interaction
agent-browser click @e1
agent-browser fill @e2 "text"
agent-browser type @e2 "text"
agent-browser press Enter
agent-browser scroll down 500

# Getting information
agent-browser get text @e1
agent-browser get title
agent-browser get url

# Waiting
agent-browser wait @e1
agent-browser wait 2000
agent-browser wait --load networkidle
```

### Notes
- `agent-browser close` must run, otherwise the process leaks
- Always snapshot before operating; never guess element references
- After submitting a form, use `wait --load networkidle` to let the page settle

---

## Part 2: Desktop App Automation (OpenReverse)

### Overview

[OpenReverse](https://github.com/zhexulong/openreverse) is a desktop interaction and evidence-collection framework for AI agents. It supports:
- **UIA mode**: Windows UI Automation, structured desktop control operations
- **CUA mode**: vision-driven interaction (Computer Use Agent), suited to complex GUIs
- **Network observation**: built-in mitmproxy proxy + local capture

### Choosing an Interaction Mode

| Mode | Best for | Underlying tech |
|------|---------|------|
| UIA | Target app has standard Windows controls (buttons, text boxes, lists) | Windows UI Automation API |
| CUA | Target app UI is complex or uses non-standard controls (IDA's disassembly view, custom-rendered UIs) | Visual recognition + mouse/keyboard |

### Network Observation Modes

| Mode | Best for |
|------|---------|
| Proxy Lane | Target app can be configured with a proxy (recommended) |
| Local Lane | Target app cannot use a proxy and needs local capture |

### Installation and Configuration

```bash
# 1. Clone the project
git clone https://github.com/zhexulong/openreverse.git
cd openreverse

# 2. Install dependencies
npm install

# 3. Integrate with the agent host (Claude Code / Codex / Zed)
npm run init:agents -- --target=all /path/to/project

# 4. Install the CUA runtime (if vision-driven mode is needed)
npm run install:cua-runtime
npm run doctor:cua-runtime

# 5. Install the network observation dependency (if packet capture is needed)
npm run install:mitmproxy
npm run doctor:network
```

### Common Combinations

| Requirement | Configuration |
|------|------|
| Only operate a desktop app | UIA or CUA, no network lane |
| Operate a desktop app + packet capture | UIA/CUA + proxy lane |
| Operate a desktop app + local capture | UIA/CUA + local lane |

### Reversing Scenario Examples

```text
Scenario: automated batch analysis with IDA Pro

1. Open IDA Pro with OpenReverse CUA mode
2. Automatically load the target binary
3. Wait for the analysis to finish
4. Export the function list through UI operations
5. Use the network lane to observe IDA's network behavior (e.g., Lumina requests)
```

```text
Scenario: automated debugging with x64dbg

1. Start x64dbg with OpenReverse UIA mode
2. Load the target program
3. Set breakpoints
4. Run and observe register/memory changes
5. Take screenshots to save evidence
```

---

## On-Demand Bootstrap

### Automation Capability Boundaries

| Tool | Auto-installable | Install method | Notes |
|------|-----------|---------|------|
| Playwright | ✓ | npm + npx playwright install | Browser automation engine |
| agent-browser CLI | ✓ | npm install -g agent-browser | Browser operation CLI |
| Node.js | ✓ | winget | Prerequisite dependency |
| OpenReverse | ✗ | Manual clone + npm install | Experimental stage, heavy dependency set |
| mitmproxy | ✗ | Manual install | OpenReverse network observation dependency |

### Bootstrap Triggers

- Browser operations missing Playwright → auto-bootstrap
- Desktop operations need OpenReverse → guide the user through manual installation (full steps provided)

### OpenReverse Manual Installation Guide

If the AI detects that desktop app automation is needed but OpenReverse is not installed:

```markdown
⚠️ **OpenReverse is required for desktop app automation**

**Installation steps**:
1. `git clone https://github.com/zhexulong/openreverse.git`
2. `cd openreverse && npm install`
3. `npm run init:agents -- --target=all <your project path>`
4. If vision mode is needed: `npm run install:cua-runtime`
5. If network observation is needed: `npm run install:mitmproxy`

**Verification**: `npm run doctor:cua-runtime` and `npm run doctor:network`
```

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master control), `routing.md`
**Applicable scenarios**: any task that needs automated browser or desktop app operations
**Downstream exits**:
- Captured requests need analysis → `anything-analyzer` or `js-reverse`
- JS debugging/hooking needed → `jshookmcp`
- Signing algorithm recovery needed → `js-reverse`
- The desktop app is a reverse-engineering tool → `ida-reverse/`

**Peer modules**: `js-reverse` (JS analysis may be needed after browser operations), `ida-reverse` (OpenReverse can automate the IDA GUI)


## Task Completion Self-Check (MUST pass before claiming done)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
