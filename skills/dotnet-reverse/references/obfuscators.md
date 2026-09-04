# .NET Obfuscator Deobfuscation in Detail

Identification, unpacking, and anti-tamper bypass for mainstream .NET obfuscators. Core tools: **de4dot** (auto-detects most packers) + **dnSpyEx** (manual patching) + **dnlib** (scripting).

## Master decision table

| Obfuscator | de4dot type | Typical features | Auto-unpack | Manual highlights |
|--------|-------------|---------|---------|---------|
| ConfuserEx 1.x/2.x | `cfze` | anti-tamper, control-flow obfuscation, string encryption, anti-debug | ✅ mostly automatic | newer builds need anti-tamper patched first |
| ConfuserEx 3.x / custom forks | `cfze` | same as above + custom protectors | ⚠️ partial | runtime dump / dnlib |
| SmartAssembly | `sa` | string encoding, resource compression, method-call hiding | ✅ automatic | resource decompression |
| Babel.NET | `babel` | method body encryption, control flow, strings | ✅ automatic | — |
| Eazfuscator.NET | `eaz` | string/resource encryption, expression obfuscation | ⚠️ partial | string decryptor |
| .NET Reactor | `reactor` | necrobit (code-section encryption) + anti-tamper | ⚠️ newer versions are hard | dump + rebuild metadata |
| Themida .NET | — | shell + virtualization | ❌ de4dot cannot | dump memory, follow native approaches |
| Agile.NET / CliSecure | `agile` | method body encryption | ✅ automatic | — |

## Standard de4dot usage

```powershell
# Auto-detect (enough for most cases)
de4dot target.exe -o target-clean.exe

# Explicitly specify the type (when auto-detection fails)
de4dot --type cfze target.exe -o target-clean.exe

# Probe the packer type first
de4dot --detect target.exe

# Batch
de4dot *.exe

# Decrypt strings only, leave control flow alone (minimal intervention)
de4dot --strtyp delegate --strtok METHOD_TOKEN target.exe
```

de4dot's `--strtyp` / `strtok` mode decrypts only the string decryptor (given the decryption method token) and keeps the original control flow. Suitable for "just want to read the plaintext strings without touching anti-tamper" scenarios.

---

## ConfuserEx (most common)

### Feature identification

- The entry module's `<module>` class carries an anti-tamper check marked `[MethodImpl(NoInlining)]`
- Numerous `Dictionary<string, T>` string-decryptor call sites
- Control-flow flattening (switch dispatch + a state variable)
- A `.cmp` compressed resource embedded in the resources
- dnSpyEx C# view: garbled class/method names (`\uXXXX` or meaningless characters), method bodies full of `int num = ...; switch(num)`

### Unpacking flow

```powershell
# 1. Standard unpacking
de4dot target.exe -o target-clean.exe

# 2. If de4dot reports "unknown" or the output does not run → new/custom ConfuserEx build
#    first confirm anti-tamper:
open in dnSpyEx → find the integrity check in Module .cctor or Main
```

### Anti-tamper bypass (common in newer ConfuserEx builds)

ConfuserEx's `anti tamper` verifies method-body hashes at runtime; the program crashes if they are modified. de4dot usually handles old versions; newer ones need manual work:

```text
Method A — patch the check function directly in dnSpyEx:
  1. Find the anti-tamper check method (usually called from the <module> static constructor)
  2. IL edit: change the check method body to ret (return immediately)
  3. Save → feed it to de4dot again

Method B — runtime dump:
  1. Run the program and dump the in-memory assembly with MegaDumper / ExtremeDumper
  2. The dump is already decrypted; clean up the leftovers with de4dot
```

### After control-flow restoration

de4dot restores flattened switch dispatch to normal if/while. If restoration is incomplete (residual state-machine code visible), run de4dot once more or trace the IL manually.

---

## SmartAssembly

```powershell
de4dot --type sa target.exe -o target-clean.exe
```

Features:
- Strings encoded with the `SmartAssembly.Runtime.Strong` family
- Resource compression (`{assembly}.Resources`)
- Method-call hiding (`ProcessCaller` / indirect calls)

de4dot has the best compatibility with SmartAssembly — basically one-click.

---

## .NET Reactor (necrobit)

`.NET Reactor`'s **necrobit** stores the real method bodies encrypted in resources and decrypts/injects them at runtime; the original method bodies are shells. de4dot works on older versions; newer ones (4.x+) often fail.

```text
When de4dot fails:
1. Let the program run (dotnet target.exe, or double-click it)
2. Dump the process memory with MegaDumper / ExtremeDumper → export the decrypted assembly
3. Use de4dot to clean the residual obfuscation from the dump
4. If the metadata is damaged, rebuild it with dnlib (see common-workflow.md)
```

---

## Manual string-decryptor extraction

Obfuscators encrypt strings and restore them at runtime through decryption methods. de4dot usually auto-detects the decryptor; when detection fails, do it manually:

```text
1. Find the decryption method in dnSpyEx (the signature is usually fixed: static string Decrypt(int), or Decrypt(string, int))
   - Telltales: called from many places, numeric constant arguments, returns string
2. Note the method token (e.g. 0x06000012)
3. Point de4dot at the decryptor:
   de4dot --strtyp delegate --strtok 0x06000012 target.exe -o target-clean.exe
```

If the decryption method itself is obfuscated (control-flow flattening), strip the control flow first, then locate the decryptor.

## Common anti-debug techniques

| Technique | Location | Bypass |
|------|------|------|
| `Debugger.IsAttached` check | any method | IL change to `ldc.i4.0; ret`, or patch the getter |
| `Debugger.IsLogging` | — | same as above |
| Timing check (`DateTime.Now` delta) | method entry | patch out the delta comparison |
| `CheckRemoteDebuggerPresent` P/Invoke | — | nop out the call |
| Exception-driven control flow (try/catch path selection) | main logic | cannot simply nop; analyze the real path of the catch block |

> .NET anti-debug is simpler than native — most of it is managed API calls, and a one-line IL change in dnSpyEx is enough.

## Fallbacks when de4dot fails

1. **de4dot --detect** to see the identification result, cross-reference the table above
2. **Runtime dump** (MegaDumper / ExtremeDumper / export the module with Process Hacker)
3. **dnlib script** to solve it manually (see the dnlib section of common-workflow.md)
4. **Dynamic first**: run it, break at the decryption point, read the plaintext directly — you can collect intelligence without unpacking at all

Community references: Washi's blog post "misconceptions-about-dotnet" (common IL analysis misconceptions), the 看雪 (Kanxue) .NET reverse section, Guided Hacking's "Top 5 .NET RE Tools".
