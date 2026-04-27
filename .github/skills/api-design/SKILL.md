---
name: api-design
description: REST API design — resources, status codes, pagination, idempotency, versioning, and OpenAPI metadata. Use when adding or reviewing HTTP endpoints, designing a public API surface, or auditing for consistency.
---

# REST API design

This skill activates when designing or reviewing HTTP endpoints. It complements [.github/instructions/api.instructions.md](../../instructions/api.instructions.md) and [.github/instructions/web.instructions.md](../../instructions/web.instructions.md).

## Resources

- Plural lower-kebab-case nouns: `/api/v1/orders`, `/api/v1/customers/{customerId}/addresses`.
- Nest at most one level. Beyond that, prefer top-level resources or query parameters.
- State-transition operations as `POST /api/v1/orders/{id}/cancel` — reserved for actions that don't map to CRUD.

## Method-status matrix

| Method | Success | Failure modes |
|---|---|---|
| `GET /resources` | 200 | 400 |
| `GET /resources/{id}` | 200 | 404, 410 |
| `POST /resources` | 201 + `Location` header | 400, 409, 422 |
| `PUT /resources/{id}` | 200 / 204 | 404, 409 |
| `PATCH /resources/{id}` | 200 / 204 | 400, 404, 409 |
| `DELETE /resources/{id}` | 204 | 404, 409 |
| `POST /resources/{id}/<action>` | 200 / 202 / 204 | 404, 409 |

- 200 with `{ "success": false }` is **forbidden** — use the proper status code.
- 422 reserved for semantic validation distinct from syntactic 400 (rare).
- 202 implies async; include a `Location` to a status resource.

## Pagination — cursor by default

```http
GET /api/v1/orders?cursor=eyJpZCI6...&limit=50

200 OK
{
  "items": [...],
  "nextCursor": "eyJpZCI6...",
  "hasMore": true
}
```

- `limit` default 50, cap 200 server-side.
- Offset pagination only for back-office UIs that need page numbers; opt-in via `?page=&pageSize=`.

## Idempotency

- `POST` with side-effects accepts `Idempotency-Key` header.
- Application stores `(key, requestHash, responseSnapshot)` for 24h. Replays return the cached response.
- `PUT`/`DELETE` are idempotent by definition — no header.

## Concurrency

- `GET` returns `ETag`.
- Mutating endpoints accept `If-Match`. 412 on mismatch.

## Errors — `ProblemDetails` (RFC 7807)

```json
{
  "type": "https://api.example.com/problems/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "See 'errors' for field-level messages.",
  "instance": "/api/v1/orders",
  "traceId": "00-...-...-01",
  "extensions": { "code": "ORDER.INVALID_QUANTITY" },
  "errors": { "items[0].quantity": ["Must be > 0"] }
}
```

- `type` is a stable URI per error category.
- `traceId` from the active `Activity`.
- Domain errors expose code in `extensions.code`.

## Versioning

- URL versioning, major only: `/api/v1/...`, `/api/v2/...`.
- Breaking change → new version. Additive changes → existing version.
- Deprecation: `Sunset` header + OpenAPI deprecation flag.

## Content

- `application/json; charset=utf-8` only by default.
- Property names camelCase.
- Enums as strings.
- Dates as ISO 8601 UTC (`2026-04-26T10:00:00Z`).
- Money: `{ "amount": "10.50", "currency": "USD" }` (string amount preserves precision).

## OpenAPI metadata — every endpoint

```csharp
.WithName("PlaceOrder")
.WithTags("Orders")
.WithSummary("Place a new order for the authenticated customer")
.WithDescription("Validates inventory, applies pricing, and emits OrderPlaced.")
.Produces<OrderDto>(StatusCodes.Status201Created)
.ProducesProblem(StatusCodes.Status400BadRequest)
.ProducesProblem(StatusCodes.Status409Conflict)
.RequireAuthorization("orders:write");
```

- Operation IDs are explicit and stable.
- All success and error statuses declared.
- Examples on request/response where helpful.

## Design checklist (apply mentally to every new endpoint)

- [ ] Resource modeled as a noun, not an action (or it's a clear state transition).
- [ ] Verb is the right HTTP method.
- [ ] All success and failure status codes defined and documented.
- [ ] Authorization policy applied.
- [ ] If `POST` with side-effects → idempotency considered.
- [ ] Mutating endpoints concurrency-aware (`If-Match`).
- [ ] Pagination if the response is a collection.
- [ ] OpenAPI metadata complete.
- [ ] Versioned URL.

## Output when designing an endpoint

```
Resource: <noun>
Operation: <verb + state transition or CRUD>
Method + route: <METHOD /api/v1/...>
Success status: <code>
Failure modes (status → cause):
  - 400 — validation
  - 404 — not found
  - 409 — <conflict reason>
Authorization: <policy>
Idempotency: <yes — header X | no>
Concurrency: <yes — ETag | no>
Pagination: <cursor | offset | n/a>
Request DTO: <name + fields>
Response DTO: <name + fields>
```
