---
name: 'Security conventions'
description: 'Authentication, authorization, secrets, input handling, and OWASP Top 10 defenses applied to every file.'
applyTo: '**/*.cs,**/*.csproj,**/Dockerfile,**/docker-compose*.yml,**/appsettings*.json'
---

# Security conventions

This file is **always** active. Security is not a feature; it is a constraint on every line.

## Authentication

- **JWT Bearer** or **OpenID Connect**. No homemade auth schemes. No basic auth in production.
- Token validation: `ValidateIssuer = true`, `ValidateAudience = true`, `ValidateLifetime = true`, `ValidateIssuerSigningKey = true`, `ClockSkew = TimeSpan.FromSeconds(30)`.
- Signing keys via JWKS endpoint (auto-rotating) or asymmetric keys from a secret store. **Never** symmetric secrets in `appsettings.json`.
- Refresh tokens: rotated on use, server-side revocation list, max 30-day absolute lifetime.

## Authorization

- **Policy-based**, never `[Authorize(Roles = "Admin")]` scattered across endpoints.
- Define policies in `Web/Authorization/AuthorizationPolicies.cs` keyed by **resource:action** (`orders:write`, `customers:read`).
- Resource-based authorization (e.g. "user can read this order if they own it") is evaluated **inside the Application handler** via `IAuthorizationService` or an `IAuthorizationRule<TRequest>` in the pipeline.
- Multi-tenant: every query filters by tenant id at the `IApplicationDbContext` global query filter level. Never trust a tenant id in a request body.

## Input handling

- Validate everything at the boundary: `FluentValidation` on commands/queries.
- Length-cap every string input. Reject anything larger than the documented max.
- Reject control characters from text inputs.
- Parse, do not validate-then-pass-string: turn an email string into an `EmailAddress` value object once.

## Injection defenses

- **SQL**: parameterized queries only. EF Core does this; for Dapper, use `@parameters`.
- **Command injection**: never pass user input to `Process.Start`. If you must, allow-list the executable and arguments.
- **LDAP, XPath, SMTP**: same rule — parameterize or escape.
- **Deserialization**: never `BinaryFormatter`. JSON only via `System.Text.Json` with `MaxDepth = 32` and explicit allowed types.
- **HTML**: API responses are JSON; if any HTML is rendered, escape via `HtmlEncoder.Default.Encode`.

## Secrets

- **Zero secrets in source**, including `appsettings.json` for non-development.
- Local dev: `dotnet user-secrets`. Cloud: Azure Key Vault / AWS Secrets Manager / HashiCorp Vault.
- Connection strings, API keys, JWT signing keys, encryption keys — all via `IConfiguration` with `IOptions<T>` validated at startup.
- `git secrets` / `trufflehog` runs in pre-commit and CI. Any leaked secret triggers immediate rotation.

## Cryptography

- Hashing passwords: **Argon2id** (preferred) or **PBKDF2** with ≥600k iterations (SHA-256). Never MD5, SHA-1, plain SHA-256 for passwords.
- Symmetric encryption: **AES-256-GCM**. Keys from the secret store, rotated annually.
- Random: `RandomNumberGenerator.GetBytes(...)` only. Never `Random` for security.
- TLS: 1.2 minimum, prefer 1.3. Disable older protocols at the host level.

## HTTPS / transport

- `UseHttpsRedirection()` and `UseHsts()` in production. HSTS max-age ≥ 1 year, `includeSubDomains`, `preload`.
- HTTP/2 enabled.

## CORS

- Explicit allow-list. **Never** `AllowAnyOrigin()` outside of local development.
- `AllowCredentials()` only with a specific origin.

## Headers

- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: no-referrer`
- `X-Frame-Options: DENY` (or CSP `frame-ancestors 'none'`)
- `Content-Security-Policy: default-src 'none'` for API endpoints
- `Permissions-Policy` minimized

## Rate limiting

- Global token bucket per IP. Stricter limits on `/auth/*` (5 req / min / IP).
- Per-tenant or per-user limits for expensive endpoints, configured via options.

## Logging — what NOT to log

- Passwords, tokens (access/refresh/API keys), secrets.
- Full request bodies on `POST /auth/*`, `/payments/*`.
- PII without redaction (email, phone, full name, IP — context dependent; align with privacy policy).
- Stack traces in responses (only in logs).

## File uploads (when applicable)

- Hard size limit configured in Kestrel + middleware.
- MIME-type sniffing on the server, not trust of client `Content-Type`.
- Allow-list of extensions + magic-byte validation (`MimeDetective` or similar).
- Storage outside web root; serve via signed URLs.

## Dependencies

- Renovate / Dependabot enabled. PRs reviewed for security advisories.
- `dotnet list package --vulnerable --include-transitive` runs in CI; build fails on Critical/High.
- No package from a publisher with fewer than 100k downloads without explicit team agreement in the PR.

## Container & deployment

- Distroless or Chiseled Ubuntu base images. **Never** `latest` tags.
- Run as non-root user.
- Read-only root filesystem. Tmpfs for `/tmp`.
- No secrets baked into images.
- Image scanning in CI (Trivy or Grype). Build fails on Critical/High CVEs.

## Forbidden

- `[ValidateAntiForgeryToken]` skipped on state-changing endpoints (or, for cookie-auth APIs, no SameSite + CSRF protection).
- Custom JWT parsers / validators.
- Logging entire `HttpRequest` or `HttpContext`.
- `eval`, `Activator.CreateInstance` with user input, `Type.GetType(userInput)`.
- Disabling SSL certificate validation in `HttpClientHandler` (yes, even "temporarily for debugging").

See [.github/PULL_REQUEST_TEMPLATE.md](../PULL_REQUEST_TEMPLATE.md) for the per-PR review checklist.
