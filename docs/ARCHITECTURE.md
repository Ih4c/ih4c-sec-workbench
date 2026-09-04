# System Architecture

## Complete Behavior Chain Flowchart

```mermaid
flowchart TD
    Start([User raises a security / reverse task]) --> Detect{Trigger keyword matched?}
    Detect -->|Yes| ReadRouting[Read SKILL.md + routing.md]
    Detect -->|No| Normal([Normal conversation])
    
    ReadRouting --> RouteMatch{Routing matrix matched?}
    RouteMatch -->|Miss| ProposeNew[Propose a new skill<br/>per CONTRIBUTING.md]
    RouteMatch -->|Hit| CheckJournal[Check field-journal<br/>for similar experience]
    
    CheckJournal --> CheckTools[Read tool-index.md<br/>confirm tool status]
    CheckTools --> ToolOK{Tools available?}
    
    ToolOK -->|Missing| Bootstrap[Call bootstrap-reverse.ps1<br/>auto-install]
    ToolOK -->|Available| Execute[Enter the skill workflow]
    
    Bootstrap --> BootOK{Install succeeded?}
    BootOK -->|Success| Execute
    BootOK -->|Failed| Guide[Output structured guidance<br/>wait for manual handling]
    Guide --> UserConfirm([User confirms installation])
    UserConfirm --> Execute
    
    Execute --> TaskDone{Task complete?}
    TaskDone -->|No| Execute
    TaskDone -->|Yes| ReviewCase[Call case-review<br/>verify the evidence graph]
    ReviewCase --> GenReport[Call docs-generator<br/>generate report + diagrams]
    
    GenReport --> WriteJournal[Write back to field-journal<br/>experience persistence]
    WriteJournal --> UpdateIndex[Update index / routing / manifest]
    UpdateIndex --> Output([Output final results])
```

## Skill Module Relationship Diagram

```mermaid
flowchart LR
    subgraph Routing layer
        SKILL[SKILL.md<br/>orchestrator entry]
        Routing[routing.md<br/>routing matrix]
    end

    subgraph Reverse analysis
        APK[apk-reverse<br/>APK reversing]
        IDA[ida-reverse<br/>IDA Pro]
        R2[radare2<br/>CLI analysis]
        RE[reverse-engineering<br/>general methodology]
        BinDiff[binary-diff<br/>symbol migration]
        PatchDiff[patch-diff-exploit<br/>N-day weaponization]
    end

    subgraph Exploitation
        Pwn[pwn-chain<br/>RE→exploit]
        Firmware[firmware-pentest<br/>firmware full chain]
    end

    subgraph Penetration testing
        Pentest[pentest-tools<br/>toolchain + loop framework]
        SrcHunter[src-hunter<br/>19 playbooks]
        EDR[edr-bypass-re<br/>EDR bypass]
    end

    subgraph Web/browser
        JS[js-reverse<br/>JS signature reversing]
        Browser[browser-automation<br/>Playwright+OpenReverse]
    end

    subgraph Infrastructure
        Bootstrap[bootstrap-reverse.ps1<br/>on-demand bootstrapping]
        Discovery[ToolDiscovery.ps1<br/>tool discovery]
        ToolIndex[tool-index<br/>status index]
    end

    subgraph Output layer
        Docs[docs-generator<br/>report generation]
        Diagram[diagram-generator<br/>diagram generation]
        Review[case-review<br/>Evidence graph audit]
        Journal[field-journal<br/>auto-evolution]
    end

    subgraph External
        CTF[CTF-Sandbox-Orchestrator<br/>40+ sub-skills]
    end

    SKILL --> Routing
    Routing --> APK & IDA & R2 & RE & BinDiff & PatchDiff
    Routing --> Pentest & JS & Browser & Pwn & Firmware & EDR
    Routing --> CTF

    Pentest --> SrcHunter
    APK -->|.so dispatch| IDA
    APK -->|.so dispatch| R2
    PatchDiff -->|write PoC| Pwn
    Firmware -->|find crash| Pwn
    Pwn -->|integrate| Pentest
    EDR -->|delivery phase| Pentest
    JS -->|browser operations| Browser
    
    Bootstrap --> Discovery --> ToolIndex
    
    APK & IDA & R2 & Pentest & JS -->|task complete| Review
    Review --> Docs
    Docs --> Diagram
    Docs --> Journal
```

## Bootstrap Bootstrapping Flow

```mermaid
flowchart TD
    Need[Missing tool detected] --> ReadManifest[Read bootstrap-manifest.json]
    ReadManifest --> Kind{Install type?}
    
    Kind -->|github-release-zip| GH[Download ZIP from GitHub Release<br/>and extract]
    Kind -->|pip-package| Pip[pip install]
    Kind -->|npm-mcp| NPM[npx launch + register MCP]
    Kind -->|npm-global| Global[npm install -g<br/>+ postInstall]
    Kind -->|winget-package| Winget[winget install]
    Kind -->|local-http-mcp| HTTP[Register URL + start service]
    
    GH & Pip & NPM & Global & Winget & HTTP --> Verify{Verified usable?}
    Verify -->|Success| AddPath[Add to PATH<br/>refresh tool-index]
    Verify -->|Failed| Manual[Output manual install guidance]
    
    AddPath --> Continue([Continue the task])
    Manual --> Wait([Wait for user confirmation])
```

## Penetration Testing Loop

```mermaid
flowchart TD
    Init[Initialize: define target/scope/tools] --> Loop

    subgraph Loop[Core loop]
        Align[1. Re-align on target] --> Review[2. Review known findings]
        Review --> Decide[3. Decide next action]
        Decide --> Risk{4. Risk gate}
        Risk -->|low/medium/high| Exec[5. Execute action]
        Risk -->|critical| Ask[Request user approval]
        Ask -->|Approved| Exec
        Exec --> Record[6. Record results]
        Record --> Check{7. Self-check}
        Check -->|Continue| Align
        Check -->|Done| Done
    end

    Done[8. Completion check] --> Report([Generate final report])
```

## Auto-Evolution Mechanism

```mermaid
flowchart LR
    Task([Task completed]) --> WriteLog[Write to field-journal<br/>pitfalls + solutions + code]
    WriteLog --> UpdateIdx[Update _index.md<br/>categorized by scenario]
    UpdateIdx --> CheckUpdate{System update needed?}
    
    CheckUpdate -->|Routing gap| FixRoute[Update routing.md]
    CheckUpdate -->|Tool changes| FixTool[Refresh tool-index]
    CheckUpdate -->|New tools| FixManifest[Update bootstrap-manifest]
    CheckUpdate -->|No update needed| Done([Done])
    
    FixRoute & FixTool & FixManifest --> Done

    NewTask([Next similar task]) --> ReadIdx[Read _index.md]
    ReadIdx --> Reuse[Reuse existing experience<br/>avoid repeating pitfalls]
```

## Multi-Platform Support Architecture

```mermaid
flowchart TD
    subgraph Shared["Shared layer (platform-agnostic)"]
        Skills[skills/<br/>SKILL.md + routing.md + references]
        CTF[CTF-Sandbox-Orchestrator/<br/>40+ sub-skills]
        Journal[field-journal/<br/>experience persistence]
        Docs[docs-generator + diagram-generator]
    end

    subgraph Windows["Windows platform layer"]
        WinScripts[skills/scripts/*.ps1<br/>PowerShell scripts]
        WinManifest[bootstrap-manifest.json<br/>winget + GitHub ZIP]
        WinRules[RULES.md<br/>Windows edition rules]
    end

    subgraph Kali["Kali Linux platform layer"]
        KaliScripts[kali/scripts/*.sh<br/>Bash scripts]
        KaliManifest[kali/scripts/bootstrap-manifest.json<br/>apt + pip + GitHub tar]
        KaliRules[kali/RULES-kali.md<br/>Kali edition rules]
    end

    Skills --> WinScripts & KaliScripts
    CTF --> WinScripts & KaliScripts
    Journal --> WinScripts & KaliScripts

    WinScripts --> WinManifest
    KaliScripts --> KaliManifest

    WinRules --> Skills
    KaliRules --> Skills
```

### Platform Selection Logic

| Environment | Rules file used | Scripts used | Package management |
|------|--------------|-----------|--------|
| Windows | `RULES.md` | `skills/scripts/*.ps1` | winget / GitHub Release ZIP |
| Kali Linux | `kali/RULES-kali.md` | `kali/scripts/*.sh` | apt / pip / npm / GitHub tar.gz |

### Kali Edition Highlights

- **Many tools preinstalled**: nmap, sqlmap, hashcat, hydra, metasploit, radare2, binwalk, burpsuite, etc. need no bootstrap
- **Unified apt management**: no winget, no manual ZIP extraction
- **Native bash**: scripts are simpler, no PowerShell dependency
- **Clean path conventions**: `/usr/bin/`, `/opt/`, `~/tools/`, no drive letters or space issues

## File Reading Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant AI as AI client
    participant R as RULES.md / RULES-kali.md
    participant SK as SKILL.md
    participant RT as routing.md
    participant TI as tool-index.md
    participant FJ as field-journal
    participant SUB as Sub-skill
    participant BS as bootstrap
    participant DOC as docs-generator

    U->>AI: Raises a security task
    AI->>R: Read routing rules
    AI->>SK: Read orchestrator entry
    AI->>RT: Routing match
    AI->>FJ: Look up similar experience
    AI->>TI: Confirm tool status
    alt Tool missing
        AI->>BS: Auto-install (.ps1 or .sh)
        BS-->>AI: Result
    end
    AI->>SUB: Enter workflow
    AI-->>U: Task results
    AI->>DOC: Generate report
    AI->>FJ: Write back experience
    AI-->>U: Done
```
