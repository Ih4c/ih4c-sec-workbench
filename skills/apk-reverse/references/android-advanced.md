# Android Advanced Reversing Reference

> Covers Native SO analysis, advanced Frida usage, SSL Pinning bypass, root-detection evasion, hardening/unpacking (加固脱壳), and Flutter/React Native reversing.

---

## Native SO Reversing

### Analysis Workflow

```text
1. Extract the .so files from the APK
   unzip app.apk lib/arm64-v8a/*.so -d extracted/

2. Confirm the architecture and basic info
   file libxxx.so
   rabin2 -I libxxx.so

3. Locate the JNI entry points
   - Search for JNI_OnLoad (dynamic registration)
   - Search for Java_com_xxx_yyy (static registration)
   - nm -D libxxx.so | grep -i java

4. Load and analyze in IDA/Ghidra
   - Import the JNI headers (jni.h types)
   - Annotate the JNIEnv* parameters
   - Find RegisterNatives calls (function table of dynamic registration)

5. Pinpoint key logic
   - Track back from native method names at the Java layer
   - Cross-reference from strings (keys, URLs, error messages)
   - Trace crypto library calls (AES/MD5/SHA)
```

### JNI Function Registration

```c
// Static registration: function name = Java_package_class_method
JNIEXPORT jstring JNICALL Java_com_example_app_Security_getSign(
    JNIEnv *env, jobject thiz, jstring input) { ... }

// Dynamic registration: call RegisterNatives in JNI_OnLoad
static JNINativeMethod methods[] = {
    {"getSign", "(Ljava/lang/String;)Ljava/lang/String;", (void*)native_getSign},
};

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    vm->GetEnv((void**)&env, JNI_VERSION_1_6);
    jclass clazz = env->FindClass("com/example/app/Security");
    env->RegisterNatives(clazz, methods, sizeof(methods)/sizeof(methods[0]));
    return JNI_VERSION_1_6;
}
```

### JNI Analysis Tips in IDA

```text
1. Import the JNI type library
   File → Load File → Parse C Header → jni.h

2. Annotate the first parameter as JNIEnv*
   Right-click parameter → Set type → JNIEnv*
   Calls such as env->FindClass / env->GetMethodID are then recognized automatically

3. Find RegisterNatives
   Search for calls through the JNIEnv vtable offset 0x35C (ARM64)
   → The third argument is the JNINativeMethod array
   → Extract all native function addresses from the array
```

---

## Advanced Frida Usage

### Hooking Native Functions

```javascript
// Hook libc functions
Interceptor.attach(Module.findExportByName("libc.so", "open"), {
    onEnter: function(args) {
        this.path = args[0].readUtf8String();
        console.log("[open] " + this.path);
    },
    onLeave: function(retval) {
        if (this.path.includes("su") || this.path.includes("magisk")) {
            console.log("[open] Blocked root check: " + this.path);
            retval.replace(-1);  // return failure
        }
    }
});

// Hook a function inside a custom SO
var base = Module.findBaseAddress("libsecurity.so");
var targetFunc = base.add(0x1234);  // offset address
Interceptor.attach(targetFunc, {
    onEnter: function(args) {
        console.log("arg0: " + args[0].readUtf8String());
    },
    onLeave: function(retval) {
        console.log("return: " + retval.readUtf8String());
    }
});
```

### Hooking Java Methods

```javascript
Java.perform(function() {
    // Hook an instance method
    var Security = Java.use("com.example.app.Security");
    Security.getSign.implementation = function(input) {
        console.log("[getSign] input: " + input);
        var result = this.getSign(input);  // call the original method
        console.log("[getSign] output: " + result);
        return result;
    };

    // Hook a constructor
    Security.$init.overload('java.lang.String').implementation = function(key) {
        console.log("[Security.<init>] key: " + key);
        this.$init(key);
    };

    // Hook an overloaded method
    Security.encrypt.overload('java.lang.String', 'int').implementation = function(data, mode) {
        console.log("[encrypt] data=" + data + " mode=" + mode);
        return this.encrypt(data, mode);
    };
});
```

### Memory Searching and Patching

```javascript
// Search a string in memory
Process.enumerateModules().forEach(function(module) {
    if (module.name === "libtarget.so") {
        Memory.scan(module.base, module.size, "48 65 6C 6C 6F", {  // "Hello"
            onMatch: function(address, size) {
                console.log("Found at: " + address);
            }
        });
    }
});

// Patch memory (patch instructions)
var addr = Module.findBaseAddress("libsecurity.so").add(0x5678);
Memory.patchCode(addr, 4, function(code) {
    var writer = new Arm64Writer(code, {pc: addr});
    writer.putNop();  // replace with NOP
    writer.flush();
});
```

---

## SSL Pinning Bypass

### Generic Approach (recommended)

```javascript
// Generic Frida SSL Pinning bypass
// Source: https://github.com/0xCD4/SSL-bypass
Java.perform(function() {
    // 1. TrustManager bypass
    var TrustManager = Java.registerClass({
        name: 'com.custom.TrustManager',
        implements: [Java.use('javax.net.ssl.X509TrustManager')],
        methods: {
            checkClientTrusted: function(chain, authType) {},
            checkServerTrusted: function(chain, authType) {},
            getAcceptedIssuers: function() { return []; }
        }
    });

    // 2. SSLContext replacement
    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    var sslContext = SSLContext.getInstance("TLS");
    sslContext.init(null, [TrustManager.$new()], null);

    // 3. OkHttp CertificatePinner bypass
    try {
        var CertificatePinner = Java.use('okhttp3.CertificatePinner');
        CertificatePinner.check.overload('java.lang.String', 'java.util.List').implementation = function() {};
    } catch(e) {}
});
```

### Per-Framework Bypasses

| Framework | Bypass method |
|------|---------|
| OkHttp3 | Hook `CertificatePinner.check` to return empty |
| Retrofit | Same as OkHttp (OkHttp underneath) |
| Volley | Hook the SSL factory of `HurlStack` |
| Flutter | Hook the `SecurityContext` of `dart:io` (special script required) |
| React Native | Hook `OkHttpClientProvider` |
| WebView | Hook `WebViewClient.onReceivedSslError` |

### Flutter-Specific

```javascript
// Flutter SSL Pinning bypass (requires locating the ssl_verify_peer_cert function)
var flutter_lib = Module.findBaseAddress("libflutter.so");
// Search for the ssl_verify_peer_cert signature
var pattern = "FF 03 05 D1 FD 7B 0F A9";  // ARM64 signature
Memory.scan(flutter_lib, Module.findModuleByName("libflutter.so").size, pattern, {
    onMatch: function(address) {
        Interceptor.replace(address, new NativeCallback(function() {
            return 0;  // return success
        }, 'int', []));
    }
});
```

---

## Root Detection Bypass

### Common Detection Methods

| Detection method | Bypass method |
|---------|---------|
| Check `/system/app/Superuser.apk` | Hook `File.exists()` to return false |
| Check the `su` command | Hook `Runtime.exec()` to intercept su calls |
| Check `/proc/self/mounts` | Hook file reads, filter out magisk-related entries |
| SafetyNet/Play Integrity | Magisk Hide / Zygisk + Shamiko |
| Check Magisk package name | Randomize the Magisk package name |
| Check `/data/adb/` | Hook `opendir`/`access` |

### Generic Frida Root Bypass

```javascript
Java.perform(function() {
    // Hook File.exists
    var File = Java.use("java.io.File");
    File.exists.implementation = function() {
        var path = this.getAbsolutePath();
        var blacklist = ["su", "Superuser", "magisk", "busybox", "xposed"];
        for (var i = 0; i < blacklist.length; i++) {
            if (path.toLowerCase().includes(blacklist[i])) {
                return false;
            }
        }
        return this.exists();
    };

    // Hook System.getProperty
    var System = Java.use("java.lang.System");
    System.getProperty.overload('java.lang.String').implementation = function(key) {
        if (key === "ro.debuggable" || key === "ro.secure") {
            return "1";
        }
        return this.getProperty(key);
    };
});
```

---

## Packer/Hardening Identification and Unpacking (加固/壳识别与脱壳)

### Common Hardening Vendors

| Hardening | Identifying feature | Unpacking method |
|------|---------|---------|
| 360 加固 | `libjiagu.so`、`com.stub.StubApp` | FART / Frida dump dex |
| 腾讯乐固 | `libshell*.so`、`com.tencent.StubShell` | FART / BlackDex |
| 梆梆加固 | `libDexHelper.so`、`com.secneo.apkwrapper` | FART |
| 爱加密 | `libexec.so`、`s.h.e.l.l` | Frida dump |
| 网易易盾 | `libnesec.so` | Frida dump |
| 娜迦 | `libnaga.so` | Frida dump |

### Generic Unpacking Methods

```text
Method 1: FART (unpack in ART environment)
- Flash a FART ROM or use the Frida-based FART
- Automatically dumps all dex files loaded by any ClassLoader

Method 2: Frida DEX Dump
- frida -U -f com.target.app -l dex_dump.js
- Hook at DexFile::OpenMemory and dump the dex from memory

Method 3: BlackDex
- Rootless unpacking tool
- Install the BlackDex APK directly, then select the target app to unpack

Method 4: Manual dump
- Enumerate all ClassLoaders with Frida
- Find the app's ClassLoader → obtain the DexFile objects
- Read and save the dex memory regions
```

### Frida DEX Dump Script

```javascript
Java.perform(function() {
    Java.enumerateClassLoaders({
        onMatch: function(loader) {
            try {
                var dexFiles = loader.getDexFileList();
                console.log("ClassLoader: " + loader);
                console.log("  DEX files: " + dexFiles);
            } catch(e) {}
        },
        onComplete: function() {}
    });
});
```

---

## React Native / Flutter Reversing

### React Native

```text
1. Unpack the APK → assets/index.android.bundle (JS code)
2. Format the JS → search for API addresses, keys, signing logic
3. If Hermes bytecode is present (.hbc files) → decompile with hermes-dec
4. Hooking: hook ReactBridge at the Java layer with Frida
```

### Flutter

```text
1. Flutter code compiles to libapp.so (Dart AOT)
2. Cannot be decompiled directly back to Dart source
3. Analysis methods:
   - reFlutter: patches libflutter.so to obtain a snapshot
   - Doldrums: parses the Dart snapshot to recover class/function info
   - Hook key functions inside libflutter.so with Frida
4. Network analysis: Flutter does not use the system proxy, SSL needs special handling
```

---

## Tool Cheatsheet

| Tool | Purpose | Install |
|------|------|------|
| jadx | Java decompiler | Already in bootstrap |
| apktool | Unpack/repack | Already in bootstrap |
| Frida | Dynamic hooking | `pip install frida-tools` |
| Objection | Frida wrapper (easier to use) | `pip install objection` |
| MobSF | Automated mobile security analysis | Docker deployment |
| BlackDex | Rootless unpacking | APK install |
| FART | ART unpacking | Flash a ROM or use the Frida version |
| hermes-dec | Hermes bytecode decompiler | npm install |
| reFlutter | Flutter reversing helper | pip install |
| Magisk + Shamiko | Root hiding | Flash |

---

## Reference Resources

| Resource | Description | Link |
|------|------|------|
| OWASP MASTG | Mobile app security testing guide | https://mas.owasp.org/ |
| FridaBypassKit | Generic bypass framework | https://github.com/okankurtuluss/FridaBypassKit |
| SSL-bypass | Generic SSL Pinning bypass | https://github.com/0xCD4/SSL-bypass |
| awesome-frida | Frida resource collection | https://github.com/dweinstein/awesome-frida |
| Android Security Awesome | Android security resources | https://github.com/ashishb/android-security-awesome |
