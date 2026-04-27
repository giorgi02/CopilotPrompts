---
description: 'Security review of a file, endpoint, or PR using OWASP Top 10 and project security rules.'
agent: 'agent'
tools: ['search/codebase', 'search/usages']
argument-hint: '<file, endpoint, or PR>'
---

# Security review

Apply [security instructions](../instructions/security.instructions.md). Map every finding to an OWASP Top 10 category where possible.

## Scan checklist

### Authentication
- New endpoints have `RequireAuthorization(...)` unless explicitly public.
- No homemade JWT validation.
- No basic auth.
- Refresh token rotation in place where applicable.

### Authorization
- Resource-based decisions evaluated **on the server**, never trusted from the client.
- Multi-tenant queries filter by tenant id at the global query filter.
- No `Roles =` strings scattered in code — policies only.

### Input
- Every command/query has a validator.
- Every string has a max length.
- Every collection has a max count.
- IDs in path parameters are bound to strongly-typed IDs, not raw `Guid`/`int`.

### Injection
- No string concatenation for SQL.
- No `Process.Start` with user input.
- No deserialization of untrusted input via `BinaryFormatter` / `XmlSerializer` / arbitrary types.
- `JsonSerializerOptions.MaxDepth` set.

### Secrets
- No keys, passwords, or connection strings in source or `appsettings.json` (non-Development).
- No secret values in logs.

### Crypto
- No MD5 / SHA-1 / plain SHA-256 for passwords.
- No `Random` for security-sensitive randomness — only `RandomNumberGenerator`.
- TLS 1.2+ enforced.

### Logging
- No secrets, tokens, full request bodies, or PII in logs.
- No exception messages or stack traces in user-facing responses.

### Headers
- HSTS, CSP, `X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options` (or CSP `frame-ancestors`).

### Dependencies
- No new packages from low-reputation publishers.
- `dotnet list package --vulnerable` clean.
- Container base image not `latest`, runs as non-root.

## Output

```
## Security review of <scope>

### Critical (block merge)
- <OWASP A0X> — <file>:<line> — <one-line>. Fix: <concrete change>.

### High
- <OWASP A0X> — <file>:<line> — <one-line>. Fix: <change>.

### Medium / Low
- <area> — <file>:<line> — <comment>.

### Verified safe
- <list of areas you specifically checked and found clean>

### Verdict
<Block | Request changes | Approve> — <one-sentence reason>
```

## Discipline

- Cite the OWASP category (e.g. `A01:2021 — Broken Access Control`) where applicable.
- Cite the specific rule from [security.instructions.md](../instructions/security.instructions.md) by section.
- A finding without a concrete fix is not a finding — it's a gripe. Always include a fix.
