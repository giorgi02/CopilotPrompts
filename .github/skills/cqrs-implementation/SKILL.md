---
name: cqrs-implementation
description: Implement CQRS use cases via MediatR — commands, queries, handlers, validators, pipeline behaviors, Result pattern. Use when adding a new use case in the Application layer, refactoring a service into handlers, or wiring a new pipeline behavior.
---

# CQRS implementation

Activates when working in `src/Application/` on commands, queries, handlers, validators, or pipeline behaviors. This is the canonical reference for CQRS conventions — the layer rules, the file shapes, and the templates all live here.

## The split

- **Command** — mutates state. Returns `Result` or `Result<T>` where `T` is the new resource's identifier or a thin DTO.
- **Query** — read-only. Returns `Result<TDto>`. Never has side effects (cache reads/writes happen in `CachingBehavior`).

One handler per request. Never share handlers.

## Naming

- Commands: imperative, ends with `Command`. `PlaceOrderCommand`, `CancelOrderCommand`, `UpdateCustomerEmailCommand`.
- Queries: descriptive, ends with `Query`. `GetOrderByIdQuery`, `ListOrdersForCustomerQuery`, `SearchOrdersQuery`.
- Handlers: `<RequestName>Handler`. Always `internal sealed`.
- Validators: `<RequestName>Validator`. Always `internal sealed`.
- DTOs: `<Resource><Shape>` — `OrderDto`, `OrderSummary`, `OrderListItem`.

## Folder shape — vertical slice

```
src/Application/Features/<Feature>/<UseCase>/
  <UseCase>Command.cs            ← record : IRequest<Result<...>>
  <UseCase>Handler.cs            ← internal sealed
  <UseCase>Validator.cs          ← internal sealed : AbstractValidator<...>
  <UseCase>Authorization.cs      ← optional rule class
  <Resource>Dto.cs               ← only if not shared
```

## Request

```csharp
public sealed record PlaceOrderCommand(
    CustomerId CustomerId,
    IReadOnlyList<PlaceOrderItem> Items,
    string? IdempotencyKey
) : IRequest<Result<OrderId>>;

public sealed record PlaceOrderItem(ProductId ProductId, int Quantity);
```

- `record` types only.
- Strongly-typed IDs over `Guid`/`int`.
- `IReadOnlyList<T>` for collections.
- Nested types as records colocated in the same file.

## Handler

```csharp
internal sealed class PlaceOrderHandler(
    IApplicationDbContext db,
    IPricingCalculator pricing,
    TimeProvider clock,
    ILogger<PlaceOrderHandler> logger)
    : IRequestHandler<PlaceOrderCommand, Result<OrderId>>
{
    public async Task<Result<OrderId>> Handle(PlaceOrderCommand request, CancellationToken ct)
    {
        var customer = await db.Customers.FindAsync([request.CustomerId], ct);
        if (customer is null) return Result.Failure<OrderId>(CustomerErrors.NotFound);
        if (!customer.IsActive) return Result.Failure<OrderId>(CustomerErrors.Inactive);

        var pricedItems = await pricing.PriceAsync(request.Items, ct);
        var orderResult = Order.Place(customer.Id, pricedItems, clock.GetUtcNow());
        if (orderResult.IsFailure) return Result.Failure<OrderId>(orderResult.Error);

        db.Orders.Add(orderResult.Value);
        // SaveChangesAsync is called by UnitOfWorkBehavior on success.
        return Result.Success(orderResult.Value.Id);
    }
}
```

- `internal sealed` — wired by MediatR, not consumed directly.
- Primary constructor for DI.
- No `try/catch` for expected failures — return `Result.Failure`.
- No `SaveChangesAsync` inside command handlers — `UnitOfWorkBehavior` does it.
- No `IMediator.Send` from inside a handler. Orchestrate at the Web layer or via domain events.

## Validator

```csharp
internal sealed class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty().Must(items => items.Count <= 100)
            .WithMessage("An order can contain at most 100 items.");
        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.ProductId).NotEmpty();
            item.RuleFor(i => i.Quantity).InclusiveBetween(1, 1000);
        });
    }
}
```

- Validators check **shape** and **basic invariants**.
- Domain rules (e.g. "customer must be active") live in Domain or in the handler — not here.
- No I/O in validators (no DB lookups). Async rules only when a unique-constraint check is genuinely needed.

## Pipeline behaviors

Registered in this order in `Application/DependencyInjection.cs`:

```csharp
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblyContaining<IApplicationMarker>();
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
    cfg.AddOpenBehavior(typeof(AuthorizationBehavior<,>));
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
    cfg.AddOpenBehavior(typeof(UnitOfWorkBehavior<,>));
    cfg.AddOpenBehavior(typeof(CachingBehavior<,>));
    cfg.AddOpenBehavior(typeof(PerformanceBehavior<,>));
});
```

| Behavior | Applies to | Purpose |
|---|---|---|
| `LoggingBehavior` | All | Log request name + correlation id, no payload |
| `AuthorizationBehavior` | Marked with `IAuthorizationRequirement` | Run rule classes |
| `ValidationBehavior` | Has registered `IValidator<TRequest>` | Short-circuit on failure |
| `UnitOfWorkBehavior` | Commands | Transaction + `SaveChangesAsync` |
| `CachingBehavior` | Marked with `ICacheableQuery` | Read-through + tag invalidation |
| `PerformanceBehavior` | All | Warn if elapsed > threshold (default 500ms) |

## Authorization

- Mark request with `IAuthorizationRequirement`.
- Implement `IAuthorizationRule<TRequest>` for resource-based checks.
- Rules return `Result.Success()` or `Result.Failure(Errors.Forbidden)`.

## Caching

- Queries opt in via `ICacheableQuery` with `CacheKey`, `Expiration`, optional `Tags`.
- Commands opt in to invalidation via `IInvalidateCache` with the tags to evict.

## Domain event handlers

- Implement `INotificationHandler<TEvent>` in **Application**, not Domain.
- Handlers are independent — one event may have many handlers, none should depend on order.
- Long-running side effects (sending email, calling external systems) go via the **outbox** — handle the event by enqueueing an Infrastructure job, not by calling the external system synchronously.

## Tests for a handler

Cover, in order:
1. Happy path → `Result.Success`.
2. Each `Result.Failure` branch.
3. Cancellation: pass a cancelled token; assert `OperationCanceledException`.
4. (For aggregates) Domain events raised via `aggregate.PopUncommittedEvents()`.

## Forbidden

- Handlers that take `IServiceProvider` (service locator).
- `IMediator` injected anywhere except the Web layer.
- Returning entities from queries — always project to a DTO.
- Validators with DB calls (except the rare unique-constraint case).
- Splitting one logical use case across multiple handlers.

## Output when generating

```
Use case: <Name>
Type: <command | query>
Returns: <Result<T> | Result>
Failure modes:
  - <ErrorCode> — <when>
Files:
  - src/Application/Features/<F>/<U>/<U>Command.cs
  - src/Application/Features/<F>/<U>/<U>Handler.cs
  - src/Application/Features/<F>/<U>/<U>Validator.cs
  - tests/Application.UnitTests/Features/<F>/<U>HandlerTests.cs
```
