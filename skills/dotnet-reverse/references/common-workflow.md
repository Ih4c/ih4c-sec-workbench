# .NET Reverse Common Workflow

Complete workflow details, IL patch reliability, string decryptor extraction, state machine recognition, dnlib scripting.

## Full workflow (end-to-end)

```text
1. Identify  → confirm it is a .NET managed program (not native)
2. Detect    → identify the obfuscator with DIE / de4dot --detect
3. Deobf     → de4dot deobfuscation (keep the original sample)
4. Static    → locate in the dnSpyEx C# view, inspect key logic in the IL view
5. Dynamic   → set breakpoints on key methods in the dnSpyEx debugger, observe runtime plaintext
6. Patch     → edit with the IL editor, Save Module
```

Artifacts for every step must be persisted: original sample `target.exe` → unpacked `target-clean.exe` → patched `target-patched.exe`.

## IL patch vs C# patch reliability

**Core takeaway: use the IL editor for key modifications, not the C# editor.**

| Dimension | C# editor (Edit Method C#) | IL editor (Edit IL) |
|------|---------------------------|---------------------|
| Compile failure risk | High (missing references, syntax, lambda rewrite failures) | Almost zero |
| Fidelity | Compiler regenerates IL, which may differ from the original IL | In-place replacement, per-instruction edits |
| Suitability | Changing a string, a constant, simple logic | Changing branches, removing checks, altering control flow |
| async/await/state machine | Often fails to compile or distorts the code | Edit state machine fields directly, reliable |

dnSpyEx's C# decompiler is read-only decompilation plus attempted recompilation; recompiling compiler-generated code (state machines, closures, `yield`) fails very easily. The IL editor edits instruction by instruction — what you see is what you get.

### Typical IL patch patterns

```text
Bypassing a check (if (check) → always true):
  before: call bool Foo::Check()
          brfalse.s SKIP
  after:  ldc.i4.1            ; push true
          brfalse.s SKIP      ; the branch is now never taken, so SKIP does not run
  or even more directly:
          ldc.i4.1
          ret                 ; the method directly returns true

Bypassing a check (if (check) → always false):
  ldc.i4.0
  ret

Removing an entire validation block:
  replace everything with nop, or with ret + the correct return value

Changing a string constant:
  the C# editor usually handles strings fine (ldstr swaps the token directly), but if the string lives in resources/encryption you must change the decryption logic

Changing a numeric constant:
  edit the operand of the ldarg / ldc instruction directly
```

## State machine recognition (async/await / yield)

C#'s `async/await` and `IEnumerator` yield compile into a **state machine**: the compiler generates a nested class whose `MoveNext()` dispatches on a `state` field with a switch. The dnSpyEx C# view restores async, but the decompilation may be distorted — the IL view of `MoveNext` is the most accurate.

```text
MoveNext structure of async/await:
  switch(this.<>1__state) {
    case 0: ... logic before await; this.<>1__state = 1; await MoveNext;
    case 1: ... logic after await;
  }

To patch async logic: change the state transitions inside MoveNext, or the checks in the specific case.
Editing async with the C# editor almost always fails → you must use IL.
```

## String decryptor extraction

See `obfuscators.md` for details. This section supplements it with dnlib scripting for bulk string decryption:

```csharp
// dnlib script: scan all string-decryptor call sites, restore the plaintext at runtime, and write it back
// Usage: dotnet script decrypt.csproj target.exe 0x06000012
using System;
using System.Reflection;
using dnlib.DotNet;
using dnlib.DotNet.Writer;
using dnlib.DotNet.Emit;

var module = ModuleDefMD.Load(args[0]);
var decryptorToken = uint.Parse(args[1], System.Globalization.NumberStyles.HexNumber);

// locate the decryption method and invoke it via reflection (the assembly must be loaded into the AppDomain)
// walk every method and replace call Decryptor(token) with ldstr "<decrypted plaintext>"
foreach (var type in module.GetTypes())
    foreach (var method in type.Methods)
    {
        if (!method.HasBody) continue;
        var instrs = method.Body.Instructions;
        for (int i = 0; i < instrs.Count; i++)
        {
            // recognize the call-decryptor pattern, invoke the decryptor to get the plaintext, replace with ldstr
            // (the reflection-invocation boilerplate is omitted here; the idea: load the original assembly →
            //   MethodInfo.Invoke to obtain the plaintext → instrs[i] = OpCodes.Ldstr + operand=<plaintext>)
        }
    }

var opts = new ModuleWriterOptions(module);
module.Write("target-decrypted.exe", opts);
```

dnlib is the de facto standard for .NET metadata programming — de4dot uses it internally. Prefer it when writing custom deobfuscation scripts.

## Dynamic debugging essentials

The dnSpyEx debugger is far friendlier to .NET programs than to native ones:

- **Breakpoint at method entry**: right-click the method → Add Breakpoint
- **Inspecting object values**: once stopped, the Locals / Watch windows directly show object fields and string contents
- **Memory writes**: runtime variable values can be changed directly (Edit Value)
- **Exception breakpoints**: Debug → Exceptions, check the exception types to break on — obfuscators commonly use exception-driven control flow, so breaking on exceptions reveals the real path

### Exception-driven control flow

Some obfuscators stuff normal logic into `try` blocks and use `throw` + `catch` as jumps. Viewed statically the IL looks like exception handling, but it is actually control flow:

```text
try { throw new CustomException(0x42); }
catch (CustomException e) {
    switch(e.Code) {
        case 0x42: real logic A; break;
        case 0x43: real logic B; break;
    }
}
```

Setting an exception breakpoint (break on `CustomException`) and tracking how the `Code` value flows is faster than grinding through the IL.

## Module initializer (Module .cctor)

The .NET module's static constructor (the `.cctor` of `<module>`) runs first when the assembly loads; obfuscators often place anti-tamper / decryption initialization there. Analysis order:

```text
1. Check <module>.cctor first (Module .cctor) — decryption/anti-debug initialization
2. Then look at Program.Main / Startup
3. If anti-tamper lives in .cctor → patch .cctor first, then unpack
```

## Generic patterns for extracting configuration / C2 / keys

Red-team tools and loaders often embed configuration encrypted in resources or fields, decrypted at runtime:

```text
Location flow:
1. strings: check whether plaintext URL/IP exists (usually gone after obfuscation)
2. Find byte[] fields + decryption methods (AES/XOR)
3. Dynamically break at the return point of the decryption method and dump the decrypted plaintext
4. Common: AES-256-CBC with Key==IV (Codegate 2013 pattern; see the .NET section of reverse-engineering/tools.md)
```

Refer to `references/sharp-tools.md` for the concrete configuration structures of red-team tools.

## Boundary with reverse-engineering

- **IL2CPP / NativeAOT** → compiled to native, no CLR metadata → go through `reverse-engineering/` (IDA/r2); this skill only does identification
- **Managed .NET** (standard C# exe/dll, Mono/Unity managed layer, Xamarin) → this skill
- **Hybrid (native loader + .NET payload)** → the loader part goes to `reverse-engineering/`; switch to this skill after dumping the .NET payload

## Artifacts checklist

Every .NET reverse task should produce:
- `target-original.exe` (original sample, untouched)
- `target-clean.exe` (after de4dot unpacking)
- `notes.md` (identified obfuscators, decryptor tokens, key method addresses, configuration/C2/key)
- `target-patched.exe` (after patching, if needed)
- `il-diff.txt` (IL comparison before/after patching, when patching was done)
