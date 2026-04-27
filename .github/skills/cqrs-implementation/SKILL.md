---
name: cqrs-implementation
description: Implement CQRS use cases via MediatR — commands, queries, handlers, validators, pipeline behaviors, Result pattern. Use when adding a new use case in the Application layer, refactoring a service into handlers, or wiring a new pipeline behavior.
---

# CQRS implementation

Activates when working in `src/Application/` on commands, queries, handlers, validators, or pipeline behaviors.

## The split

- **Command** — mutates state. Returns `Result` or `Result<T>` where `T` is the new ID or a thin DTO. Names: imperative verb + `Command`.
- **Query** — reads. Returns `Result<TDto>`. No side effects (cache reads/writes happen in `CachingBehavior`). Names: `Get`/`List`/`Search` + `Query`.

One handler per request. Never share.

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

## Handler

```csharp
internal sealed class PlaceOrderHandler(
    IApplicationDbContext db,
    IPricingCalculator pricing,
    TimeProvider clock)
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
        return Result.Success(orderResult.Value.Id);
    }
}
```

- `internal sealed` — wired by MediatR, not consumed directly.
- Primary constructor for DI.
- No `try/catch` for expected failures — `Result.Failure`.
- No `SaveChangesAsync` — `UnitOfWorkBehavior` does it.
- No `IMediator.Send` from a handler — orchestrate at Web or via domain events.

## Validator

```csharp
internal sealed class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty().Must(i => i.Count <= 100);
        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.ProductId).NotEmpty();
            item.RuleFor(i => i.Quantity).InclusiveBetween(1, 1000);
        });
    }
}
```

- Shape and basic invariants only. Domain rules live in Domain.
- No I/O. Async only when a unique-constraint check is genuinely required.

## Pipeline behaviors (registered in order)

1. `LoggingBehavior` — request name + correlation, no payload.
2. `AuthorizationBehavior` — runs `IAuthorizationRule<TRequest>` instances.
3. `ValidationBehavior` — runs FluentValidation; short-circuits on failure.
4. `UnitOfWorkBehavior` — transaction + `SaveChangesAsync` for commands.
5. `CachingBehavior` — read-through and tag invalidation for `ICacheableQuery`.
6. `PerformanceBehavior` — warns over threshold (default 500ms).

## Authorization

- Mark request with `IAuthorizationRequirement`.
- Implement `IAuthorizationRule<TRequest>` for resource-based checks.
- Rules return `Result.Success()` or `Result.Failure(Errors.Forbidden)`.

## Caching

- `ICacheableQuery` exposes `CacheKey`, `Expiration`, `Tags`.
- Commands implement `IInvalidateCache` to evict tags.

## Tests for a handler

Cover, in order:
1. Happy path → `Result.Success`.
2. Each `Result.Failure` branch.
3. Cancellation: pass cancelled token; assert `OperationCanceledException`.
4. (For aggregates) Domain events raised via `aggregate.PopUncommittedEvents()`.

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

See [cqrs instructions](../../instructions/cqrs.instructions.md) for the layer rules.
