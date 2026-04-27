---
name: secure-coding
description: Write and review code with OWASP Top 10 defenses, authn/z policies, input validation, secrets handling, and crypto best practices. Use when implementing or reviewing authentication, authorization, file uploads, payment flows, transport / CORS / header configuration, or any code handling user input or secrets.
---

# Secure coding

Activates whenever code touches authentication, authorization, secrets, user input, file I/O, deserialization, transport configuration, or external systems. This is the canonical security reference — security is a constraint on every line, not a feature.

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

- **JWT Bearer** or **OpenID Connect**. No homemade auth schemes. No basic auth in production.
- Token validation: `ValidateIssuer = true`, `ValidateAudience = true`, `ValidateLifetime = true`, `ValidateIssuerSigningKey = true`, `ClockSkew = TimeSpan.FromSeconds(30)`.
- Signing keys via JWKS endpoint (auto-rotating) or asymmetric keys from a secret store. **Never** symmetric secrets in `appsettings.json`.
- Refresh tokens rotated on use; server-side revocation list; max 30-day absolute lifetime.

## Authorization

- **Policy-based**, never `[Authorize(Roles = "Admin")]` strings scattered across endpoints.
- Define policies in `Web/Authorization/AuthorizationPolicies.cs` keyed by **resource:action** (`orders:write`, `customers:read`).
- Resource-based decisions (e.g. "user can read this order if they own it") are evaluated **server-side in the Application handler** via `IAuthorizationService` or an `IAuthorizationRule<TRequest>` in the pipeline — never trusted from the client payload.
- Multi-tenant: every query filtered by tenant id at the `IApplicationDbContext` global query filter level. Tenant id from `ICurrentUser`, never from request body.

## Input handling

- Validate everything at the Application boundary via FluentValidation.
- Length-cap every string (e.g. email max 254, name max 200). Reject anything larger than the documented max.
- Reject control characters from text inputs.
- Parse, don't validate-then-pass-string: turn an email string into an `EmailAddress` value object once.
- IDs as strongly-typed types, not raw `Guid`/`int`.

## Injection defenses

- **SQL**: parameterized queries only. EF Core does this; for Dapper, use `@parameters`.
- **Command**: never pass user input to `Process.Start`. If you must, allow-list the executable and arguments.
- **LDAP, XPath, SMTP**: same rule — parameterize or escape.
- **Deserialization**: never `BinaryFormatter`. JSON only via `System.Text.Json` with `MaxDepth = 32` and explicit allowed types.
- **HTML**: API responses are JSON; if any HTML is rendered, escape via `HtmlEncoder.Default.Encode`.

## Secrets

- **Zero secrets in source**, including `appsettings.json` for non-Development.
- Local dev: `dotnet user-secrets`. Cloud: Azure Key Vault / AWS Secrets Manager / HashiCorp Vault.
- Connection strings, API keys, JWT signing keys, encryption keys — all via `IConfiguration` bound to `IOptions<T>` with `ValidateDataAnnotations()` and `ValidateOnStart()`.
- `git secrets` / `trufflehog` runs in pre-commit and CI. Any leaked secret triggers immediate rotation.

## Cryptography

- Hashing passwords: **Argon2id** (preferred) or **PBKDF2** with ≥600k iterations (SHA-256). Never MD5, SHA-1, plain SHA-256 for passwords.
- Symmetric encryption: **AES-256-GCM**. Keys from the secret store, rotated annually.
- Random: `RandomNumberGenerator.GetBytes(...)` only. Never `Random` for security.
- TLS: 1.2 minimum, prefer 1.3. Disable older protocols at the host level.

## HTTPS / transport

- `UseHttpsRedirection()` and `UseHsts()` in production.
- HSTS max-age ≥ 1 year, `includeSubDomains`, `preload`.
- HTTP/2 enabled.

## CORS

- Explicit allow-list. **Never** `AllowAnyOrigin()` outside of local development.
- `AllowCredentials()` only paired with a specific origin.

## Security headers (configured in middleware)

- `Strict-Transport-Security` via HSTS (see above)
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: no-referrer`
- `X-Frame-Options: DENY` (or CSP `frame-ancestors 'none'`)
- `Content-Security-Policy: default-src 'none'` for API endpoints
- `Permissions-Policy` minimized

## Rate limiting

- Global token bucket per IP.
- Stricter limits on `/auth/*` (5 req / min / IP).
- Per-tenant or per-user limits for expensive endpoints, configured via options.

## Logging — what NOT to log

- Passwords, tokens (access/refresh/API keys), secrets.
- Full request bodies on `POST /auth/*`, `/payments/*`.
- PII without redaction (email, phone, full name, IP — context dependent; align with privacy policy).
- Stack traces in responses (only in logs).
- Always structured: `LogInformation("...{UserId}...", userId)`.

## File uploads (when applicable)

- Hard size limit configured in Kestrel + middleware.
- MIME-type sniffing on the server, not trust of client `Content-Type`.
- Allow-list of extensions + magic-byte validation (`MimeDetective` or similar).
- Storage outside web root; serve via signed URLs.

## Dependencies

- Renovate / Dependabot enabled. PRs reviewed for security advisories.
- `dotnet list package --vulnerable --include-transitive` runs in CI; build fails on Critical / High.
- No package from a publisher with fewer than 100k downloads without explicit team agreement in the PR.

## Container & deployment

- Distroless or Chiseled Ubuntu base images. **Never** `latest` tags.
- Run as non-root user.
- Read-only root filesystem. Tmpfs for `/tmp`.
- No secrets baked into images.
- Image scanning in CI (Trivy or Grype). Build fails on Critical / High CVEs.

## Forbidden

- `[ValidateAntiForgeryToken]` skipped on state-changing endpoints (or, for cookie-auth APIs, no SameSite + CSRF protection).
- Custom JWT parsers / validators.
- Logging entire `HttpRequest` or `HttpContext`.
- `eval`, `Activator.CreateInstance` with user input, `Type.GetType(userInput)`.
- Disabling SSL certificate validation in `HttpClientHandler` (yes, even "temporarily for debugging").
- `BinaryFormatter`, dynamic SQL with string concatenation, `Process.Start` with user input.

## Pre-merge checklist

Apply mentally to any change that touches input, auth, or secrets:

- [ ] Inputs validated, length-capped, parsed to value objects where applicable.
- [ ] Authorization policy on the endpoint; resource-based check inside the handler if needed.
- [ ] No new secrets in source; new options bound + validated.
- [ ] No new dependencies without vetting (publisher reputation, advisories).
- [ ] No new `Process.Start`, `BinaryFormatter`, dynamic SQL.
- [ ] Logs reviewed for PII / secrets.
- [ ] CORS / CSP / HSTS unchanged or tightened, never relaxed.

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
