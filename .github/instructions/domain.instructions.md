---
name: 'Domain layer rules'
description: 'DDD aggregate, entity, and value object rules for the Domain project. Pure C#, zero framework dependencies.'
applyTo: 'src/Domain/**/*.cs'
---

# Domain layer rules

The Domain layer is the **center of gravity**. It encodes business rules, invariants, and the ubiquitous language. Everything else exists to serve it.

## Hard constraints

- **Zero framework dependencies.** No `Microsoft.*`, no EF Core, no MediatR, no Serilog, no `System.Text.Json` attributes, no ASP.NET, no NuGet packages other than `System.*` BCL.
- Pure C# only. If the type cannot compile against `netstandard2.1`, it does not belong here.
- No I/O. No async. No `Task`-returning methods unless modeling a true domain async concept (rare).
- No primitive obsession. Use **value objects** for any concept the business gives a name to.

## Aggregate roots

- Inherit `AggregateRoot<TId>` (raises domain events).
- Identity is a strongly-typed ID value object (e.g. `OrderId(Guid Value)`), **not** raw `Guid`.
- Constructors are `private`. Creation goes through a static factory: `Order.Create(...)` returning `Result<Order>`.
- All mutations are methods that enforce invariants. **No public setters.**
- Raise domain events via `Raise(new OrderPlaced(...))`.
- Aggregates do not call other aggregates' methods directly — they communicate via domain events handled in Application.

## Entities

- Inherit `Entity<TId>`.
- Identity-based equality (compare IDs).
- Setters are `private`. Mutations are methods.
- No reference to a `DbContext`, no navigation property loading logic.

## Value objects

- Inherit `ValueObject` (structural equality via `GetEqualityComponents()`) **or** use `record` types.
- Immutable. Self-validating in their constructor.
- Examples: `Money`, `EmailAddress`, `PhoneNumber`, `Address`, `DateRange`, `Quantity`.
- Construction returns `Result<T>` for invalid inputs: `EmailAddress.Create("not-an-email")` → `Result.Failure(DomainErrors.Email.Invalid)`.

## Domain events

- Past-tense names: `OrderPlaced`, `PaymentCaptured`, `InvoiceIssued`.
- Records implementing `IDomainEvent`.
- Carry only the data needed for handlers — IDs and value objects, not full aggregates.
- Dispatched by Infrastructure after `SaveChanges` (transactional outbox pattern), not in the aggregate method itself.

## Domain services

- Used **only** when logic spans multiple aggregates and cannot belong to any single one.
- Stateless. Named with the operation: `PricingCalculator`, `TransferEligibilityPolicy`.
- Located under `src/Domain/Services/`.

## Errors

- Static error catalog per aggregate: `OrderErrors.NotFound`, `OrderErrors.AlreadyShipped`.
- Type: `Error(string Code, string Message, ErrorType Type)` where `ErrorType` ∈ `{ Validation, NotFound, Conflict, Unauthorized, Forbidden, Failure }`.

## Specifications

- Use the **Specification pattern** for reusable query predicates that express a domain rule (e.g. `ActiveCustomerSpecification`).
- Specifications are pure expressions. They return `Expression<Func<T, bool>>` so EF Core can translate them.
- File location: `src/Domain/<Aggregate>/Specifications/`.

## Forbidden in this layer

- `using Microsoft.EntityFrameworkCore;`
- `using MediatR;`
- `using System.Text.Json.Serialization;`
- `[Required]`, `[MaxLength]`, `[Column]` and any other persistence/serialization attributes.
- Constructors that take repositories, services, or `DbContext`.
- `static` factory methods that perform I/O.

## File layout per aggregate

```
src/Domain/Orders/
  Order.cs                     ← aggregate root
  OrderId.cs                   ← strongly-typed ID
  OrderItem.cs                 ← entity
  OrderStatus.cs               ← enum or smart enum
  Events/
    OrderPlaced.cs
    OrderShipped.cs
  Errors/
    OrderErrors.cs
  Specifications/
    OrdersAwaitingShipmentSpecification.cs
```

See the [ddd-modeling skill](../skills/ddd-modeling/SKILL.md) for full aggregate templates.
