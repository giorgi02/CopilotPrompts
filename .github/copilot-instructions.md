# Repository instructions for GitHub Copilot

> Loaded on every Copilot interaction. Stays under 4,000 chars (Copilot Code Review truncates the rest). Layer detail lives in [.github/instructions/](./instructions/); workflows in [.github/prompts/](./prompts/), [.github/skills/](./skills/), [.github/agents/](./agents/).

## What this codebase is
ASP.NET Core 9 Web API on **Clean Architecture**. Layers **Domain → Application → Infrastructure → Web**. Dependencies point inward only; Domain depends on nothing; `Program.cs` composes everything.

## Hard rules — never violate

1. **No layer skipping.** Web → Application → Domain. Infrastructure implements abstractions defined in Application. Web references Infrastructure only in `Program.cs`.
2. **Domain has zero framework deps.** No `Microsoft.*`, EF Core, MediatR, JSON attributes, ASP.NET — `System.*` BCL only.
3. **CQRS via MediatR.** Commands return `Result`/`Result<T>`; queries return `Result<TDto>`. One handler per request, `internal sealed`.
4. **Result pattern for expected failures.** Validation/not-found/conflict/unauthorized → `Result.Failure`. Throw only for programmer errors.
5. **FluentValidation in a `ValidationBehavior`** runs before handlers. Handlers assume valid input.
6. **Persistence hidden behind `IApplicationDbContext`** or aggregate repositories. Never inject `DbContext` into handlers.
7. **Aggregates own invariants.** Mutations via methods. Setters are `private`.
8. **`CancellationToken`** through every async chain. No `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`; no `async void` outside event handlers.
9. **`ProblemDetails` (RFC 7807)** at the API boundary. Never leak exception text.
10. **OpenAPI metadata** on every public endpoint: `Produces`, every `ProducesProblem`, `WithName`, `WithTags`.

## Defaults Copilot applies without asking

- Endpoints → **Minimal APIs** as `IEndpoint` modules per feature, mapped via one `MapEndpoints()` extension.
- Use cases → **vertical slice** at `src/Application/Features/<Feature>/<UseCase>/`.
- Persistence → **EF Core 9 + PostgreSQL** (snake_case), `AsNoTracking()` on reads, `Select(...)` projection to DTO.
- Logging → **Serilog** structured props; `LoggerMessage` on hot paths. Never log secrets, tokens, PII.
- Telemetry → **OpenTelemetry** OTLP. Activity names `Service.Layer.Operation`.
- Caching → `HybridCache` with explicit TTL. Background work → **Hangfire** + `Channel<T>` queues.
- IDs: **Guid v7**. Time: inject **`TimeProvider`**. Money: `Money` value object. Equality: records / identity-only entities.

## Forbidden — refuse and propose the alternative

- `DbContext` in Domain or Web (composition root excepted).
- `try { … } catch (Exception) { return null; }` — silent swallowing.
- `Repository<T>` wrapping `DbContext` for its own sake (repos are for aggregates).
- `dynamic`, `BinaryFormatter`, reflection on hot paths.
- SQL string concat. Magic strings/numbers. Public setters on entities.
- Custom controller bases — use Minimal APIs + endpoint filters.

## Testing expectations

- **xUnit + FluentAssertions + NSubstitute + Testcontainers.** No Moq, no SQLite-as-substitute.
- Handler tests: in-memory, no DB, `FakeTimeProvider`. Integration: real Postgres + Redis via Testcontainers.
- Coverage gate **80% line / 70% branch** on Application + Domain. Infrastructure exempt.
- One behavior per test. `[Theory]` + `[InlineData]` over duplicated `[Fact]`.

## Reviews

- New patterns, libraries, or cross-cutting policies must update the relevant `.github/instructions/*.instructions.md` and explain the rationale in the PR description.

## When in doubt

Run a prompt from [.github/prompts/](./prompts/): `/api-generation`, `/bug-investigation`, `/migration`, `/refactoring`, `/testing`, `/code-review`, `/security-review`, `/performance-review`, `/architecture-review`. Specialist personas live in [.github/agents/](./agents/).
