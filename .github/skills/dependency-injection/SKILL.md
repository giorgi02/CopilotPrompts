---
name: dependency-injection
description: Service registration, lifetimes, composition root, and the options pattern for the ASP.NET Core 9 stack. Use when editing `Program.cs`, any `DependencyInjection.cs` / `*Module.cs` extension, choosing a service lifetime, introducing an interface, or wiring a typed `HttpClient` / `IOptions<T>` / hosted service.
---

# Dependency injection

Activates when wiring services anywhere in `src/` — composition root, per-layer `Add<Layer>` extensions, or per-feature `*Module.cs` extensions. This is the canonical reference for lifetimes, registration style, and forbidden patterns.

## Composition root

- The single composition root is `src/Web/Program.cs`. It calls layer extensions in order:

  ```csharp
  builder.Services
      .AddDomain()
      .AddApplication()
      .AddInfrastructure(builder.Configuration)
      .AddWeb(builder.Configuration);
  ```

- Each layer exposes **one** `Add<Layer>(...)` extension in `<Layer>/DependencyInjection.cs`.
- Per-feature wiring inside a layer goes into `*Module.cs` extensions called from the layer's `Add<Layer>`.
- `Program.cs` does composition only — no business logic, no inline registrations beyond the layer calls and ASP.NET Core middleware pipeline.

## File shape

```
src/<Layer>/
  DependencyInjection.cs         ← Add<Layer>(IServiceCollection, IConfiguration?)
  Features/<Feature>/<Feature>Module.cs   ← Add<Feature>(IServiceCollection)
```

`Add<Layer>` calls each `Add<Feature>` in turn. Module extensions are `internal static` — they are not part of the public API of the layer.

## Lifetimes — defaults

| Type | Lifetime |
|---|---|
| Stateless services (handlers, validators, calculators) | Scoped |
| Repositories, `DbContext`, `IUnitOfWork` | Scoped |
| `HttpClient` typed clients | Scoped (factory-managed) |
| `IDistributedCache`, `IConnectionMultiplexer` (Redis), `NpgsqlDataSource` | Singleton |
| `TimeProvider`, `ILogger<>`, `IOptions<T>` | Singleton |
| `BackgroundService`, hosted services | Singleton |
| `MediatR` and pipeline behaviors | Scoped (default) |

Choose the **shortest** lifetime that still allows the service to do its job. When in doubt, scoped.

## Registration style

- Use `services.AddScoped<TInterface, TImpl>()`. Avoid `services.AddScoped(typeof(TImpl))` unless the consumer takes the concrete type by design.
- Group registrations by **bounded context / module**, not by lifetime.
- One registration per service. **No** auto-registration via assembly scanning. We accept the verbosity in exchange for clarity.
- Typed `HttpClient` via `services.AddHttpClient<TClient>(...)`. Never `new HttpClient()`.

## Interfaces — when to introduce them

- Introduce an interface when there is a **second implementation** (test fake, alternative provider) or when the abstraction lives in a different layer than the implementation (Application defines, Infrastructure implements).
- Do not introduce `IFooService` for a single concrete `FooService` consumed only within the same layer.

## Options pattern

```csharp
public sealed class StripeOptions
{
    public const string SectionName = "Stripe";

    [Required] public required string ApiKey { get; init; }
    [Required, Url] public required string WebhookEndpoint { get; init; }
    public TimeSpan TimeoutSeconds { get; init; } = TimeSpan.FromSeconds(30);
}

services.AddOptions<StripeOptions>()
    .Bind(configuration.GetSection(StripeOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

- Always `ValidateOnStart()`. Bad config crashes the app at boot, not in production at request time.
- Inject `IOptions<T>` for static config, `IOptionsSnapshot<T>` for per-request, `IOptionsMonitor<T>` for hot-reload (rare).
- One options class per configuration section. Always set `SectionName` as a `const string`.

## Singletons that need scoped work

A singleton (e.g. `BackgroundService`, hosted job) that needs scoped services creates a scope per unit of work:

```csharp
internal sealed class OutboxProcessor(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var dispatcher = scope.ServiceProvider.GetRequiredService<IOutboxDispatcher>();
            await dispatcher.DispatchPendingAsync(ct);
        }
    }
}
```

Never capture a scoped service in a singleton's constructor.

## Forbidden

- `IServiceProvider` injected into business code (service locator).
- `IServiceScopeFactory.CreateScope()` outside `BackgroundService` / hosted scenarios.
- `static` mutable state.
- Singleton holding scoped dependencies (use `IServiceScopeFactory` per the pattern above).
- Registering open generics for cross-cutting concerns without explicit team agreement (we already have MediatR pipeline behaviors).
- Calling `BuildServiceProvider()` more than once during startup.
- Inline `new HttpClient()` — always use `IHttpClientFactory` or a typed client.
