---
name: 'Infrastructure layer rules'
description: 'EF Core configurations, external service adapters, caching, messaging, and DI registration for Infrastructure.'
applyTo: 'src/Infrastructure/**/*.cs'
---

# Infrastructure layer rules

The Infrastructure layer **implements** abstractions defined in Application. It is the only layer that touches I/O, third-party SDKs, and the database.

## Hard constraints

- Implements interfaces defined in **Application** or **Domain**. Never invents its own contracts that Application has to import.
- Never references the **Web** project. Wiring happens in Web's `Program.cs` via an `AddInfrastructure(IConfiguration)` extension exposed by this project.
- All registrations live in `Infrastructure/DependencyInjection.cs` and per-feature `*Module.cs` files.

## EF Core

- One `ApplicationDbContext : DbContext` implementing `IApplicationDbContext` (defined in Application).
- `DbContext` lifetime: scoped (per request).
- **No** entity attributes. Use `IEntityTypeConfiguration<T>` per aggregate under `Persistence/Configurations/`.
- Configure value objects with `OwnsOne` or value converters. Never expose value object internals to the model.
- Strongly-typed IDs use a `ValueConverter` defined once per ID type.
- `SaveChangesAsync` is overridden to:
  1. Apply audit fields (`CreatedAt`, `UpdatedAt`, `CreatedBy`, `UpdatedBy`) using `TimeProvider` and `ICurrentUser`.
  2. Collect domain events and write to the **outbox** table within the same transaction.
- Outbox is processed by a `BackgroundService` that publishes events via MediatR.

## Migrations

- Generated with `dotnet ef migrations add <Name> --project src/Infrastructure --startup-project src/Web --output-dir Persistence/Migrations`.
- Never edited by hand except to remove generated junk (e.g. unnecessary `AlterColumn` round-trips).
- Each migration is **forward-only**. Drops require explicit justification in the PR description.
- Migrations are applied at startup in non-prod via `db.Database.MigrateAsync()`. Production uses a separate migration step in CI/CD.

## Repositories

- Used **only** for aggregates with non-trivial loading (multiple includes, specifications). Otherwise, `IApplicationDbContext.Orders` is sufficient.
- One repository per aggregate, not per entity.
- Methods return aggregates, not `IQueryable`.

## Caching

- `HybridCache` (default) for read-through caching. Tag-based invalidation.
- `IDistributedCache` (Redis) for cross-instance state (rate limit counters, session-like data).
- Cache keys: `"<area>:<entity>:<id>"`, lower kebab-case. Centralize in a `CacheKeys` static class.
- TTLs are explicit. Never cache without a TTL.

## External services

- HTTP clients via `IHttpClientFactory` and **typed clients**. Never `new HttpClient()`.
- Resilience via **Polly v8** pipelines: timeout → retry (with jitter) → circuit breaker, registered via `AddResilienceHandler`.
- Each external client implements an Application-defined interface (`IPaymentGateway`, `IEmailSender`).
- Secrets via `IConfiguration` bound to strongly-typed options validated with `ValidateDataAnnotations` and `ValidateOnStart`.

## Messaging

- Outbox-first. Domain events → outbox → publisher → broker.
- For inbound messages: idempotency keys stored in an `inbox` table; reject duplicates.
- Use **MassTransit** or raw broker SDK — pick one team-wide and do not mix in the same project.

## Background jobs

- `Hangfire` for cron and recurring jobs. Job classes implement `IJob` with `ExecuteAsync(CancellationToken)`.
- In-process queues use `Channel<T>` + a single `BackgroundService` consumer.
- Long-running work never runs inside an HTTP request handler.

## Forbidden

- Returning `IQueryable` from repositories.
- Catching DbUpdateException to translate it — let the global handler convert it; surface conflicts via aggregate methods returning `Result`.
- Direct calls to `Console.*`, `Environment.GetEnvironmentVariable` (use `IConfiguration`).
- Hardcoded connection strings, keys, URLs. Always via options.
- `static` mutable state.

## File layout

```
src/Infrastructure/
  DependencyInjection.cs
  Persistence/
    ApplicationDbContext.cs
    Configurations/
      OrderConfiguration.cs
    Repositories/
    Outbox/
    Migrations/
  Caching/
  Messaging/
  External/
    Stripe/
    SendGrid/
  BackgroundJobs/
  Identity/
```
