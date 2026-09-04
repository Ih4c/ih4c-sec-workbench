# APK Security Testing Cheatsheet

> Compiled from the OWASP MASTG (Mobile Application Security Testing Guide).
> Covers six dimensions: static analysis, dynamic analysis, network communications, data storage, authentication/authorization, and code protection.

---

## Static Analysis Checklist

### Manifest Audit

```text
□ android:debuggable="true" → debuggable (should not appear in production)
□ android:allowBackup="true" → data can be extracted via backup
□ Components with android:exported="true" → exposed Activity/Service/Receiver/Provider
□ Custom permission protectionLevel → is it normal (should be signature)
□ scheme in intent-filter → can the custom deeplink be hijacked
□ android:usesCleartextTraffic="true" → cleartext HTTP allowed
□ minSdkVersion too low → may lack security features
```

### Key Code Audit Points

```text
□ Hardcoded keys/tokens (search for "key", "secret", "password", "api_key")
□ Insecure randomness (java.util.Random instead of SecureRandom)
□ Insecure crypto (ECB mode, DES, MD5 for passwords)
□ WebView configuration (setJavaScriptEnabled + addJavascriptInterface = RCE risk)
□ SQL injection (rawQuery concatenating user input)
□ Path traversal (ContentProvider's openFile without path validation)
□ Log leakage (Log.d/Log.i printing sensitive info)
□ Clipboard leakage (ClipboardManager storing sensitive data)
□ Implicit Intent leakage (sendBroadcast without a package name)
```

### Third-Party Library Audit

```text
□ Outdated OkHttp/Retrofit versions (known vulnerabilities)
□ Outdated WebView kernel
□ SDKs with known vulnerabilities (check CVEs)
□ Ad SDK data collection scope
□ Push SDK configuration (does it leak tokens)
```

---

## Dynamic Analysis Checklist

### Priority Frida Hook Targets

| Target | Hook point | Purpose |
|------|---------|------|
| Login/auth | `LoginActivity.login()` | Observe credential handling |
| Signature generation | `*Sign*`、`*sign*`、`*encrypt*` | Recover the signing algorithm |
| SSL Pinning | `CertificatePinner.check` | Bypass for packet capture |
| Root detection | `*root*`、`*su*`、`*magisk*` | Bypass the detection |
| Crypto operations | `javax.crypto.Cipher` | Extract keys/IV |
| Token storage | `SharedPreferences.getString` | Observe token read/write |
| Network requests | `OkHttpClient.newCall` | Observe request construction |

### Handy One-Line Frida Commands

```bash
# Trace all crypto operations
frida-trace -U -f com.target.app -j '*Cipher*!*'

# Trace all HTTP requests
frida-trace -U -f com.target.app -j '*OkHttp*!*'

# Trace SharedPreferences reads/writes
frida-trace -U -f com.target.app -j '*SharedPreferences*!*'

# Trace all native function calls
frida-trace -U -f com.target.app -i 'Java_*'
```

### Quick Objection Commands

```bash
# Connect
objection -g com.target.app explore

# Common commands
android hooking list activities
android hooking list services
android sslpinning disable
android root disable
android clipboard monitor
env                              # view the app directory
sqlite connect <db_path>         # connect to a database
```

---

## Network Communications Security

### Packet Capture Setup

```text
Method 1: System proxy + Burp/mitmproxy
- Set the WiFi proxy → Burp listening address
- Install the CA certificate on the device
- Android 7+ requires network_security_config or a Frida bypass

Method 2: VPN mode (recommended)
- Use HttpCanary / Packet Capture
- No root required, no proxy configuration needed
- But cannot decrypt SSL-pinned traffic

Method 3: Frida + r2frida
- Intercept network calls directly inside the process
- Not restricted by proxy/VPN
```

### Check Items

```text
□ Is HTTPS used (for all API calls)
□ Is SSL Pinning (certificate binding) present
□ Is certificate validation correct (self-signed not accepted)
□ Is there a certificate transparency (CT) check
□ Are API keys transmitted in cleartext in requests
□ Do tokens have an expiration mechanism
□ Is there request signing to prevent tampering
□ Is there replay attack protection (nonce/timestamp)
□ Are WebSockets encrypted
□ Is any sensitive data passed in URL parameters (which gets logged)
```

---

## Data Storage Security

### Locations to Check

| Location | Risk | Check command |
|------|------|---------|
| SharedPreferences | Tokens/passwords stored in cleartext | `adb shell cat /data/data/pkg/shared_prefs/*.xml` |
| SQLite databases | Unencrypted sensitive data | `adb pull /data/data/pkg/databases/` |
| External storage | Readable by any app | `adb shell ls /sdcard/Android/data/pkg/` |
| App logs | Debug info leakage | `adb logcat \| grep pkg` |
| Backup files | allowBackup=true | `adb backup -f backup.ab pkg` |
| Keyboard cache | Input history | Check whether `inputType` is `textPassword` |
| Screenshot protection | Sensitive pages can be screenshotted | Check for `FLAG_SECURE` |

### Encrypted Storage Options Compared

| Option | Security | Notes |
|------|--------|------|
| SharedPreferences cleartext | ❌ | Directly readable after root |
| EncryptedSharedPreferences | ✓ | AndroidX Security library |
| SQLCipher | ✓ | Encrypted SQLite |
| Android Keystore | ✓✓ | Hardware-level key protection |
| Custom AES encryption | ⚠️ | Depends on key management |

---

## Authentication and Authorization

### Common Vulnerabilities

| Vulnerability | Test method |
|------|---------|
| Weak password policy | Try 123456, password, etc. |
| No lockout mechanism | Brute force the login endpoint |
| Tokens never expire | Replay an old token after logout |
| Privilege escalation (越权) | Modify user_id in requests |
| SMS verification code brute-forceable | 4/6-digit code with no rate limiting |
| Misconfigured OAuth | redirect_uri can be tampered with |
| Biometric auth bypass | Hook BiometricPrompt |
| Device binding bypass | Modify device_id |

### Test Payloads

```bash
# Privilege escalation test
curl -H "Authorization: Bearer USER_A_TOKEN" \
     "https://api.target.com/users/USER_B_ID/profile"

# Token replay
# 1. Log in normally to obtain a token
# 2. Log out
# 3. Request with the old token → should return 401

# SMS verification code brute force
for code in $(seq 0000 9999); do
    curl -X POST "https://api.target.com/verify" \
         -d "phone=13800138000&code=$code"
done
```

---

## Code Protection Assessment

| Protection | Detection method | Bypass difficulty |
|---------|---------|---------|
| ProGuard obfuscation | Check in jadx whether class names are a/b/c | Low (renaming only) |
| String encryption | Find the decryption function, Hook to get plaintext | Medium |
| Anti-debugging | Try to attach a debugger | Medium (Frida can bypass) |
| Root detection | Run on a rooted device | Medium (generic scripts bypass) |
| Emulator detection | Run on an emulator | Low-medium |
| Integrity checks | Install after modifying the APK | Medium (patch the check function) |
| Hardening/packer (加固/壳) | Look at the entry class and .so files | Medium-high (unpacking required) |
| Native protection | Core logic in .so | High (IDA analysis required) |
| VMP virtualization | Code executed in virtualized form | Very high |

---

## Quick Test Flow (30 minutes)

```text
1. [5min] Unpack + Manifest audit
   apktool d app.apk
   Check debuggable/allowBackup/exported/cleartext

2. [10min] Quick code audit
   jadx -d out app.apk
   Search for: password, key, secret, token, http://

3. [5min] Network testing
   Configure proxy → operate the app → check for cleartext/weak crypto

4. [5min] Storage checks
   adb shell → check shared_prefs and databases

5. [5min] Dynamic verification
   Frida hook key functions → confirm findings
```
