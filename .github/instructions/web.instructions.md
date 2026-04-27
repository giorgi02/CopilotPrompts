---
name: 'Web layer (ASP.NET Core) rules'
description: 'Minimal API endpoints, ProblemDetails, OpenAPI, middleware, and composition in the Web project.'
applyTo: 'src/Web/**/*.cs'
---

# Web layer rules

The Web layer is the HTTP boundary. Its job: deserialize, dispatch via MediatR, serialize the result. Nothing more.

## Hard constraints

- Endpoints are **Minimal APIs** organized as `IEndpoint` modules per feature.
- Endpoints are **handler-thin** — typically 5–15 lines. Logic belongs in Application.
- No business rules in endpoints. No DB calls in endpoints. No domain types serialized to the wire — always DTOs.
- Composition root: `Program.cs` calls `builder.Services.AddDomain().AddApplication().AddInfrastructure(builder.Configuration).AddWeb();`.

## Endpoint module pattern

```csharp
public sealed class PlaceOrderEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapPost("/api/v1/orders", async (
                PlaceOrderRequest request,
                ISender sender,
                CancellationToken ct) =>
            {
                var result = await sender.Send(request.ToCommand(), ct);
                return result.ToHttpResult();
            })
            .WithName("PlaceOrder")
            .WithTags("Orders")
            .Produces<OrderDto>(StatusCodes.Status201Created)
            .ProducesProblem(StatusCodes.Status400BadRequest)
            .ProducesProblem(StatusCodes.Status409Conflict)
            .RequireAuthorization("orders:write");
}
```

- One endpoint per file. File name = endpoint name + `Endpoint`.
- Group endpoints under feature folders mirroring Application: `Endpoints/Orders/PlaceOrderEndpoint.cs`.
- Use `RouteGroupBuilder` to share prefixes/auth/tags within a feature.

## Versioning

- URL versioning: `/api/v{version:apiVersion}/...` via `Asp.Versioning.Http`.
- Default version 1.0. Mark deprecated versions explicitly. Sunset header on deprecated routes.

## Request / Response shapes

- Request DTOs are records suffixed `Request`, defined in the Web project next to the endpoint.
- Response DTOs come from the Application layer (`OrderDto`).
- Never expose Domain entities directly.

## ProblemDetails (RFC 7807)

- Configure `AddProblemDetails()` once in `Program.cs`.
- Map `Result.Error` codes to HTTP status via a `ResultExtensions.ToHttpResult()` extension:
  - `Validation` → 400 with errors dictionary
  - `NotFound` → 404
  - `Conflict` → 409
  - `Unauthorized` → 401 (typically handled by auth middleware)
  - `Forbidden` → 403
  - `Failure` → 500
- Never include exception messages or stack traces in production responses.

## OpenAPI

- `Microsoft.AspNetCore.OpenApi` (built-in) for spec generation.
- **Scalar** (`Scalar.AspNetCore`) for the docs UI in Development.
- Every endpoint declares `.Produces<T>(...)`, `.ProducesProblem(...)`, and `.WithDescription(...)`.
- Operation IDs are explicit and stable: `.WithName("PlaceOrder")`.

## Middleware order

```
UseExceptionHandler → UseStatusCodePages → UseHsts (prod)
→ UseHttpsRedirection → UseSerilogRequestLogging
→ UseRouting → UseRateLimiter → UseCors
→ UseAuthentication → UseAuthorization
→ UseOutputCache → MapEndpoints
```

## Authentication / Authorization

- JWT Bearer or OpenID Connect. **Never** custom auth schemes.
- Authorization is policy-based: `RequireAuthorization("orders:write")`.
- Policies live in `Web/Authorization/AuthorizationPolicies.cs`.
- Resource-based authorization uses `IAuthorizationService` evaluated inside the Application handler when the rule depends on data.

## Rate limiting

- `AddRateLimiter` configured globally (token bucket per IP) and per-endpoint where stricter limits apply (`/auth/login`, `/auth/refresh`).

## CORS

- Explicit allow-list. **Never** `AllowAnyOrigin()` in production.

## Health checks

- `/health/live` — process is up.
- `/health/ready` — DB, cache, broker reachable. Returns 503 if any dependency down.
- Excluded from auth.

## Forbidden

- Calling `DbContext` from an endpoint.
- Returning anonymous types.
- `[ApiController]` + controllers — we use Minimal APIs only.
- `app.MapControllers()` unless a controller is explicitly justified in the PR for a specific case (e.g. file streaming with custom binders).
- Mixing concerns in `Program.cs` — extension methods only.
