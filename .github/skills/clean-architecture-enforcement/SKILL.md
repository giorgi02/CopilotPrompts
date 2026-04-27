---
name: clean-architecture-enforcement
description: Enforce Clean Architecture dependency direction and layer responsibilities in this ASP.NET Core repo. Use when adding, moving, or reviewing code that crosses Domain / Application / Infrastructure / Web boundaries, or when proposing a new abstraction.
---

# Clean Architecture enforcement

The repo's structural integrity depends on dependency direction. This skill loads when:
- A file is being added or moved between layer projects.
- A new interface or abstraction is being proposed.
- A review needs to verify "is this in the right layer".

## The dependency rule (one rule)

**Source code dependencies point inward.** No exception.

```
Web → Application → Domain
         ↑
   Infrastructure
```

- `Domain` depends on **nothing** (BCL only).
- `Application` depends on `Domain` and on its own abstractions.
- `Infrastructure` depends on `Application` (to implement its abstractions) and `Domain`.
- `Web` depends on `Application` (to send commands/queries). It also references `Infrastructure` **only in `Program.cs` for DI composition**.

## Layer responsibilities

| Layer | Owns | Forbidden |
|---|---|---|
| Domain | aggregates, entities, value objects, domain events, domain services, errors, specifications | EF Core, MediatR, ASP.NET, JSON attributes, any NuGet beyond `System.*` |
| Application | use cases (commands/queries/handlers), validators, pipeline behaviors, abstractions (`IApplicationDbContext`, `IEmailSender`), DTOs, domain event handlers | EF Core, `Npgsql`, `HttpClient`, `IHttpContextAccessor`, business rules |
| Infrastructure | EF configurations, migrations, repositories, external service adapters, caching, messaging, background jobs, authentication backends | business rules, HTTP routing |
| Web | endpoints, middleware, OpenAPI, request DTOs, composition root | DB calls, business rules, MediatR pipeline behaviors |

## Decision flow — "where does this code go?"

```
Is it a business rule (invariant, value, calculation)?
  └─ Yes → Domain.
Does it orchestrate a use case (load → call domain → save)?
  └─ Yes → Application as a handler.
Does it touch I/O (DB, HTTP, file, broker, email, cache)?
  └─ Yes → Infrastructure.
Does it shape an HTTP request/response or wire DI?
  └─ Yes → Web.
None of the above?
  └─ It is probably a value object (Domain) or a small Application service.
```

## Common violations and the right fix

| Smell | Right fix |
|---|---|
| `using Microsoft.EntityFrameworkCore;` in `src/Domain/...` | Remove. The data is loaded in Application; Domain only knows aggregates. |
| Handler injects `ApplicationDbContext` directly | Inject `IApplicationDbContext` (defined in Application, implemented in Infrastructure). |
| Endpoint queries `DbContext` to skip a handler | Add a query handler. The endpoint dispatches via `ISender`. |
| `IHttpContextAccessor` injected into a handler | Wrap as `ICurrentUser` in Application; implement in Web/Infrastructure. |
| Aggregate has public setters | Replace with methods that enforce invariants. Setters become `private`. |
| Validator queries the DB | Move the rule to Domain or to the handler with an explicit lookup; validators are sync shape checks. |
| Application references a concrete `Stripe.PaymentService` | Application defines `IPaymentGateway`; Infrastructure implements with Stripe. |

## How to check a change

1. List the projects modified.
2. For each, list the new `using` directives.
3. Cross-reference against the layer's "forbidden" list.
4. If a new abstraction was added, confirm it is **defined** in the right layer (the consumer's, not the implementer's).

## Architecture tests

The repo's `tests/Architecture.Tests/` enforces these rules in CI. When a test fails:
- Read the failing assembly + offending types.
- Map the violation to a row in the table above.
- Apply the fix at the design level, not by suppressing the test.

## When to introduce a new abstraction

Only when **at least one** is true:
- It crosses a layer boundary (consumer is one layer in, implementer is the layer out).
- It has at least two implementations or a test fake is genuinely needed.
- It encapsulates a side effect (I/O, time, randomness) you need to control.

Otherwise, do not abstract. A class with one consumer and one implementation is just a class.

## Output

When this skill is engaged in a review or design conversation, end with:

```
Layer plan:
- Domain: <what changes, or "no change">
- Application: <what changes>
- Infrastructure: <what changes>
- Web: <what changes>

Dependency check: <pass | violations: list with rule>
New abstractions introduced: <list with consumer-layer location>
```

See [.github/instructions/domain.instructions.md](../../instructions/domain.instructions.md), [.github/instructions/application.instructions.md](../../instructions/application.instructions.md), [.github/instructions/infrastructure.instructions.md](../../instructions/infrastructure.instructions.md), [.github/instructions/web.instructions.md](../../instructions/web.instructions.md).
