# Red-Team Sharp* Tool Analysis & Tool Installation Matrix & dnSpy MCP

## Red-Team Sharp* Tool Analysis

Red-team tooling is overwhelmingly written in C# (the Sharp* family); reversing it is a common scenario: understanding detection logic, changing signatures/features, and extracting embedded configuration.

### Common Sharp* tools at a glance

| Tool | Function | Reverse focus |
|------|------|-----------|
| **Rubeus** | Kerberos attacks (AS-REP roast / Kerberoast / S4U / pass-the-ticket) | Rubeus's project structure is fixed; look at the `Interop.*` P/Invoke section for the native calls |
| **SharpHound** | BloodHound data collector | LDAP query logic, the collected attribute sets |
| **SharpShell / SharpWS** | Remote execution, lateral movement | WMI / WinRM calls, command obfuscation |
| **Seatbelt** | Information gathering | the collection-item list, decision logic |
| **SharpRoast** | Kerberoasting | ticket request/parsing |
| **Inveigh / SharpSploit** | man-in-the-middle / general exploitation framework | reflective loading, API call chains |

### Generic analysis routine

```text
1. Open in dnSpyEx (usually unobfuscated; a few teams add ConfuserEx)
2. Look at Program.Main or the entry command dispatch (Rubeus is a switch(command) structure)
3. Find the implementation class/method of the target command
4. Inspect the P/Invoke section (Interop.* namespace) — native API calls live here
5. Extract embedded resources (some tools embed config/templates)
6. To change features (EDR evasion): alter command strings, API calls, string constants
```

### Rubeus structure example

Rubeus dispatches on commands, one class per subcommand. Finding the Kerberoasting logic:

```text
Entry:   Rubeus.CommandLineParser → parses args
Dispatch: switch(command) → "kerberoast" → execute Ask.TGS(...)
P/Invoke: Rubeus.Interop.Lsa* / Native.cs → native Kerberos API
Key:     LsaCallAuthenticationPackage (KERB_RETRIEVE_TKT_REQUEST)
```

Changing features (evasion): rename the command string `"kerberoast"` to a custom name, replace the `Rubeus` banner string, and alter the P/Invoke call order.

### Embedded configuration extraction

Many loaders/tools embed C2, keys, and certificates encrypted in resources or fields:

```powershell
# Look at Resources (the resource tree) in dnSpyEx
# Or via command line
powershell -c "[System.Reflection.Assembly]::LoadFile('target.exe').GetManifestResourceNames()"
# Once the resource is found: right-click it in dnSpyEx → Extract / Save
```

For configuration decrypted at runtime → dynamically break at the decryption method's return point and dump the plaintext (see `common-workflow.md`).

---

## Tool Installation Matrix

### Windows (preferred; dnSpyEx is a GUI)

```powershell
# Option A: Chocolatey
choco install dnspy ilspy de4dot detect-it-easy

# Option B: download releases manually (recommended, version-controlled)
# dnSpyEx:    https://github.com/dnSpyEx/dnSpy/releases
# de4dot:     https://github.com/de4dot/de4dot/releases
# ILSpy:      https://github.com/icsharpcode/ILSpy/releases
# DIE:        https://github.com/horsicq/Detect-It-Easy/releases
# dnlib:      dotnet add package dnlib  (NuGet)
```

### Linux / macOS (no dnSpyEx GUI; use the CLI)

```bash
# ILSpy CLI decompilation
dotnet tool install -g ilspycmd
ilspycmd target.exe -p -o outdir/         # decompile to a directory

# de4dot cross-platform (needs mono or dotnet)
# download the .dll from the de4dot releases and run it with dotnet
dotnet de4dot.dll target.exe -o target-clean.exe

# dnlib (scripting, needs the dotnet SDK)
dotnet new console -o dnclean && cd dnclean
dotnet add package dnlib

# DIE CLI (diec)
# Linux: install from https://github.com/horsicq/Detect-It-Easy
diec target.exe
```

### .NET runtime prerequisites

```bash
# Linux
sudo apt install dotnet-runtime-8.0        # or 6.0/7.0 depending on the target
# macOS
brew install --cask dotnet-sdk
```

> dnSpyEx (with the IL editor + debugger) is Windows-GUI-only. On Linux/macOS, .NET reversing is limited to `ilspycmd` decompilation + `dnlib` script patching — there is no equivalent interactive debugging GUI. Prefer Windows when patching is required.

---

## dnSpy MCP Integration

The community already has several dnSpy MCP projects that expose dnSpy's decompilation/IL inspection as MCP tools an AI can call directly — fully aligned with reverse-skill's MCP philosophy.

### Mainstream dnSpy MCP projects

| Project | Characteristics | Fit |
|------|------|------|
| **soufianetahiri/dnspy-mcp** | core MCP Server exposing decompile, IL inspection and other tools | Claude Code / Cursor |
| **AgentSmithers/DnSpy-MCPserver-Extension** | runs as a dnSpyEx extension with deep GUI integration | loaded inside dnSpyEx |
| **malwarecakefactory/dnspy-mcp-extension** | 33 tools covering the whole triage → deobfuscation flow | full-flow automation |

### Registering into the Claude MCP config

After installing the dnSpyEx extension per the project's README, register it in `~/.claude/mcp.json` (exact command/args follow the project README):

```json
{
  "mcpServers": {
    "dnspy": {
      "command": "dotnet",
      "args": ["path/to/dnspy-mcp.dll"]
    }
  }
}
```

Once registered, this skill's AI integration path: the user says "analyze this .NET" → route to `dotnet-reverse/` → prefer the `dnspy_decompile` / `dnspy_inspect_il` tool surfaces → fall back to the GUI only when needed.

> dnSpy MCP is not a built-in bootstrap capability of reverse-skill; the user must manually install the extension per the project README and register it. It could later be added to `bootstrap-manifest.json`.

---

## Community Resource Index

### Strongly recommended

- **Washi's blog** — .NET reverse expert: https://blog.washi.dev/posts/misconceptions-about-dotnet/
  - Core point: **do not over-rely on dnSpy's C# decompiler; get familiar with the IL editor** (consistent with this project's IL-first principle)
- **dnSpyEx** — the actively maintained fork of dnSpy: https://github.com/dnSpyEx/dnSpy
- **de4dot** — .NET deobfuscation: https://github.com/de4dot/de4dot
- **dnlib** — metadata programming: https://github.com/dnlib/dnlib

### Hands-on tutorials

- Medium, "De-obfuscating and reversing a .NET/C# spyware" — hands-on info-stealer deobfuscation with dnSpy + de4dot
- YouTube, "dnSpy Patch .NET EXEs & DLLs" — step-by-step patching + keygen
- The 看雪 (Kanxue) forum .NET reverse section — searching ".net 逆向" / "dnSpy" / "ConfuserEx" turns up many hands-on threads, Nuitka reverse, and evasion discussions
- Guided Hacking, "Top 5 .NET Reverse Engineering Tools" — dnSpy still ranks first
- StackExchange / Reverse Engineering — advanced topics such as `DynamicMethod` debugging

### .NET resources already in this repo (linkage)

- The `.NET Analysis` section of `reverse-engineering/tools.md` — dnSpy/ILSpy quick reference + the Codegate 2013 two-stage XOR+AES-CBC pattern
- The `.NET` section of `reverse-engineering/field-notes.md` — tool notes
- `reverse-engineering/awesome-re-resources.md` — de4dot is listed
- `field-journal/seed-014_unity-il2cpp-reverse.md` — Unity IL2CPP (native side, complementary to the .NET managed layer)

In-depth .NET reverse content converges into this module; `reverse-engineering/` keeps only quick-reference pointers.
