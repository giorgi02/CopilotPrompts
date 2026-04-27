---
name: secure-coding
description: Write and review code with OWASP Top 10 defenses, authn/z policies, input validation, secrets handling, and crypto best practices. Use when implementing or reviewing authentication, authorization, file uploads, payment flows, or any code handling user input or secrets.
---

# Secure coding

Activates whenever code touches authentication, authorization, secrets, user input, file I/O, deserialization, or external systems. See [.github/instructions/security.instructions.md](../../instructions/security.instructions.md).

## OWASP Top 10 mapping

| Category | This codebase's defense |
|---|---|
| A01 Broken Access Control | Policy-based authorization, multi-tenant query filter, resource-based rules in handlers |
| A02 Cryptographic Failures | Argon2id passwords, AES-256-GCM, asymmetric JWT signing, TLS 1.2+ |
| A03 Injection | Parameterized queries, value-object input parsing, no `Process.Start` with user input |
| A04 Insecure Design | Result pattern + aggregate invariants, FluentValidation pipeline, threat modeling per feature |
| A05 Security Misconfiguration | `ValidateOnStart` for options, distroless containers, security headers middleware |
| A06 Vulnerable Components | Dependabot, `dotnet list package --vulnerable` in CI, image scanning |
| A07 Identification & Auth Failures | OIDC/JWT, refresh rotation, server-side revocation, rate-limited `/auth/*` |
| A08 Software & Data Integrity | Image signing, SBOM in CI, `BinaryFormatter` forbidden |
| A09 Logging & Monitoring | Structured logs, OTel traces+metrics, no secrets/PII in logs |
| A10 SSRF | Allow-list outbound URLs, `IHttpClientFactory` typed clients, no user-controlled URL fetches |

## Authentication

- JWT Bearer or OpenID Connect. No homemade auth.
- Token validation flags all `true`, `ClockSkew = 30s`.
- Asymmetric signing keys via JWKS or secret store.
- Refresh tokens rotated on use; revocation list server-side; max 30-day absolute lifetime.

## Authorization

- **Policy-based**, never `[Authorize(Roles = "Admin")]` strings scattered.
- Policy keys are `resource:action` (`orders:write`, `customers:read`).
- Resource-based decisions evaluated **server-side in the handler**, never trusted from the client payload.
- Multi-tenant: every query filtered by tenant id at the global query filter level. Tenant id from `ICurrentUser`, never from request body.

## Input handling

- Validate everything at the Application boundary via FluentValidation.
- Length-cap every string (e.g. email max 254, name max 200).
- Reject control characters from text inputs.
- Parse don't validate: turn an email string into an `EmailAddress` value object.
- IDs as strongly-typed types, not raw `Guid`/`int`.

## Injection defenses

- SQL: parameterized only. EF does this; Dapper uses `@params`.
- Command: never `Process.Start` with user input. Allow-list executable + arguments if you must.
- LDAP, XPath, SMTP: parameterize / escape.
- Deserialization: never `BinaryFormatter`. JSON only via `System.Text.Json`, `MaxDepth = 32`.
- HTML: API is JSON; if HTML rendered, use `HtmlEncoder.Default.Encode`.

## Secrets

- Zero secrets in source or `appsettings.json` (non-Development).
- `dotnet user-secrets` locally; cloud secret store in deployed envs.
- Bind to `IOptions<T>` with `ValidateDataAnnotations()` and `ValidateOnStart()`.
- `git secrets` / `trufflehog` in pre-commit and CI; leaked secret = immediate rotation.

## Cryptography

- Passwords: Argon2id (preferred) or PBKDF2 ≥600k iterations.
- Symmetric: AES-256-GCM. Keys from secret store, rotated annually.
- Random: `RandomNumberGenerator.GetBytes(...)`. Never `Random` for security.
- TLS: 1.2 minimum, prefer 1.3.

## Headers (configured in middleware)

- HSTS, CSP, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `X-Frame-Options: DENY` (or CSP `frame-ancestors`).

## Rate limiting

- Global token bucket per IP.
- Stricter `/auth/*` (5 req/min/IP).
- Per-tenant or per-user limits for expensive endpoints.

## Logging hygiene

- Never log: secrets, tokens, full request bodies for sensitive routes, PII.
- Stack traces: yes in logs, no in user responses.
- `LogInformation("...{UserId}...", userId)` — structured only.

## File uploads (when applicable)

- Hard size limit in Kestrel + middleware.
- Magic-byte sniffing on the server.
- Allow-list extensions.
- Storage outside web root; signed URLs to serve.

## Container & deploy

- Distroless / Chiseled base image.
- Non-root user, read-only root FS, tmpfs `/tmp`.
- No `latest` tag.
- Trivy / Grype scan in CI; fail on Critical / High.

## Pre-merge checklist

Apply mentally to any change that touches input, auth, or secrets:

- [ ] Inputs validated, length-capped, parsed to value objects where applicable.
- [ ] Authorization policy on the endpoint; resource-based check inside the handler if needed.
- [ ] No new secrets in source; new options bound + validated.
- [ ] No new dependencies without vetting (publisher reputation, advisories).
- [ ] No new `Process.Start`, `BinaryFormatter`, dynamic SQL.
- [ ] Logs reviewed for PII / secrets.

## Output when reviewing or implementing

```
Security implications: <none | list>
OWASP categories engaged: <A01, A03, ...>
Defenses applied:
  - <defense> in <file>:<line>
Defenses missing (must fix):
  - <gap> — <fix>
PII / secret leak risk: <none | list with file:line>
```

See [security instructions](../../instructions/security.instructions.md).
