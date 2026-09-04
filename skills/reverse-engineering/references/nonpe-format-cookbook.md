# Non-PE / Multi-Format Agent Response Cookbook U–AV + AW–DN

> Parallel to the PE anti-debug cookbook A–T (../anti-analysis.md): organized by **file type**, giving "trigger → one-line action → Evidence".  
> **Not** a second main workflow. After Triage identifies the type, jump to the corresponding skill + this table.  
> Default is **authorized isolated lab / authorized samples and devices**. Wiper ("bricking"), BYOVD, reflective injection, etc. are written up as **detection and forensics**, not unauthorized destruction/exploit tutorials.  
> Bypass or restoration failures MUST also record Evidence; silently writing "benign" is forbidden.
>
> §1–§8 / U–AV = original rules (Issue #65). §9–§23 / AW–DN = extended rules (Issue #87, deduplicated + semantic enhancements + edge-case patches).

## 0. Routing Quick Reference

| Type Clue | Primary Skill | This Table's Section |
|----------|----------|----------|
| .bat / .cmd / batch files | malware-analysis | §1, §19 |
| .ps1 / PowerShell | malware-analysis | §2, §20 |
| Office macros / VBA / XLM / .docm/.xlsm | malware-analysis | §3 (incl. DD OLE extraction, DJ XLM macros) |
| .docx/.xlsx/.pptx OOXML external links / DDE / .rtf OLE | malware-analysis | §10 (incl. DK RTF) |
| Web/front-end JS obfuscation, JSVMP | js-reverse | §4, §21 (incl. DE/DF) |
| .sys / kernel drivers | reverse-engineering/kernel-driver-reverse.md + cre | §5 |
| .dll focus | malware-analysis / re-agent-workflow | §6 (deduplicated against A–T) |
| APK / Magisk / hidden icons | apk-reverse | §7–§8, §23 |
| .pdf / PDF documents | malware-analysis | §9 |
| .wasm / WebAssembly | reverse-engineering | §11 |
| .jar/.class / Java bytecode | reverse-engineering | §12 |
| .exe(AutoIt) / .au3 | malware-analysis | §13 |
| .hta / HTML Application | malware-analysis | §14 |
| .wsf/.jse/.vbe | malware-analysis | §15 |
| .msi / Windows Installer | malware-analysis | §16 |
| .reg / registry scripts | malware-analysis | §17 |
| .vbs / VBScript | malware-analysis | §18 |
| Xposed/LSPosed modules | apk-reverse | §22 |
| ELF / Linux binaries | reverse-engineering | → elf-analysis.md, anti-analysis.md |
| Mach-O / macOS/iOS | reverse-engineering | → platforms.md |
| Python bytecode | reverse-engineering | → languages.md |

## 1. BAT/CMD (U V W)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **U** | Many single-character SET variables + %a%%b% concatenation, or ^ line-continuation splitting commands | Expand the SETs line by line; list the restored commands; batch deobfuscation tools may be used; **forbidden** to call it "no action" before restoration | E-batch-deobf | P0 |
| **V** | Text opens as mojibake, hex header FF FE (UTF-16 LE BOM) | Confirm the BOM → convert to UTF-8 and parse; or chcp 65001 + type | E-batch-encoding | P2 |
| **W** | Lots of REM/:: and redundant GOTO/labels drowning the real logic | Strip comments; comb the real GOTO paths; run in isolation and capture cmd's actual command log | E-batch-deadcode | P1 |

## 2. PowerShell (X Z)

> IDs preserve the proposer's convention: **no Y entry**.

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **X** | Multi-layer FromBase64String / Gzip / Compress / nested -replace | Decode **layer by layer**; record each layer's result separately; tools optional (PowerDecode etc.); without tools, do it by hand/script | E-ps-decode-layer-N | P0 |
| **Z** | Reversed strings, fragments + concatenation then Invoke-Expression/IEX | Restore the full string; breakpoint on IEX or use script-block logging; the plaintext command goes into Evidence | E-ps-string-restore | P1 |

## 3. VBA Macros / XLM (AA AB AC DD DJ)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **AA** | olevba/OLEDump shows only P-Code and an empty source stream (VBA Stomping) | Use P-Code decompilation tools; if incomplete, observe via Word/Excel macro debugging; clearly state the limitations | E-vba-pcode | P0 |
| **AB** | Lots of Chr() concatenation or Base64 strings; suspected shellcode/nested script | Restore strings via the Immediate window/script; classify after decoding; watch CreateObject/Shell dynamically | E-vba-str-decode | P1 |
| **AC** | Meaningless If 1=2, or InsertLines/DeleteLines self-modification | Follow the real branch statically; dynamically bp the self-modifying APIs and dump the modified macro | E-vba-selfmod | P2 |
| **DD** | olevba/oledump detects a VBA macro project (vbaProject.bin); extension .docm/.xlsm/.pptm | Inspect the OLE stream structure with oledump.py; extract the VBA source with olevba to detect suspicious APIs; check auto-executing macros such as AutoOpen/Workbook_Open | E-office-vba | P0 |
| **DJ** | .xls/.xlsm contains Excel 4.0/XLM macros (hidden in cell formulas, not the VBA stream); olevba reports XLM macro markers | Extract the XLM macro formulas with olevba --xlm; check for EXEC/CALL/REGISTER functions in hidden worksheets; restore via XLMMacroDeobfuscator dynamic emulation | E-office-xlm | P0 |

## 4. JavaScript (AD AE AF) → main path js-reverse

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **AD** | Custom bytecode array + while/switch interpreter (JSVMP) | Find the VM entry and opcode dispatch; log the trace dynamically; run AST + dynamic in parallel; see the js-reverse DeepDive | E-js-vmp | P0 |
| **AE** | while(1){switch} + large string-array indexing | Rebuild with AST/Babel; restore strings via array indices; wakaru etc. optional; **do not** paste the whole PE ollvm-deobfuscation long-form write-up | E-js-deobf | P0 |
| **AF** | debugger statements, console hijacking, performance.now deltas, DevTools detection | Disable breakpoints / freeze time sources / headless browser; patch the detection points; authorized pages | E-js-anti-debug | P1 |

## 5. SYS Kernel Drivers (AG AH AI)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **AG** | DriverEntry is very short; logic is not at the entry | Scan MajorFunction[] for non-empty slots; prioritize IRP_MJ_DEVICE_CONTROL/CREATE; the address list goes into Evidence | E-driver-irp-handlers | P0 |
| **AH** | DeviceIoControl / IOCTL dispatch present | Build the control-code → handler table; mark METHOD_* and buffer direction; user-mode communication surface | E-driver-ioctl | P0 |
| **AI** | Sample loads/drops a known-vulnerable driver or an abnormally signed driver (BYOVD pattern) | Cross-check against **public** lists such as LOLDrivers; record driver name/hash/signature; analyze the **calling intent**; **do not** expand exploit steps | E-driver-byovd | P1 |

For the full workflow see kernel-driver-reverse.md; this table only adds agent action anchors.

## 6. DLL (AJ–AQ) — deduplicated against A–T / #72

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **AJ** | DLL analysis only looked at exports/EP, ignoring TLS or DllMain | **Both TLS callbacks and DllMain must be reviewed**; dynamic breakpoint order still follows the four-level escalation (TLS→EP/DllMain→API→ExitProcess) | E-dll-tls-dllmain | P0 |
| **AK** | Exports whose names are benign-looking but malicious, wrong names, or exports inconsistent with behavior | Cross the export table against actual calls; list anomalous exports | E-exports-anomaly | P0 |
| **AL** | No exports or very few, yet still loaded | Locate via entry point, strings, xrefs, and callers; do not give up because "no exports" | E-dll-noexport | P0 |
| **AM** | A DLL missing from the static IAT, only used at runtime | **See A–T Patch R** (Delay-Load / E-delay-import); no duplicated long-form write-up here | E-delay-import | P0 pointer |
| **AN** | Need to recover exported function parameters and calling conventions | Cross-references + dynamic register/stack inspection; annotate stdcall/fastcall etc. | E-dll-export-abi | P1 |
| **AO** | Suspected DLL hijacking/sideloading | Check same-named DLLs in the app directory, search paths, KnownDLLs; legitimate program + anomalous DLL combinations | E-dll-sideload | P1 |
| **AP** | No file mapping / reflective-loading clues | Memory characteristics, loader behavior, pathless modules; forensics in an authorized environment | E-dll-reflective | P1 |
| **AQ** | Downgrading risk solely because export names "don't look malicious" | **Forbidden** to judge safety by export names alone; combine section permissions, entry point, strings, and dynamic behavior | E-dll-export-priority | P1 |

The DLL/SYS hard gate remains: E-imports + E-exports (see re-agent-workflow).

## 7. Android Wiper / Persistence (AR AS AT) → apk-reverse

> **Authorized samples, images, or test devices only.** The actions are detection and extraction of IOCs and persistence paths, not carrying out destruction.

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **AR** | Magisk modules/scripts containing **wiper-typical commands**: deleting databases, flashing, mass rm of system partitions, etc. | Table the characteristic commands + module paths; flag the high-risk destructive capability; do not execute the wiper commands | E-android-wiper-cmd | P0 |
| **AS** | Looped curl|sh / remotely pulling scripts, non-standard C2 URLs | Pull the URL; analyze whether the downloaded body contains wiper commands; record temporary paths | E-android-wiper-backdoor | P0 |
| **AT** | /data/adb/service.d, post-fs-data.d, suspicious /system/priv-app, etc. | List the persistence scripts/APKs; summarize contents into Evidence | E-android-persistence | P1 |

## 8. Android Transparent/Hidden Icons (AU AV) → apk-reverse

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **AU** | LAUNCHER icon fully transparent/empty label, Theme.NoDisplay, no LAUNCHER category, component disabled | aapt dump badging + manifest; decompile to inspect icon pixels; anomalous entries into Evidence | E-android-hidden-icon-manifest | P0 |
| **AV** | Installed but no desktop icon, with background traffic/auto-start/high-risk permissions/dynamically restored icons | pm list vs. the launcher; dumpsys package; broadcasts and device_admin; behavior into Evidence | E-android-hidden-icon-behavior | P1 |

---

> **§9–§23 below are the Issue #87 extended rules (AW–DC).**
> Sections duplicating existing files have been removed: ELF (→ elf-analysis.md), Mach-O (→ platforms.md), Python (→ languages.md).

## 9. Malicious PDF Documents (AW AX AY AZ)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **AW** | pdfid reports /JS, /JavaScript, /OpenAction, /AA, /Launch counts >0 (including obfuscated counts with hex-encoded names like /4A#61#76#61...) | Get counts with pdfid -e (compare plain vs. obfuscated counts); extract suspicious objects with pdf-parser; peepdf interactive analysis + JS emulation | E-pdf-autoaction | P0 |
| **AX** | pdfid reports /EmbeddedFile >0; object streams contain cascading FlateDecode/ASCIIHexDecode filter chains; or encoded payloads hidden in /Annot objects | Extract stream data with pdf-parser; decode multi-layer cascading filters with peepdf (including AES-encrypted streams under security handlers r5/r6); inspect Annotation objects; identify the decoded result's type with file | E-pdf-embedded | P0 |
| **AY** | Extracted PDF JS full of eval, unescape, String.fromCharCode, atob | Trace execution in peepdf's JS emulation environment; decode Base64/Hex/ROT13 layer by layer; CyberChef as aid | E-pdf-js-deobf | P1 |
| **AZ** | Anomalous PDF structure: /JBIG2Decode, manipulated XREF tables, jumping object numbers | Rename suspicious keywords with pdfid -d; check known CVE exploitation patterns; extract the exploit trigger conditions | E-pdf-exploit | P1 |

## 10. Office OOXML / DDE / RTF (BA BB DK) → complements §3 VBA

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BA** | After ZIP-extracting docx/xlsx/pptx, suspicious external relationships in word/_rels/ or xl/_rels/ (including remote template injection) | Inspect *.rels external links; check vbaData.xml; extract embedded OLE objects; check protocol-handler abuse (ms-msdt: / search-ms: / ms-officecmd:) | E-office-ooxml | P0 |
| **BB** | Document contains DDEAUTO or DDEEXEC field codes executing external commands via fields | Scan with olevba --dde; extract the DDE command arguments; check whether they point to PowerShell/external exe | E-office-dde | P0 |
| **DK** | .rtf files containing embedded OLE objects (not OOXML, not classic OLE compound documents) | Extract embedded OLE objects with rtfobj; analyze the object type with oleobj; check Equation Editor exploits (CVE-2017-11882 etc.); identify the extracted items with file | E-rtf-ole | P0 |

## 11. WebAssembly (BC BD BE)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BC** | File starts with the \x00asm magic bytes; or JS code contains WebAssembly instantiation logic | Convert to text with wasm2wat; inspect the import section to identify host-environment imports; generate pseudocode with wasm-decompile; check Emscripten glue signatures (__wasm_call_ctors) to determine whether it was compiled from JS | E-wasm-struct | P0 |
| **BD** | Many WASM functions but simple logic, function bodies split into tiny functions; or meaningless block/loop nesting | Assess the function-minimization level with diswasm; deep analysis with JEB Pro / IDA WASM plugins; dynamically trace execution logs | E-wasm-obfuscation | P1 |
| **BE** | WASM module interacts with the browser via JS import/export functions, with WebSocket, fetch, WebGL calls | Analyze the JS glue code and the WASM module together; trace data exchange in browser DevTools; extract network URLs/domains | E-wasm-c2 | P1 |

## 12. Java JAR/Class (BF BG BH BI)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BF** | Opening the JAR in JD-GUI/jadx shows meaningless short class/method names (a.a.a / _0x prefixes / numeric class names); or heavy while/switch control-flow obfuscation | Identify the obfuscator type (ProGuard / Allatori / ZKM); static deobfuscation with Java Deobfuscator; for heavy cases, dynamic debugging to trace key logic | E-java-obfuscation | P0 |
| **BG** | Lots of Class.forName(), Method.invoke(), Constructor.newInstance(); or custom ClassLoader + defineClass() loading classes in memory from byte arrays; harmless import table but malicious classes loaded dynamically at runtime | Inspect reflection call details with javap -c -v; trace the Class.forName argument strings; check the source of defineClass() byte arrays; dynamically breakpoint Method.invoke | E-java-reflection | P0 |
| **BH** | JAR contains .so (Linux/Android) or .dll (Windows); or System.loadLibrary() calls | Extract the native library files; identify the format with file; switch to the standalone ELF/PE analysis workflow | E-java-native | P1 |
| **BI** | After unpacking, the JAR/ZIP contains nested JAR/WAR/EAR; high-entropy .dat/.bin/.img files in /resources, /assets | Recursively extract all nested archives; entropy analysis to decide encryption/compression; check META-INF/MANIFEST.MF and pom.xml | E-java-nested | P1 |

## 13. AutoIt (BJ BK BL DM)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BJ** | PE strings contain AutoIt / AU3 / EA05 / EA06 signatures; or the resource section holds AutoIt script resources (note: distinguish from AutoHotKey — MITRE T1059.010 covers both) | Extract the compiled script with autoit-ripper; identify the encoding family (EA05 = AutoIt3.00 / EA06 = AutoIt3.26); after the EA06 header extract the 8-byte decryption key to decrypt the payload; restore the source | E-autoit-extract | P0 |
| **BK** | Extracted script is full of StringEncrypt/_StringEncrypt; or Execute dynamic execution + meaningless variable names | Static decompilation with myAutToExe; identify anti-debug techniques; analyze the obfuscated control flow | E-autoit-deobf | P1 |
| **BL** | Script contains RegWrite (registry persistence), FileInstall (file dropping), InetGet (network download), Run/RunWait | Flag sensitive API call sequences; analyze the InetGet URLs; trace FileInstall drop paths | E-autoit-malicious | P0 |
| **DM** | AutoIt acts as a loader performing process hollowing: CallWindowProc/EnumWindows callbacks + shellcode + injection into a legitimate process (regsvcs.exe etc.), dropping a .NET payload (DarkGate / Snake Keylogger / ArechClient2 pattern) | Check DllCall/DllCallbackRegister call chains into kernel32 injection APIs; extract the shellcode data; identify the injected target process; extract the .NET payload for standalone analysis | E-autoit-hollowing | P0 |

## 14. HTA / HTML Application (BM BN BO)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BM** | HTML contains HTA:APPLICATION tags, window.execScript, or CreateObject calls | Check HTA:APPLICATION attributes (Application, WindowState); extract the VBS/JS inside script tags | E-hta-bypass | P0 |
| **BN** | After being launched via mshta.exe, the HTA pulls and executes a remote payload via XMLHttpRequest / ActiveXObject | Extract the network request URLs; track ActiveXObject creation (ADODB.Stream etc.); restore the complete download-and-execute chain | E-hta-download-chain | P0 |
| **BO** | HTA contains only a single extremely long obfuscated string executed via eval / execScript | Extract and decode Base64/Hex-encoded payloads; recursively detect the encoding type with CyberChef; restore the payload | E-hta-oneline | P1 |

## 15. WSF / JSE / VBE (BP BQ BR BS)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BP** | .wsf contains \<job\> + \<script language="..."\> tags mixing JScript/VBScript/Python | Split code blocks by \<script language\>; analyze each under its language's rules | E-wsf-multi | P0 |
| **BQ** | .jse/.vbe begins with the #@~^ signature — Microsoft Script Encoder encoded | Decode with screnc-decoder; without a tool, execute dynamically and dump the decoded script | E-jse-decode | P0 |
| **BR** | WSF with multiple \<script\> blocks + \<package\> referencing external resources + \<component\> referencing COM components | Build a cross-block call graph; trace function calls between \<script\> blocks; restore the complete execution flow | E-wsf-call-chain | P1 |
| **BS** | WSF uses WshShell.SendKeys to bypass UAC, WshShell.Run with window hiding (0), WScript.Sleep delays to evade | Check whether user-simulation is used to bypass security prompts; record covert execution parameters | E-wsf-anti-detect | P1 |

## 16. MSI Installers (BT BU BV)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BT** | MSI contains a CustomAction table (Binary / Script / DLL custom actions) | Extract contents with msiexec /a or lessmsi; inspect the CustomAction table; extract custom-action binaries | E-msi-custom-action | P0 |
| **BU** | MSI Binary table contains VBScript/JScript custom-action scripts | Extract the script binaries from the Binary table and decode into readable scripts; analyze under VBS/JS rules | E-msi-script | P1 |
| **BV** | MSI installs silently via /quiet /passive /qn; ALLUSERS=1 privilege escalation | Record the installation command-line arguments; analyze the Property table privilege settings; flag the silent + privilege combination | E-msi-privilege | P1 |

## 17. REG Registry Scripts (BW BX BY)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BW** | .reg writes to auto-start locations such as HKCU\...\Run or HKLM\...\Run | Extract all paths; mark Run entries as persistence; record full paths and values | E-reg-persistence | P0 |
| **BX** | .reg modifies HKCR\...\shell\open\command (file associations) or HKCR\CLSID\{...}\InprocServer32 (DLL injection) | Check whether shell\open\command points to an unusual exe; check the InprocServer32 DLL path | E-reg-hijack | P0 |
| **BY** | .reg modifies HKLM\...\Policies\System (UAC level), EnableLUA, ConsentPromptBehaviorAdmin | Check the default security config before modification; analyze the impact on UAC; flag downgrade behavior | E-reg-uac-bypass | P1 |

## 18. VBScript (BZ CA CB CC DN)

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **BZ** | .vbs/.js parsed by both VBScript and JScript; conditional compilation (@_win32) or cross-language execution | Separate VBScript/JScript code blocks; analyze each syntactically; identify mixed-execution logic | E-vbs-mixed | P1 |
| **CA** | Script contains CreateObject("WScript.Shell") / CreateObject("Shell.Application") / Scripting.FileSystemObject | Flag high-risk COM object calls; track Run/Exec arguments; trace file paths created via FSO | E-vbs-com-abuse | P0 |
| **CB** | Script begins with the #@~^ signature — Microsoft Script Encoder encoded (VBS-specific) | Decode with screnc-decoder; without a tool, execute dynamically and dump the decoded script | E-vbs-encoded | P0 |
| **CC** | VBA/VBScript contains WScript.Shell.Run + cmd /c + PowerShell, followed by process injection | Trace the CreateObject COM object chain; analyze injection characteristics in the Run arguments; record the full process-creation chain | E-vbs-inject-chain | P0 |
| **DN** | VBScript/JScript implements fileless persistence via WMI ActiveScriptEventConsumer (no Startup folder / no registry Run keys) | Check WMI event subscriptions (__EventFilter + __FilterToConsumerBinding + ActiveScriptEventConsumer); extract the bound script contents; flag fileless persistence | E-vbs-wmi-persist | P0 |

## 19. Advanced BAT/CMD Obfuscation (CD–CI) → complements §1 U–W

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **CD** | setlocal enabledelayedexpansion + !var! + dynamic variable names (!var_%i%!) | Expand one by one with delayed expansion enabled; auto-expand with Batch-Dump --expand | E-bat-delayed-expand | P0 |
| **CE** | Executes via type/more/findstr reading its own or a file's :stream ADS alternate data stream | Check for : suffix references (file.bat:payload); list ADSs with dir /r; extract with type file:stream | E-bat-ads-hidden | P0 |
| **CF** | Many echo lines write .tmp/.cmd temp files line by line, then call executes them | Extract all echo redirections to restore temp file contents; monitor scripts generated in the temp directory | E-bat-temp-gen | P1 |
| **CG** | for %%i in (...) do set var=%%i accumulating variables; for /f parsing command output line by line | Expand each for loop iteration and record the assignments; reconstruct for /f results in order | E-bat-for-expand | P1 |
| **CH** | Main batch receives arguments via %1 %*, with obfuscated instructions passed in by a parent process/downloader | Inspect the calling context and record the passed arguments; decode Base64 arguments; restore the full call chain | E-bat-param-call | P1 |
| **CI** | certutil -decode / powershell -Command / echo \| findstr combined decode-and-execute | Extract and decode Base64/Hex strings; check whether the decoded result is an executable script/PE | E-bat-encoded-exec | P0 |

## 20. Advanced PowerShell Bypasses (CJ–CO, DL) → complements §2 X–Z

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **CJ** | AMSI bypasses via [Ref].Assembly.GetType('...AmsiUtils') / amsiInitFailed / GetTypes() (including hardware-breakpoint bypass: CPU debug registers, no memory writes/VirtualProtect) | Identify the bypass pattern (patch / registry / environment variable / hardware breakpoint); confirm dynamically that it takes effect; tag the bypass technique type | E-ps-amsi | P0 |
| **CK** | Manipulating the [PSConstraintLanguage] type or modifying session state via DefaultRunspace to bypass CLM | Identify the CLM bypass pattern; tag bypass-clm; analyze the execution context after bypass | E-ps-clm-bypass | P0 |
| **CL** | [ScriptBlock]::Create / $ExecutionContext.InvokeCommand constructors; or overriding ScriptBlock logging settings | Check whether the script disables logging; dynamically verify whether logging is bypassed | E-ps-sb-log-bypass | P1 |
| **CM** | Fileless execution via IEX (New-Object Net.WebClient).DownloadString(...) or [Reflection.Assembly]::Load(FromBase64...) | Extract the download URL and check domain/IP reputation; capture the memory-loaded code from PS logs; emulate in a network-isolated environment to extract the payload | E-ps-reflect-load | P0 |
| **CN** | More than three layers of nested encoding: outer Base64 → Gzip → XOR → plaintext (beyond §2 X's two-layer scope) | Decode recursively until plaintext or no further progress; record intermediate state per layer; automate with PowerDecode; each layer's result goes into Evidence | E-ps-multi-decode | P0 |
| **CO** | Set-Alias maps IEX to a single-character alias; Get-ChildItem variable: dynamically fetches variable values | Expand all alias mappings back to the original command names; restore variables via AST analysis | E-ps-alias-decode | P1 |
| **DL** | Script patches ntdll.dll EtwEventWrite (stomping) to silence telemetry; often combined with AMSI bypasses | Check for EtwEventWrite address resolution + in-memory patching (ret 0xC3); check together with the CJ AMSI bypass; flag the dual-bypass combination | E-ps-etw-bypass | P0 |

## 21. Advanced JavaScript Obfuscation (CP CQ DE DF) → complements §4 AD–AF

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **CP** | JS uses Proxy objects to intercept property access + the Reflect API to call methods dynamically, bypassing static analysis | Identify Proxy get/set/apply trap functions; trace the real targets of Reflect.get; flag dynamic interception behavior | E-js-proxy | P1 |
| **CQ** | JS contains _0x... hex string arrays + while(!![]) infinite loops + for+switch control flow (obfuscator.io signature) | Identify obfuscator.io signatures (string arrays + infinite loops); auto-deobfuscate with de4js / jsnice; the restored code goes into Evidence | E-js-obfuscator | P0 |
| **DE** | JS body is a large bytecode array + VM interpreter loop (multiple while/switch), entry pointing at eval/Function constructors; business logic fully unreadable (deepening of §4 AD) | Identify the VM entry function and trace the opcode → handler mapping; hook eval output during dynamic browser execution; AST rebuild with JSimplifier; record the opcode mapping table | E-jsvmp-deep | P0 |
| **DF** | JS uses eval to dynamically generate and immediately execute new code, document.write to rewrite the page, or Function constructors to build function bodies dynamically | Hook eval and the Function constructor to record generated code; dynamic browser execution captures self-modifying content | E-js-selfmod | P1 |

## 22. Xposed/LSPosed Module Analysis (CR–CX) → apk-reverse

> Analyzing the Xposed/LSPosed **module itself** as the reverse-engineering target (not the tool-usage scenario).

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **CR** | AndroidManifest.xml has no android:name entry Activity; meta-data declares xposedmodule=true | Check assets/xposed_init to determine the entry class; search for implementations of IXposedHookLoadPackage/ZygoteInit/CmdInit | E-xp-entry | P0 |
| **CS** | Code contains XposedHelpers.findAndHookMethod / XposedBridge.hookMethod / findClass | Extract findAndHookMethod's first argument (target class) + second argument (target method); build a manifest of target apps | E-xp-hook-targets | P0 |
| **CT** | Module uses DexClassLoader/PathClassLoader dynamic loading; or Runtime.exec / ProcessBuilder to run commands | Trace DexClassLoader constructor arguments; extract the dynamically loaded DEX for standalone analysis; check the exec command arguments | E-xp-dynamic-load | P0 |
| **CU** | Hook targets touch sensitive APIs: payments/biometrics/SMS/contacts/location/crypto keys | Classify hooked target classes/methods by sensitivity; tag payment/biometric/SMS-contacts categories; aggregate a threat level | E-xp-sensitive-hooks | P0 |
| **CV** | Code evades XposedBridge detection / cleans Zygote injection traces / has custom network communication | Check stacktrace modification / removal of XposedBridge class references; check standalone network requests (OkHttp/Socket); identify C2 targets | E-xp-anti-detection | P1 |
| **CW** | Code dynamically replaces Resources / intercepts View drawing / declares AccessibilityService | Check AssetManager replacement / Resources.updateConfiguration; check AccessibilityService configuration; identify UI hijacking | E-xp-ui-hijack | P1 |
| **CX** | AndroidManifest.xml declares the lsposed xposedscope meta-data; or code contains package whitelist checks | Parse the xposedscope target-app scope; check dynamic whitelist bypass (reflection-modifying scope); identify global-hook privilege overreach | E-xp-scope-bypass | P1 |

## 23. Deep Magisk Module Analysis (CY–DC, DG–DI) → complements §7 AR–AT

> §7 focuses on wiper/destructive behavior. This section covers non-destructive but suspicious module behaviors: install-script analysis, file dropping, Zygisk injection, anti-detection, persistence, privilege escalation, and lateral infection.

| ID | Trigger | Action (summary) | Evidence | Priority |
|----|------|--------------|----------|------|
| **DG** | Module ZIP root contains config.sh / install.sh; META-INF/com/google/android/update-binary is a non-standard installer | Extract the on_install/print_modname/set_permissions functions from config.sh/install.sh; check whether update-binary carries extra payloads; flag pm install / dd block-device / mount -o remount,rw operations | E-mg-install-script | P0 |
| **DH** | ZIP contains system/ / vendor/ / data/ directory trees; or boot-time scripts such as post-fs-data.sh / service.sh | Extract dropped file paths to see whether an APK is dropped into /system/priv-app/; review service.sh + post-fs-data.sh contents to spot boot auto-start/background keep-alive/C2 communication; flag all writes to system partitions | E-mg-file-drop | P0 |
| **CY** | Module contains a zygisk/ directory (native libraries like arm64-v8a.so); or config.sh declares IS_ZYGISK=true | Extract the zygisk/ native libraries and analyze ZygiskModule callbacks (onLoad / preAppSpecialize / postAppSpecialize); check for JNI hooks | E-mg-zygisk | P0 |
| **DI** | Module scripts write to /data/adb/service.d/ or /data/adb/post-fs-data.d/; or modify crontab/init.rc (deepening of §7 AT) | Extract the scripts written to service.d + post-fs-data.d; check for logic that infects other modules on uninstall (post-uninstall.sh / module directory monitoring); check protection mechanisms triggered by magisk --remove-modules | E-mg-persistence | P0 |
| **CZ** | Module scripts use resetprop to modify system properties / magiskhide / DenyList; or integrate Shamiko (hides Zygisk itself) / TrickyStore (tampers the certificate chain) / PlayIntegrityFork (spoofs the Play Integrity API) | Extract all resetprop calls to identify the modified properties (ro.debuggable / ro.build.tags etc.); check DenyList self-hiding; identify Shamiko/TrickyStore/PlayIntegrityFork module-level anti-detection | E-mg-anti-detect | P0 |
| **DA** | Module scripts contain setenforce 0 / mount -o rw,remount /system / chmod 777 on sensitive directories | Check SELinux operations (setenforce/chcon/restorecon); check system partition remounts + dm-verity disabling; flag the high-risk privilege escalation | E-mg-privilege | P0 |
| **DB** | Dropped APK/scripts contain curl/wget/HTTP clients; or the dropped APK requests INTERNET + sensitive permissions like READ_CONTACTS/SMS | Extract network request target URLs/IPs; analyze the dropped APK's permission declarations; identify data-exfiltration logic | E-mg-c2 | P0 |
| **DC** | Scripts walk /data/adb/modules/, modify other modules' files, or write copies of themselves into other modules | Check module.prop for injected malicious directives; check other modules' service.sh for appended malicious code; identify "parasitic" logic | E-mg-cross-infect | P0 |

---

## 24. Constraints (Global)

1. **No parallel main workflow**: phase gates still follow re-agent-workflow / each skill.  
2. **Evidence is mandatory**: including failures, partial restorations, and quality= annotations.  
3. **Deduplicated against A–T**: no PE anti-debug duplication; AM → R; AJ adds the DLL perspective without overturning the TLS escalation.  
4. **Missing tools**: record n/a + the manual equivalent; do not pretend a commercial suite was used.  
5. **Authorization**: destructive/injection/driver-vulnerability classes get defensive analysis and forensic descriptions only.  
6. **Extended-rule deduplication**: ELF → elf-analysis.md; Mach-O → platforms.md; Python → languages.md. This table does not repeat rules for those formats.

## 25. Minimal P0 Checklist (when the type matches)

```text
□ bat/cmd → U (+ V/W when needed; advanced CD–CI)
□ ps1 → X (+ Z; advanced CJ–CO + DL ETW)
□ vba/xlm → AA + DD + DJ (+ AB/AC)
□ office ooxml/rtf → BA + DK (+ BB if DDE is suspected)
□ js heavy obfuscation → AD or AE (+ AF; advanced CP/CQ/DE/DF)
□ sys → AG + AH (+ AI if BYOVD is suspected)
□ dll → AJ + AK/AL; Delay-Load via R
□ apk destructive/hidden → AR/AS or AU (+ AT/AV)
□ pdf → AW + AX (+ AY/AZ)
□ wasm → BC (+ BD/BE)
□ jar/class → BF + BG (+ BH/BI)
□ autoit → BJ + BL + DM (+ BK)
□ hta → BM + BN (+ BO)
□ wsf/jse/vbe → BP + BQ (+ BR/BS)
□ msi → BT (+ BU/BV)
□ reg → BW + BX (+ BY)
□ vbs → CA + CB + CC + DN (+ BZ)
□ xposed module → CR + CS + CT + CU (+ CV–CX)
□ magisk deep → DG + DH + CY + DI + CZ + DA (+ DB/DC)
```
