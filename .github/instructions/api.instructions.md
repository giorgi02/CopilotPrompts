---
name: 'REST API design conventions'
description: 'Resource modeling, status codes, pagination, idempotency, and error contract conventions for HTTP endpoints.'
applyTo: 'src/Web/**/*Endpoint*.cs,src/Web/**/Endpoints/**/*.cs'
---

# REST API design conventions

## Resource modeling

- URLs are **nouns**, plural, lower kebab-case: `/api/v1/orders`, `/api/v1/customers/{customerId}/addresses`.
- Verbs in URLs only when the operation does not map to a CRUD primitive on a resource: `/api/v1/orders/{id}/cancel`. Use `POST` for these state-transition endpoints.
- Nest resources at most **one level deep**. Beyond one level, prefer query parameters or top-level resources.
- IDs in paths use the strongly-typed ID's URL form (Guid v7, hex without braces).

## HTTP methods and status codes

| Method | Success | Common failures |
|---|---|---|
| `GET /resources` | 200 with `PagedList<T>` | 400 (bad query) |
| `GET /resources/{id}` | 200 with DTO | 404, 410 (gone) |
| `POST /resources` | 201 with `Location` header + DTO | 400, 409 (duplicate), 422 |
| `PUT /resources/{id}` | 200 or 204 | 404, 409 (concurrency) |
| `PATCH /resources/{id}` | 200 or 204 | 400, 404, 409 |
| `DELETE /resources/{id}` | 204 | 404, 409 (cannot delete) |
| `POST /resources/{id}/<action>` | 200, 202 (async), or 204 | 404, 409 (illegal transition) |

- **Never** return 200 with `{ "success": false, ... }`. Use the proper status code.
- 422 is reserved for **semantic** validation failures distinct from syntactic 400 (rare; prefer 400 with the `errors` map).
- 202 Accepted requires a `Location` header pointing to a status resource.

## Pagination

- Cursor-based by default for collection endpoints:
  ```
  GET /api/v1/orders?cursor=eyJpZCI6...&limit=50
  ```
- Response shape:
  ```json
  {
    "items": [...],
    "nextCursor": "eyJpZCI6...",
    "hasMore": true
  }
  ```
- `limit` defaults to 50, max 200. Server clamps silently.
- Offset pagination only for back-office UIs that need page numbers — opt-in via `?page=1&pageSize=50`.

## Filtering and sorting

- Filtering: explicit query parameters (`?status=pending&customerId=...`). Avoid free-form OData / RSQL.
- Sorting: `?sort=createdAt:desc,total:asc`. Whitelist sortable fields per endpoint.

## Idempotency

- All `POST` endpoints that create or trigger side-effects accept an `Idempotency-Key` header.
- The Application layer stores `(idempotencyKey, requestHash, responseSnapshot)` for 24h. Replays return the cached response.
- `PUT`/`DELETE` are naturally idempotent; do not add the header.

## Concurrency control

- Mutating endpoints accept `If-Match` with the resource's `ETag` (the aggregate's row version).
- 412 Precondition Failed if the version does not match.
- `GET` responses include `ETag`.

## Versioning

- URL versioning: `/api/v{n}/...`. Major versions only.
- Breaking changes require a new version. Non-breaking additions go to the existing version.
- Deprecation: `Sunset` header (RFC 8594) and a deprecation entry in the OpenAPI description.

## Error contract — `ProblemDetails` (RFC 7807)

```json
{
  "type": "https://api.example.com/problems/validation",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "detail": "See 'errors' for field-level messages.",
  "instance": "/api/v1/orders",
  "traceId": "00-...-...-01",
  "errors": {
    "customerId": ["Required."],
    "items[0].quantity": ["Must be greater than zero."]
  }
}
```

- `type` is a stable URI per error category, not a generated GUID.
- `traceId` always populated from the active `Activity`.
- For domain errors: include the error code in `extensions.code` (e.g. `"ORDER.ALREADY_SHIPPED"`).

## Content negotiation

- JSON only by default. `application/json; charset=utf-8`.
- Property names: **camelCase** (`PropertyNamingPolicy = JsonNamingPolicy.CamelCase`).
- Enums: serialize as **strings** (`JsonStringEnumConverter`), never integers, in API responses.
- Dates: ISO 8601 UTC (`2026-04-26T10:00:00Z`). Never local time.
- Money: `{ "amount": "10.50", "currency": "USD" }` — string for amount to preserve precision.

## OpenAPI

- Every endpoint declares: name, summary, tags, request/response types, all status codes including problem types.
- Examples on every request/response via `[OpenApiExample]` or `WithOpenApi(op => op.Examples = ...)`.
- Group operations by feature tag, not by HTTP method.

## HATEOAS

- Not used by default. Add hypermedia links only on a per-endpoint basis when there is a clear consumer benefit (workflow APIs).

## Caching

- `GET` endpoints with stable representations include `Cache-Control` and `ETag`.
- Write endpoints respond with `Cache-Control: no-store`.

## Security headers (set by middleware, not endpoint)

- `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Content-Security-Policy` for any HTML responses.
