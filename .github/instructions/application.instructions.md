---
name: 'Application layer rules'
description: 'Use cases, CQRS handlers, validators, pipeline behaviors, and DTOs for the Application project.'
applyTo: 'src/Application/**/*.cs'
---

# Application layer rules

The Application layer hosts **use cases**. Each use case is a vertical slice: command/query + handler + validator + DTO + (optional) authorization rule.

## Hard constraints

- Depends on **Domain only** and on **abstractions** that Infrastructure implements (`IApplicationDbContext`, `IEmailSender`, `ICurrentUser`, etc.).
- **No EF Core**, no `Npgsql`, no `HttpClient`, no `IHttpContextAccessor` references — those are Infrastructure or Web concerns.
- No business rules. Business rules live in Domain. The Application layer **orchestrates**.
- Handlers are **thin**: load aggregate(s) → call domain methods → persist → return Result.

## Folder layout — vertical slice per use case

```
src/Application/Features/Orders/PlaceOrder/
  PlaceOrderCommand.cs         ← record : IRequest<Result<OrderDto>>
  PlaceOrderHandler.cs         ← IRequestHandler<PlaceOrderCommand, Result<OrderDto>>
  PlaceOrderValidator.cs       ← AbstractValidator<PlaceOrderCommand>
  PlaceOrderAuthorization.cs   ← optional, IAuthorizationRequirement
  OrderDto.cs                  ← response shape
```

One folder per use case. Do **not** colocate read/write handlers in the same folder.

## CQRS via MediatR

- Commands: `record FooCommand(...) : IRequest<Result>` or `IRequest<Result<T>>`. Mutate state.
- Queries: `record FooQuery(...) : IRequest<Result<TDto>>`. Side-effect free.
- Handlers: one handler per command/query. **Never share** a handler across requests.
- Handlers are `internal sealed` — they are wired by MediatR, not consumed directly.
- Constructor injection only. Primary constructors preferred.

## Result pattern

- All handlers return `Result` or `Result<T>`. Never raw `T`. Never throw for expected outcomes.
- Errors come from the static catalogs in Domain (`OrderErrors.NotFound`).
- Map errors to HTTP status at the Web boundary, not in the handler.

## Validation

- Every command/query that takes user input has a `FluentValidation` validator.
- Validation runs in the MediatR `ValidationBehavior` pipeline before the handler executes.
- Validators check **shape and basic invariants** (required, length, format). Domain rules stay in Domain.
- A failing validator short-circuits with `Result.Failure(Error.Validation(...))`.

## Pipeline behaviors

In execution order:
1. `LoggingBehavior` — request name + correlation id, no payload (PII risk).
2. `AuthorizationBehavior` — checks `IAuthorizationRequirement` markers on the request.
3. `ValidationBehavior` — runs FluentValidation.
4. `UnitOfWorkBehavior` — wraps commands in a transaction, calls `SaveChangesAsync` on success.
5. `CachingBehavior` — for queries marked `ICacheableQuery`.
6. `PerformanceBehavior` — logs warning if handler exceeds threshold (default 500ms).

## DTOs

- `record` types. Suffix with `Dto`, `Response`, or `Summary` based on shape.
- Mapped manually or via `Mapster`. **No AutoMapper.**
- DTOs do not contain Domain types. Convert at the handler boundary.

## Read queries — projection rules

- Use `IQueryable.Select(...)` to project directly to DTO. **Never** load the entity then map in memory.
- Use `AsNoTracking()` on every read query.
- Pagination: cursor-based by default (`PagedList<T>` with `NextCursor`); offset only for back-office UIs.

## Authorization

- Implement `IAuthorizationRequirement` on the request type or use a separate `XxxAuthorization` class.
- The `AuthorizationBehavior` resolves all `IAuthorizationRule<TRequest>` for a request and runs them.
- Rules return `Result.Success()` or `Result.Failure(Errors.Forbidden)`.

## Forbidden

- `IServiceProvider` or `IServiceScopeFactory` injected into a handler (service locator anti-pattern).
- Direct `DbContext` usage — use `IApplicationDbContext` or repositories.
- `async void`, blocking calls, fire-and-forget without `IBackgroundJobClient`.
- Side-effects in queries.
- Handlers calling other handlers via `IMediator.Send` from inside a handler — extract a domain service or compose at the Web layer.
- `try/catch` to convert exceptions to `Result` — let them bubble; the global handler logs and returns 500.

See [cqrs-implementation skill](../skills/cqrs-implementation/SKILL.md) for canonical templates.
