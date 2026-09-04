# reverse-skill Quick Start and Community FAQ

## Project Positioning

`reverse-skill` is a collection of reverse-engineering and security research skills, rules, and tool documentation that AI clients can read — it is not a single executable application. Before using it, make sure you have clear authorization for your analysis target, or that you are working in a legitimate CTF, educational, or test environment.

## Basic Usage

First, download the project:

    git clone https://github.com/Ih4c/ih4c-sec-workbench.git
    cd ih4c-sec-workbench

Then provide the project directory to your AI client as a workspace or documentation source. The core files include:

- `RULES.md`: general rules and security boundaries.
- `skills/MASTER-ROUTING.md`: skill routing and task dispatch.
- `skills/*/SKILL.md`: skill descriptions for each specialty domain.
- `docs/platforms/`: tool installation guides for different operating systems.

This project stays client-neutral, so the actual loading method for OpenCode, Codex, Cursor, Claude Code, and other clients follows each client's official documentation; do not assume that one client's plugin or sync format applies to other clients.

## OpenCode, Codex, and Sync Issues

If a client shows `Not Synchronizable` or cannot sync, first confirm that the workspace points to the full repository root, that the file permissions allow reading, and that the client supports that directory format. The most reliable alternative is to open the repository directly in the local workspace and explicitly reference the required rules or skill files in the conversation. If the issue still reproduces, include the client version, operating system, full error message, and minimal reproduction steps when reporting.

## AI Refusing to Handle Analysis Requests

An AI's safety policy will not necessarily allow every operation just because the prompt says "I am authorized." Only handle legally authorized targets, and avoid requesting unauthorized intrusion, credential theft, persistence, or destructive operations; you can scope requests to code understanding, sample analysis, vulnerability remediation, CTF, or defensive validation. For a specific APK, website, or account, first prepare a verifiable authorization scope and test environment.

## Python Tools and uv

For standalone command-line tools you can use:

    uv tool install PACKAGE_NAME

For project dependencies, use an isolated environment:

    uv venv
    uv pip install -r requirements.txt

Do not mechanically replace every `pip` string with `uv pip`; `python -m pip`, `pipx` bootstrap, and existing virtual environments each serve different purposes. If `uv` is not installed, use the operating system package manager or a deliberately created virtual environment — do not install security tools directly into the system-global Python.

More complete installation and archive security guidance is in the [Installation and Download Security Guide](UV-AND-DOWNLOAD-SECURITY.md).

## ZIP False Positives and Download Safety

Reverse-engineering tools may contain binaries, debuggers, packers, or test data that easily trigger heuristic detection by antivirus software. An antivirus warning means neither proven safe nor proven malicious. Do not disable your antivirus or blindly dismiss warnings.

Before opening archives, download from the expected HTTPS repository or release page, verify the checksum or release digest (if provided), inspect the archive contents, and scan with up-to-date security software. Do not execute unknown binaries, scripts, or installers inside a file just because the download succeeded.

## Accounts, Contributions, and Vague Reports

Follow the terms of service of the AI clients, GitHub, tool vendors, and target environments. The repository itself cannot guarantee that third-party platforms will not restrict accounts, nor can it decide account policy on behalf of platforms. To contribute to radare2 or other skills, first read `skills/CONTRIBUTING.md` and submit small, verifiable Pull Requests.

Only issues that include the full error message, environment information, and reproduction steps are suitable for further fixes. For reports containing only "virus", "gaha", or "test", please add the file name, download URL, scanning product, version, and reproduction method.
