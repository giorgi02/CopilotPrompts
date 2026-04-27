---
name: ddd-modeling
description: Tactical DDD modeling — aggregates, entities, value objects, domain events, and bounded contexts. Use when designing or refactoring a domain model, when an "anemic" smell is suspected, or when business invariants are leaking into Application or Web.
---

# DDD tactical modeling

This skill activates when you are shaping the **business core** — aggregates, value objects, events. It does not activate for HTTP / persistence / DI work.

## Core questions before code

1. What is this concept in the business language? (Get the team's term, not a generic one.)
2. Is it an **entity** (has identity through change), a **value object** (defined by attributes), or a **service** (a stateless operation across aggregates)?
3. If entity: what aggregate does it belong to? What is the **root**?
4. What invariants must hold at all times?
5. What state transitions are allowed?
6. What domain events does this raise?

## Aggregates

- One transactional consistency boundary. All changes inside an aggregate commit together.
- One aggregate per `SaveChanges`. Multi-aggregate operations communicate via **domain events** (handled async via outbox), not direct calls.
- Reference other aggregates by **ID only**, never by reference.
- The root has a private constructor; creation is a static factory `Create(...)` returning `Result<T>`.
- All mutations are methods. **Setters are private.** Public properties are read-only.

## Value objects

- Immutable, structurally equal, self-validating.
- Construction returns `Result<T>` for invalid inputs.
- No identity. Two `Money(10, USD)` are the same.
- Examples: `EmailAddress`, `Money`, `PhoneNumber`, `DateRange`, `Quantity`, `Address`, `PostalCode`.

## Entities (non-root)

- Have identity inside their aggregate. Equality by ID.
- Setters private. Mutations through methods.
- Cannot be loaded or saved independently of the root.

## Domain events

- Past tense: `OrderPlaced`, `PaymentCaptured`, `EmailVerified`.
- Records implementing `IDomainEvent`. Carry IDs and small value objects.
- Raised inside the aggregate method that caused them. Dispatched after `SaveChanges` via the outbox.

## Domain services

- Used **only** when logic spans multiple aggregates and cannot belong to one.
- Stateless. Named with the operation: `PricingCalculator`, `TransferEligibilityPolicy`.
- Located in `src/Domain/Services/`.

## Specifications

- Reusable predicates that express a domain rule.
- Pure expressions; return `Expression<Func<T, bool>>` so EF can translate.
- Located in `src/Domain/<Aggregate>/Specifications/`.

## Smells and fixes

| Smell | Fix |
|---|---|
| Anemic entities (just data, all logic elsewhere) | Move the logic into methods on the entity that enforce invariants. |
| Public setters on entities | Replace with intent-revealing methods (`Order.MarkAsShipped(at)`). |
| Aggregate that loads other aggregates by reference | Replace with FK by ID; load explicitly in the handler if needed. |
| Aggregate that "knows about" persistence | It does not. Even an `Id` property is `private set`. |
| Validator enforcing a domain rule | Move it into the aggregate method; the validator stays for shape only. |
| Domain event with a reference to a full aggregate | Carry IDs and value objects only — events outlive the in-memory aggregate. |
| Two aggregates being saved together | Split into one aggregate + one event handler that updates the other. |
| Primitive obsession (raw `string` for email, `decimal` for money) | Replace with a value object. |

## Naming

- Aggregates and entities: noun, business term — `Order`, not `OrderEntity`.
- Value objects: noun — `Money`, `EmailAddress`.
- Domain events: past tense — `OrderPlaced`.
- Domain services: noun phrase ending with the operation — `PricingCalculator`, `RefundEligibilityPolicy`.
- Errors: `<Aggregate>Errors.<SpecificName>` — `OrderErrors.AlreadyShipped`.

## Output when modeling

```
Concept: <name in business language>
Type: <aggregate root | entity | value object | domain service>
Aggregate: <root name> (or "is the root")
Identity: <strongly-typed ID or "value object — no identity">
Invariants:
  - <invariant>
  - ...
Operations (methods):
  - <Method>(<params>) — <effect, events>
Events emitted:
  - <Event> — <data carried>
Errors:
  - <Errors.X> — <when>
```

See [domain instructions](../../instructions/domain.instructions.md) — sections "Aggregate roots", "Value objects", and "Domain events".
