---
name: 'C# language conventions'
description: 'Modern C# style, nullability, async, pattern matching, and language version conventions for every .cs file.'
applyTo: '**/*.cs'
---

# C# language conventions

## Language version
- Target **C# 13** (.NET 9). Use modern features: `field` keyword, primary constructors, collection expressions, `params Span<T>`, `Lock` type.
- `<Nullable>enable</Nullable>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` are mandatory in every project.
- File-scoped namespaces. One type per file.
- `using` directives **outside** the namespace, sorted, system namespaces first.
- Top-level statements only in `Program.cs`.

## Nullability
- Reference types are non-nullable by default. Mark intent with `?`.
- Never silence with `!` unless paired with a comment explaining the invariant.
- Constructor-required references use `required` or are set via primary constructor parameters.
- `ArgumentNullException.ThrowIfNull(arg)` for public method preconditions.

## Async / await
- Every async method ends in `Async` and accepts `CancellationToken cancellationToken` (last parameter, no default in libraries; `default` allowed in Web layer entry points).
- Forward `cancellationToken` to every awaited call.
- Never `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`. Never `Task.Run` to escape sync context.
- `async void` only for event handlers.
- Use `ValueTask` only for hot paths with measured benefit, not by default.

## Pattern matching & expressions
- Prefer switch expressions over switch statements.
- Prefer pattern matching over `is X x && x.Y == ...` chains.
- Use `is null` / `is not null`, never `== null`.

## Collections
- Use collection expressions: `int[] x = [1, 2, 3];`, `List<int> y = [..source];`.
- Return `IReadOnlyCollection<T>` / `IReadOnlyList<T>` from public APIs unless mutability is required.
- Prefer `IEnumerable<T>` for streaming results, materialize at the boundary.

## Types
- `record` for DTOs and value objects (immutable, structural equality).
- `record struct` for small (<16 bytes) immutable values.
- `class` for entities, services, handlers.
- `sealed` by default. Open for extension only with intent.
- Static classes only for true utilities; prefer DI-able interfaces.

## Equality and hashing
- Override both `Equals` and `GetHashCode` together, or use a `record`.
- Do not implement `IComparable` unless ordering is a domain concept.

## Exceptions
- Throw the most specific exception. Do not throw `Exception` or `ApplicationException`.
- Do not catch `Exception` except in top-level boundaries (middleware, hosted services, background jobs).
- Custom exceptions inherit from `Exception` directly, not `ApplicationException`.
- Validation, not-found, and conflict are **not** exceptions in this codebase — they are `Result.Failure`.

## Logging
- `ILogger<T>` injected by constructor or primary constructor.
- Use `LoggerMessage` source generators (`[LoggerMessage]` attribute) for hot paths.
- Structured logging only: `_logger.LogInformation("Order {OrderId} placed by {UserId}", orderId, userId);`.
- Never log secrets, tokens, full request bodies, PII.

## Disposability
- `IDisposable`/`IAsyncDisposable` resources go in `using` declarations.
- Implement `IAsyncDisposable` if you own async resources.

## Reflection / dynamic
- `dynamic` is forbidden.
- Reflection is allowed in startup/composition only, never in request hot paths.

## Comments
- No XML doc comments on internal/private members.
- XML docs on public API contracts (Application interfaces, Domain aggregates) — describe the *contract*, not the implementation.
- Inline comments only for non-obvious *why*.
