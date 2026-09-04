---
name: apk-reverse
description: 在 CLI 环境下做 Android APK 逆向时使用 (use when reversing Android APK in a CLI environment)。适用于 APK 解包、Java 反编译、smali 修改、重打包、Frida 动态 Hook，以及按需切换到 so/native 分析 (APK unpacking, Java decompilation, smali patching, repackaging, Frida dynamic hooking, and on-demand switch to so/native analysis)。优先使用本机已安装的 jadx、apktool、frida、adb、ida-reverse、radare2 (prefer the locally installed jadx, apktool, frida, adb, ida-reverse, radare2)。
---

## ACTION REQUIRED (execute immediately after reading)

> Community comparisons for endpoint extraction / Frida adaptation etc.: ../references/community-security-skills.md; dynamic analysis requires a scope-authorized device.

1. `NOW`: Read `../field-journal/precedent-reverse.md` — confirm this skill's operations are authorized routine work
2. `NOW`: Confirm whether the current task falls within this skill's scope of application
3. `NEXT`: Read `../tool-index.md`, verify tool availability and actual paths
4. `NEXT`: If a tool is missing, invoke bootstrap; never guess paths
5. `ACT`: Enter step one of the "Workflow" below and execute — do not stop at an acknowledgment state

# APK Reversing CLI Work Guidelines

## Scope of Application

Prefer this skill when the task matches any of the following scenarios:

- Analyzing the Java business logic of an APK
- Locating login, signing, risk control (风控), certificate validation, root detection logic
- Inspecting and modifying `AndroidManifest.xml`
- Inspecting and modifying smali
- Repackaging an APK
- Dynamic Java/native hooking with Frida
- Switching to native analysis when the APK contains `.so` files

## CLI Tools Verified Working on This Machine

- `jadx` `1.5.5`
- `apktool` `3.0.2`
- `frida-ps` `17.9.6`
- `adb`
- `java`

## Scenarios Where Scripts Are Preferred

The following flows are high-frequency and error-prone in their parameters — prefer the skill's bundled scripts:

- One-shot `jadx + apktool` decode to disk with summary output: `scripts/decode.ps1`
- Frida device checks, process listing, spawn/attach injection: `scripts/frida-run.ps1`
- Rebuild, align, sign, install APK: `scripts/rebuild-sign-install.ps1`
- Quick extraction of key Manifest components and permissions: `scripts/manifest-summary.ps1`

The following one-liners stay as direct calls, no wrapper needed:

- `adb devices`
- `adb logcat`
- `frida-ps -U`
- `jadx --version`
- `apktool --version`

## Bundled Scripts

### `scripts/decode.ps1`

Purpose:

- Runs `jadx` and `apktool` uniformly
- By default creates a task output directory next to the original APK
- Outputs a summary of `package`, `java_files`, `smali_dirs`, `so_files`, etc.
- Tolerates partial jadx decompilation errors as long as usable artifacts are produced

Examples:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\decode.ps1" -ApkPath "D:\DOWNLOAD\app.apk" -Clean
pwsh -File "<skill-root>\apk-reverse\scripts\decode.ps1" -ApkPath "D:\DOWNLOAD\app.apk" -Name demo -SkipJadx
```

### `scripts/frida-run.ps1`

Purpose:

- Unified entry for Frida devices, processes, and spawn/attach
- Avoids mixing up `-f`, `-n`, `-U` when handwriting parameters

Examples:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -ListDevices
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -Usb -ListProcesses
pwsh -File "<skill-root>\apk-reverse\scripts\frida-run.ps1" -Usb -Spawn -Package com.example.app -ScriptPath "D:\hooks\test.js"
```

### `scripts/rebuild-sign-install.ps1`

Purpose:

- `apktool b` to rebuild the APK
- `zipalign` to align
- `apksigner` to sign and verify
- Optional direct `adb install`

Examples:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "C:\work\apktool_out" -Clean
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "C:\work\apktool_out" -Install -Reinstall -DeviceSerial "127.0.0.1:7555"
```

Notes:

- Generates and reuses a debug keystore by default
- Outputs by default to the same directory as `ProjectDir`, so it sits alongside the original package and the decoded directory

### `scripts/manifest-summary.ps1`

Purpose:

- Extract the package name
- List permissions
- List activity/service/receiver/provider
- Mark the main launcher activity

Examples:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\manifest-summary.ps1" -ManifestPath "C:\work\apktool_out\AndroidManifest.xml"
```

If you need to analyze `.so`, `lib/arm64-v8a/*.so`, `lib/armeabi-v7a/*.so`, combine with:

- `ida-reverse`
- `radare2`

## Tool Division of Labor

### `jadx`

Used for:

- Reading Java decompilation
- Searching package names, class names, method names
- Understanding the APK from the high-level logic first

Common commands:

```bash
jadx -d jadx_out app.apk
jadx --single-class com.example.LoginActivity -d jadx_out app.apk
jadx --deobf -d jadx_out app.apk
```

### `JEB Pro` (optional commercial tool)

Used for:

- Cross-validation and deep decompilation of Android DEX / APK / ARM
- Supplementing static analysis when jadx output is incomplete or heavily obfuscated
- Second-toolchain verification of classes, methods, and call relationships on the same target

Boundaries:

- JEB Pro is commercial software; the user must obtain and install a valid license themselves. This package will not download, crack, or circumvent licensing.
- Only invoke JEB when `tool-index` confirms it is available on this machine; otherwise continue with `jadx`, `apktool`, Ghidra, IDA, or radare2.
- Third-party JEB MCP bridges are not a dependency of this package. Before installing any, review source, permissions, network behavior, and version per `../ops/skill-supply-chain.md`, and have the user explicitly confirm registration.

### `apktool`

Used for:

- Unpacking APKs
- Inspecting and modifying `AndroidManifest.xml`
- Inspecting and modifying smali
- Rebuilding APKs

Common commands:

```bash
apktool d app.apk -o apktool_out
apktool b apktool_out -o rebuilt.apk
```

### `frida`

Used for:

- Dynamically observing Java method calls
- Hooking native exported functions
- Bypassing root detection, certificate validation, debugger detection

Common commands:

```bash
frida-ps -U
frida -U -f com.example.app -l hook.js
frida-trace -U -f com.example.app -j '*!*certificate*'
```

### `adb`

Used for:

- Device connection
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

First determine the overall composition of the APK; do not rush into patching or hooking.

Recommended actions:

1. Export Java code with `jadx -d jadx_out app.apk`
2. Export smali and resources with `apktool d app.apk -o apktool_out`
3. Look first at:
   - `AndroidManifest.xml`
   - The main `package`
   - `application`, `activity`, `service`, `receiver`
   - Whether `lib/` contains `.so` files
4. Issue #65 threat-shape quick check (authorized samples/devices; see `../reverse-engineering/references/nonpe-format-cookbook.md` §7–8):
   - Transparent/hidden icon (AU): `aapt dump badging` + manifest theme/label/icon → `E-android-hidden-icon-manifest`
   - Magisk/script-wipe-machine signatures and remote curl|sh (AR/AS) → enter signatures and URL into evidence, do **not** execute destructive commands
   - Persistence paths (AT): `service.d` / `priv-app` etc. → `E-android-persistence`

### 2. Java Logic Observation

Prefer reading from `jadx_out`:

- `MainActivity`
- `Application`
- Login, network, crypto, risk-control related classes
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

### 3. Confirmation at the Smali and Resource Layer

When `jadx` output is incomplete, heavily obfuscated, or an actual patch is needed, switch to `apktool_out`:

- Inspect `smali*/`
- Inspect `res/values/strings.xml`
- Inspect `AndroidManifest.xml`

Preferred patch targets:

- `android:exported`
- Debug flags
- Root detection return values
- Login verification logic
- Certificate validation branches

### 4. Rebuild and Install

After modifications:

```bash
apktool b apktool_out -o rebuilt.apk
```

Or close the loop with the script:

```powershell
pwsh -File "<skill-root>\apk-reverse\scripts\rebuild-sign-install.ps1" -ProjectDir "apktool_out" -Install -Reinstall -DeviceSerial "127.0.0.1:7555"
```

Notes:

- This skill only guarantees the `apktool` rebuild chain
- If a formal install to a device is required later, a signing step is usually also needed
- If the task moves into signing/alignment, add `apksigner` / `zipalign`

### 5. Dynamic Hooking

When static analysis is insufficient, use Frida:

- Hook login functions
- Hook `OkHttp` / `Retrofit` / `WebView` key points
- Hook `javax.crypto`, `MessageDigest`
- Hook root detection functions
- Hook SSL pinning logic

Principles:

- Hook the Java layer first, then decide whether native hooking is needed
- Print arguments and return values first, then decide whether to actively modify return values

Suggestions:

- For simple one-off commands, use `frida-*` directly
- For injection flows that need stable reuse, prefer `scripts/frida-run.ps1`

### 6. Native `.so` Routing

If the APK contains a critical `.so`:

- Use `apktool` or `jadx` to find `lib/**/*.so`
- If only exported symbols, strings, or a quick triage is needed, `radare2` suffices
- For deeper long-term analysis, decompilation, renaming, and type recovery, use `ida-reverse`

Switch to native as soon as you see these signals:

- The Java layer is only a JNI wrapper
- The core signing logic is not in Java
- Key logic disappears after `System.loadLibrary()`
- Certificate validation / risk control lives in the `.so`

## Output Requirements

The final report must at least state:

- Entry components and key classes
- Whether key logic lives in Java, smali, or `.so`
- Confirmed sensitive points: login, signing, root, SSL, WebView, JNI
- If a patch was made, what was changed
- If a hook was made, which class/method/exported function was hooked

## Prohibited Behaviors

- Do not blindly patch smali at the very start
- Do not write hooks before reading the manifest and the main entry point
- Do not equate incomplete Java decompilation with "logic not analyzable"
- Do not keep grinding the Java layer when the `.so` clearly carries the core logic

## Quick Command Memo

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

# Spawn and inject
frida -U -f com.example.app -l hook.js
```

---

## Routing Context

**Upstream entry**: `skills/SKILL.md` (master control), `routing.md`
**Downstream exits**:
- Core logic in `.so` → `ida-reverse/` or `radare2/`
- Dynamic hook/verification needed → `reverse-engineering/tools-dynamic.md` (Frida chapter)
- Generic reversing methodology → `reverse-engineering/SKILL.md`

**Peer module**: `reverse-engineering/` (.so analysis and advanced Frida usage)

---

## On-Demand Bootstrap

This skill's entry scripts are wired into the unified bootstrap system. Missing tools do not fail hard — they trigger an automatic install attempt.

### Automation Capability Boundaries

| Tool | Auto-installable | Install method | Notes |
|------|-----------|---------|------|
| jadx | ✓ | GitHub Release ZIP | Auto-download and extract to `%USERPROFILE%\Tools\jadx\` |
| apktool | ✓ | GitHub Release JAR + wrapper | Auto-download the jar and generate a bat at `%USERPROFILE%\Tools\apktool\` |
| JEB Pro | ✗ | User installs manually and provides a valid license | Optional Android / ARM cross-validation tool; third-party MCP bridges need separate review |
| frida / frida-ps | ✓ | pip install frida-tools | Requires Python to be installed |
| adb | ✓ | winget / fallback path | Auto-installs Android Platform-Tools |
| zipalign | ✗ | Requires manual install of Android Build-Tools | `sdkmanager "build-tools;35.0.0"` |
| apksigner | ✗ | Requires manual install of Android Build-Tools | Same as above |

### Bootstrap Trigger Points

- `scripts/decode.ps1`: calls `bootstrap-reverse.ps1` automatically when jadx or apktool is missing
- `scripts/rebuild-sign-install.ps1`: calls bootstrap automatically when adb or apktool is missing
- `scripts/frida-run.ps1`: still a manual check for now (frida is usually already installed via pip)

### When Bootstrap Fails

If automatic installation fails, the script throws a clear error with manual-install links. Common causes:
- No network access (GitHub API / PyPI unreachable)
- winget unavailable (Windows version too old)
- Java not installed (apktool depends on the JDK)


## Task Completion Self-Check (MUST pass before claiming done)

- [ ] Did I execute every step of the workflow (not just read it)?
- [ ] Did I use real tool paths based on `tool-index`?
- [ ] Did I produce reproducible evidence (commands/scripts/screenshots/reports)?
- [ ] Did I complete and write back the Checklist items required by RULES?
- [ ] If hidden-icon / wipe-machine / persistence clues were hit: did I record the E-android-* Evidence per the U–AV cookbook (within the authorized scope)?
