---
name: 'CQRS / MediatR conventions'
description: 'Command, query, handler, and pipeline behavior conventions for MediatR-based use cases.'
applyTo: 'src/Application/**/*Command*.cs,src/Application/**/*Query*.cs,src/Application/**/*Handler*.cs,src/Application/**/*Behavior*.cs'
---

# CQRS / MediatR conventions

## The split

- **Command** = mutates state. Returns `Result` or `Result<T>` where `T` is the new resource's identifier or a thin DTO.
- **Query** = read-only. Returns `Result<TDto>`. Never has side effects (not even cache writes — those happen in `CachingBehavior`).
- One handler per request. Never share handlers.

## Naming

- Commands: imperative, ends with `Command`. `PlaceOrderCommand`, `CancelOrderCommand`, `UpdateCustomerEmailCommand`.
- Queries: descriptive, ends with `Query`. `GetOrderByIdQuery`, `ListOrdersForCustomerQuery`, `SearchOrdersQuery`.
- Handlers: `<RequestName>Handler`. Always `internal sealed`.
- Validators: `<RequestName>Validator`. Always `internal sealed`.
- DTOs: `<Resource><Shape>` — `OrderDto`, `OrderSummary`, `OrderListItem`.

## Request shape

```csharp
public sealed record PlaceOrderCommand(
    CustomerId CustomerId,
    IReadOnlyList<PlaceOrderItem> Items,
    string? IdempotencyKey) : IRequest<Result<OrderId>>;

public sealed record PlaceOrderItem(ProductId ProductId, int Quantity);
```

- `record` types only.
- Strongly-typed IDs preferred over `Guid`/`int`.
- Collections as `IReadOnlyList<T>`.
- Nested types as records colocated in the same file.

## Handler shape

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

- Constructor: primary constructor with injected dependencies.
- No `try/catch` for expected failures — return `Result.Failure`.
- No `await db.SaveChangesAsync(ct)` inside command handlers — `UnitOfWorkBehavior` does this.
- No `IMediator.Send` from inside a handler. If you need orchestration, do it at the Web layer or via domain events.

## Validator shape

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
| `UnitOfWorkBehavior` | Commands (`IRequest<Result>` or `Result<T>` non-query) | Transaction + SaveChanges |
| `CachingBehavior` | Marked with `ICacheableQuery` | Read-through + invalidation |
| `PerformanceBehavior` | All | Log warning if elapsed > threshold |

## Caching

- Queries opt in via `ICacheableQuery` with `CacheKey`, `Expiration`, optional `Tags`.
- Commands opt in to invalidation via `IInvalidateCache` with the tags to evict.

## Domain event handlers

- Implement `INotificationHandler<TEvent>` in **Application**, not Domain.
- Handlers are independent — one event may have many handlers, none should depend on order.
- Long-running side effects (sending email, calling external systems) go via the **outbox** — handle the event by enqueueing an Infrastructure job, not by calling the external system synchronously.

## Forbidden

- Handlers that take `IServiceProvider` (service locator).
- `IMediator` injected anywhere except the Web layer.
- Returning entities from queries — always project to a DTO.
- Validators with DB calls (except the rare unique-constraint case).
- Splitting one logical use case across multiple handlers.

See [cqrs-implementation skill](../skills/cqrs-implementation/SKILL.md) for the canonical templates.
