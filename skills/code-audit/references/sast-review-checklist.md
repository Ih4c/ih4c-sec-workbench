# Code Audit Checklist (condensed)

- [ ] List of all external input entry points
- [ ] Auth/auth middleware coverage
- [ ] Whether multi-tenant IDs are bound to the session
- [ ] Deserialization / pickle / YAML load
- [ ] SSRF outbound access and protocol restrictions
- [ ] Key and token storage
- [ ] File upload paths and types
- [ ] Dangerous exec/system/Runtime
