---
description: 'Generate a new HTTP endpoint following Minimal API + CQRS conventions, with validator, handler, OpenAPI metadata, and tests.'
agent: 'agent'
tools: ['search/codebase', 'search/usages', 'edit/files']
argument-hint: '<HTTP method> <route> for <feature>  (e.g. "POST /api/v1/orders for placing an order")'
---

# Generate a new API endpoint

You will scaffold a new endpoint end-to-end across the four layers, using the conventions in this repo. **Stay disciplined** — produce only the files listed below, in the locations listed, with the shapes shown.

## Inputs you must clarify before generating

1. **Resource and operation** (e.g. `Order`, `Place`).
2. **HTTP method and route** (e.g. `POST /api/v1/orders`).
3. **Auth policy** (e.g. `orders:write`).
4. **Request shape** — fields, types, constraints.
5. **Response shape** — DTO fields and the success status code.
6. **Failure modes** — which `Error` codes the use case can produce, and which HTTP statuses they map to.
7. **Idempotency required?** — `POST` endpoints with side-effects need an `Idempotency-Key`.

If any of these are missing, **ask** before generating.

## Files to produce

For an example operation `PlaceOrder` on resource `Order`, expect to create exactly:

```
src/Application/Features/Orders/PlaceOrder/PlaceOrderCommand.cs
src/Application/Features/Orders/PlaceOrder/PlaceOrderHandler.cs
src/Application/Features/Orders/PlaceOrder/PlaceOrderValidator.cs
src/Application/Features/Orders/PlaceOrder/OrderDto.cs                  ← only if not yet present
src/Web/Endpoints/Orders/PlaceOrderEndpoint.cs
src/Web/Endpoints/Orders/PlaceOrderRequest.cs
tests/Application.UnitTests/Features/Orders/PlaceOrderHandlerTests.cs
tests/Web.IntegrationTests/Orders/PlaceOrderEndpointTests.cs
```

Plus, **only if needed**:
- A new aggregate or value object in `src/Domain/Orders/`.
- An `IEntityTypeConfiguration<>` in `src/Infrastructure/Persistence/Configurations/`.
- A migration via `dotnet ef migrations add`.

## Conventions to apply

- Follow [.github/instructions/cqrs.instructions.md](../instructions/cqrs.instructions.md), [.github/instructions/api.instructions.md](../instructions/api.instructions.md), and [.github/instructions/web.instructions.md](../instructions/web.instructions.md).
- Endpoint stays **handler-thin** (≤ 15 lines).
- Handler is `internal sealed`, returns `Result<T>`.
- Validator covers shape, not business rules.
- Tests cover: happy path, each failure mode, validation failure, authorization failure (integration only).
- Add OpenAPI metadata: `.WithName`, `.WithTags`, `.Produces<T>`, every `.ProducesProblem(status)`, `.WithSummary`, `.WithDescription`.

## Process

1. **Locate a sibling endpoint** in the same area or the closest analog. Match its structure.
2. **Generate the Application slice** first (command + handler + validator + DTO).
3. **Generate the Web endpoint** that maps to the command.
4. **Generate the unit tests** for the handler covering all `Result` branches.
5. **Generate the integration test** that exercises the full HTTP path including auth.
6. **Verify** the dependency direction: did you accidentally introduce an Infrastructure import in Application or Web?
7. **List exactly** the files you created and a one-line summary of each.

## Output format

End with:

```
Files created:
  - <path> — <one-line summary>
  - ...

Next actions for the developer:
  - Run `dotnet build` and `dotnet test`.
  - If schema changed: `dotnet ef migrations add <Name> --project src/Infrastructure --startup-project src/Web`.
  - Update the OpenAPI snapshot test if applicable.
```

Do not output prose narration of what you are doing during generation. Output only the files and the final summary.
