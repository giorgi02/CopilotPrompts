---
name: 'Presentation (Web) layer rules'
description: 'Web project structure, Minimal API endpoint modules, middleware pipeline, Result-to-ProblemDetails mapping, and composition-root discipline.'
applyTo: 'src/Web/**/*.cs'
---

# Presentation (Web) layer rules

The Web project is the **composition root** and HTTP boundary. It translates HTTP into MediatR requests, maps `Result` outcomes to status codes, and wires every cross-cutting concern. It contains **no business logic** and **no persistence code**.

For HTTP contract conventions (URLs, status codes, pagination, ProblemDetails shape) see [api.instructions.md](./api.instructions.md). This file governs **how the Web project is organized**.

## Hard constraints

- Web depends on **Application**. It depends on **Infrastructure only inside `Program.cs`** for DI wiring — never from an endpoint, filter, or service.
- No `DbContext`, `IApplicationDbContext`, repository, or `Npgsql` type ever appears in `src/Web` outside `Program.cs`.
- No business rules. Endpoints **dispatch** to MediatR and **shape** the HTTP response. That is the entire job.
- No custom controller bases. **Minimal APIs** only.
- No `IHttpContextAccessor` reaching into `HttpContext` for business state — wrap accessors behind `ICurrentUser` (Application abstraction, Infrastructure-implemented).

## Project layout

```
src/Web/
  Endpoints/
    Orders/
      PlaceOrderEndpoint.cs        ← IEndpoint, one file per route
      CancelOrderEndpoint.cs
      GetOrderByIdEndpoint.cs
  Middleware/
    RequestLoggingMiddleware.cs
    CorrelationIdMiddleware.cs
  Filters/
    IdempotencyEndpointFilter.cs
    ValidationEndpointFilter.cs    ← only if not handled by MediatR ValidationBehavior
  ExceptionHandlers/
    GlobalExceptionHandler.cs      ← IExceptionHandler producing ProblemDetails
  Infrastructure/                  ← Web-only infra: ResultExtensions, OpenApi config
    ResultExtensions.cs
    OpenApiTransformers.cs
  Extensions/
    DependencyInjection.cs         ← AddWebServices()
    ApplicationBuilderExtensions.cs
  Program.cs
```

One endpoint per file. Group by feature folder, not by HTTP verb.

## Endpoint module pattern

Every endpoint implements `IEndpoint` and is auto-discovered:

```csharp
internal sealed class PlaceOrderEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/v1/orders", Handle)
            .WithName(nameof(PlaceOrderEndpoint))
            .WithTags("Orders")
            .Produces<OrderDto>(StatusCodes.Status201Created)
            .ProducesValidationProblem()
            .ProducesProblem(StatusCodes.Status409Conflict)
            .RequireAuthorization()
            .AddEndpointFilter<IdempotencyEndpointFilter>();
    }

    private static async Task<IResult> Handle(
        PlaceOrderRequest request,
        ISender sender,
        CancellationToken ct)
    {
        var command = request.ToCommand();
        var result = await sender.Send(command, ct);
        return result.Match(
            order => Results.Created($"/api/v1/orders/{order.Id}", order),
            ResultExtensions.Problem);
    }
}
```

- `internal sealed`. Endpoints are wired by reflection, not consumed directly.
- Static `Handle` method — no instance state on the endpoint type.
- `CancellationToken` parameter on every async handler. Always passed to `ISender.Send`.

## Endpoint discovery

One discovery extension scans the assembly and maps every `IEndpoint`:

```csharp
public static IEndpointRouteBuilder MapEndpoints(this IEndpointRouteBuilder app)
{
    var endpoints = typeof(IWebAssemblyMarker).Assembly
        .GetTypes()
        .Where(t => t is { IsClass: true, IsAbstract: false } && typeof(IEndpoint).IsAssignableFrom(t));

    foreach (var type in endpoints)
        ((IEndpoint)Activator.CreateInstance(type)!).MapEndpoint(app);

    return app;
}
```

Called once from `Program.cs`: `app.MapEndpoints();`. Do not register endpoints individually in `Program.cs`.

## Request and response contracts

- **Request DTOs** (the HTTP body shape) live next to the endpoint: `PlaceOrderRequest` in the same file or a sibling.
- **Response DTOs** are the Application-layer DTOs (`OrderDto`) — the Web layer does not redefine them.
- Mapping HTTP request → MediatR command happens via a `ToCommand()` method on the request type. Keep it pure.
- `record` types only. `JsonStringEnumConverter` enforced globally. camelCase property names.

## Result → IResult mapping

A single `ResultExtensions` translates `Result`/`Result<T>` into HTTP. **No `if (result.IsFailure) return BadRequest(...)`** scattered across endpoints.

```csharp
public static IResult Problem(Result result) => result.Error.Type switch
{
    ErrorType.Validation   => Results.ValidationProblem(result.Error.ToDictionary()),
    ErrorType.NotFound     => Results.Problem(statusCode: StatusCodes.Status404NotFound, ...),
    ErrorType.Conflict     => Results.Problem(statusCode: StatusCodes.Status409Conflict, ...),
    ErrorType.Unauthorized => Results.Problem(statusCode: StatusCodes.Status401Unauthorized, ...),
    ErrorType.Forbidden    => Results.Problem(statusCode: StatusCodes.Status403Forbidden, ...),
    _                      => Results.Problem(statusCode: StatusCodes.Status500InternalServerError)
};
```

Always include `traceId` from the active `Activity`, the stable `type` URI per error category, and `extensions.code` for domain error codes (e.g. `"ORDER.ALREADY_SHIPPED"`).

## Middleware pipeline order

In `Program.cs`, fixed order:

1. `UseExceptionHandler()` — registered `IExceptionHandler` produces ProblemDetails. **Never leak exception text.**
2. `UseHsts()` / `UseHttpsRedirection()` (non-Development).
3. `UseCors()` — named policy from configuration; no `AllowAnyOrigin`.
4. `CorrelationIdMiddleware` — reads/sets `X-Correlation-Id`, attaches to `Activity.Current`.
5. `UseSerilogRequestLogging()` — structured request log; PII redacted via destructuring policies.
6. `UseRateLimiter()` — fixed/sliding window per route group as configured.
7. `UseAuthentication()` → `UseAuthorization()`.
8. `MapEndpoints()`.
9. `MapHealthChecks("/health/live")`, `MapHealthChecks("/health/ready", { Predicate = r => r.Tags.Contains("ready") })`.

## Authentication and authorization

- **JWT bearer** via `AddAuthentication().AddJwtBearer()`. Authority + audience from `IOptions<JwtOptions>` with validation.
- Policy-based authorization. Policies defined in `Extensions/AuthorizationPolicies.cs` and registered via `AddAuthorization(options => ...)`.
- Per-endpoint: `.RequireAuthorization("PolicyName")`. Anonymous endpoints opt in explicitly via `.AllowAnonymous()`.
- Domain-level rules (e.g. "user owns this order") run as `IAuthorizationRule<TRequest>` in the Application `AuthorizationBehavior` — **not** as ASP.NET policy handlers.

## Composition root — `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddDomain()
    .AddApplication()
    .AddInfrastructure(builder.Configuration)
    .AddWebServices(builder.Configuration);

var app = builder.Build();
app.UsePipeline();      // applies the middleware order above
app.MapEndpoints();
app.Run();
```

- One `AddXxx` extension per layer. Web-specific service registration lives in `Extensions/DependencyInjection.cs`.
- `Program.cs` is the **only** place Web references Infrastructure.
- Configuration binding via `IOptions<T>` with `ValidateOnStart()` + `ValidateDataAnnotations()`. Fail fast on misconfiguration.

## Idempotency

- `IdempotencyEndpointFilter` reads the `Idempotency-Key` header on `POST` endpoints that mark themselves with `.AddEndpointFilter<IdempotencyEndpointFilter>()`.
- The filter delegates to `IIdempotencyStore` (Application abstraction). On replay, returns the stored response without invoking the handler.
- Missing key on a filtered endpoint → 400 with the `idempotency-key-required` problem type.

## OpenAPI

- `AddOpenApi()` (built-in) + `Microsoft.AspNetCore.OpenApi` transformers in `Infrastructure/OpenApiTransformers.cs`.
- Every endpoint declares `Produces<T>`, every relevant `ProducesProblem(...)`, `WithName`, `WithTags`, `WithSummary`, `WithDescription`.
- Examples on each request/response.
- Group operations by feature tag. Group by HTTP method is forbidden.

## Health checks

- `/health/live` — liveness, no dependencies. Returns 200 if the process is up.
- `/health/ready` — readiness, includes Postgres and Redis pings tagged `"ready"`.
- Health responses use the JSON writer; never expose connection strings or stack traces.

## Forbidden

- Business logic, validation rules, or domain decisions inside an endpoint handler. The handler dispatches and shapes — that is all.
- `DbContext`, `IApplicationDbContext`, repositories, or `NpgsqlDataSource` referenced anywhere in `src/Web` outside `Program.cs`.
- `[ApiController]` / `ControllerBase` subclasses. Minimal APIs only.
- Catching `Exception` in an endpoint to convert it to a response — let the global `IExceptionHandler` do it.
- `HttpContext.Items` for cross-handler state — use `ICurrentUser` / scoped services.
- `Results.Ok(new { success = false, ... })` or any 200-with-error envelope. Use the correct status code via `ResultExtensions.Problem`.
- Raw exception text, stack traces, or framework messages in any HTTP response body.
- Endpoint-specific DI registrations scattered across files — all Web service wiring goes through `AddWebServices`.
- Synchronous I/O (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) anywhere in the request path.
