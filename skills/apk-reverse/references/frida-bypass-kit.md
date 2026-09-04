# Frida Bypass Kit — Universal Android Security Bypass Framework

> Source: [FridaBypassKit](https://github.com/okankurtuluss/FridaBypassKit) (2025)
> Use case: when APK dynamic analysis needs to bypass root detection, SSL pinning, emulator detection, and anti-debugging

## Overview

FridaBypassKit is a Frida script that combines four bypass capabilities. It needs no per-app customization and works out of the box.

## The Four Bypass Capabilities

### 1. Root Detection Bypass

- Hooks `File.exists()` to hide su binaries
- Intercepts `Runtime.exec()` root-check calls
- Hides root-related packages (Magisk, SuperSU, etc.) from PackageManager
- Modifies system properties so the device looks unrooted

### 2. SSL Pinning Bypass

- Hooks `TrustManagerImpl.verifyChain()`
- Hooks `TrustManagerImpl.checkTrustedRecursive()`
- Bypasses certificate chain verification
- Returns an empty certificate chain to avoid validation
- Compatible with OkHttp, Retrofit, and custom implementations

### 3. Emulator Detection Bypass

- Fakes TelephonyManager return values
- Returns fake phone numbers and carrier names
- Modifies Build properties

### 4. Anti-Debugging Bypass

- Hooks `Debug.isDebuggerConnected()`
- Blocks debugger detection
- Bypasses anti-debugging checks

## Usage

```bash
# Prerequisites
pip install frida-tools
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell su -c /data/local/tmp/frida-server &

# Inject into the target app
frida -U -f com.example.app -l FridaBypassKit.js
```

## Other Recommended Frida Bypass Scripts

| Project | Highlights | Link |
|---------|------------|------|
| httptoolkit/frida-interception-and-unpinning | Directly MitM all HTTPS traffic | [GitHub](https://github.com/httptoolkit/frida-interception-and-unpinning) |
| 0xCD4/SSL-bypass | Generic non-customized SSL bypass | [GitHub](https://github.com/0xCD4/SSL-bypass) |
| incogbyte/ssl-bypass gist | Bypasses common SSL pinning approaches | [Gist](https://gist.github.com/incogbyte/1e0e2f38b5602e72b1380f21ba04b15e) |
| Zero3141/Frida-OkHttp-Bypass | Targeted at OkHttp CertificatePinner | [GitHub](https://github.com/Zero3141/Frida-OkHttp-Bypass) |

## Integration with This Package

Use it inside the `apk-reverse` workflow when you hit the following situations:

1. The app detects root and refuses to run → enable Root Detection Bypass
2. HTTPS request bodies stay encrypted during capture → enable SSL Pinning Bypass
3. The app detects the emulator and refuses to run → enable Emulator Detection Bypass
4. The app crashes after attaching Frida → enable Debug Detection Bypass

Recommended combination: run the full FridaBypassKit first, then tune it for the specific target.
