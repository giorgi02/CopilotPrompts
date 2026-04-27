---
name: 'Dependency injection conventions'
description: 'Service registration, lifetimes, options pattern, and composition root rules.'
applyTo: 'src/**/DependencyInjection.cs,src/Web/Program.cs,src/**/*Module.cs'
---

# Dependency injection conventions

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
- Group by **bounded context / module**, not by lifetime.
- One registration per service. **No** auto-registration via assembly scanning. We accept the verbosity in exchange for clarity.

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

## Forbidden

- `IServiceProvider` injected into business code (service locator).
- `IServiceScopeFactory.CreateScope()` outside `BackgroundService`/hosted scenarios.
- `static` mutable state.
- Singleton holding scoped dependencies (use a factory or `IServiceScopeFactory`).
- Registering open generics for cross-cutting concerns without explicit team agreement (we already have MediatR pipeline behaviors).
- Calling `BuildServiceProvider()` more than once during startup.
