---
name: 'Anti-patterns and review heuristics'
description: 'Named anti-patterns for fast review communication, plus the architectural and dependency rules that fail the build or block PRs.'
applyTo: '**/*.cs,**/*.csproj,**/Directory.Packages.props,**/Directory.Build.props'
---

# Anti-patterns and review heuristics

A named smell is faster to review than a paragraph of explanation. Use these names in PR comments. Each entry has a one-line description and the canonical fix.

This file complements — and does not replace — the layer-specific rules in [domain.instructions.md](./domain.instructions.md), [application.instructions.md](./application.instructions.md), [infrastructure.instructions.md](./infrastructure.instructions.md), [web.instructions.md](./web.instructions.md), [security.instructions.md](./security.instructions.md), and [performance.instructions.md](./performance.instructions.md).

---

## Architectural anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **Domain bleed** | Framework or persistence dep imported into Domain (`using Microsoft.EntityFrameworkCore;` in `src/Domain`) | Remove. Move logic to Application/Infrastructure. |
| **Layer skip** | Web calling Infrastructure (or `DbContext`) directly | Add an Application handler. Web dispatches via `ISender`. |
| **Reverse dependency** | Application referencing a concrete Infrastructure type | Define the interface in Application; implement in Infrastructure. |
| **Anemic domain** | Entities reduced to data bags; logic lives in services | Push behavior into entity methods that enforce invariants. |
| **Cross-aggregate transaction** | One handler mutates multiple aggregates in one `SaveChanges` | One aggregate per `SaveChanges`. Use domain events + outbox. |
| **God handler** | One handler doing many things — load, validate, mutate two aggregates, send email, log audit | Split into multiple handlers, push to domain methods, or use events. |
| **God service** | A `XxxService` class with ten unrelated public methods | Split by use case into handlers; or by domain concept into a domain service. |
| **Service-locator** | `IServiceProvider` injected into a handler | Inject the specific dependencies. |
| **Handler chaining** | `IMediator.Send(...)` from inside a handler | Orchestrate at the Web layer or via domain events. |
| **Validator with DB** | A FluentValidation rule that hits the database (other than rare unique-constraint checks) | Move to the handler or Domain. |
| **Query with side effects** | A query that writes to the DB or sends an email | Split into a command + a query. |
| **Mapping in handler** | Loading an entity then converting to DTO in code | Use EF projection (`Select(...)`) directly to DTO. |

## Domain anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **Primitive obsession** | Raw `string` for email, `decimal` for money, `Guid` for entity identity | Introduce a value object or strongly-typed ID. |
| **Public setter** | `public string Status { get; set; }` on an entity | `private set` and an intent-revealing method. |
| **Loaded reference between aggregates** | `Order.Customer` navigation property | `Order.CustomerId`. Load the customer separately if needed. |
| **Synchronous side effect in aggregate** | Aggregate method calls an external service | Raise a domain event. Outbox + handler triggers the side effect. |
| **Aggregate event with full aggregate reference** | Event carries `Order` rather than `OrderId` + value objects | Carry IDs + value objects only — events outlive the aggregate. |

## Persistence anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **Lazy loading** | Enabled at `DbContext` level | Disable. Explicit `Include` or projection. |
| **`FromSqlRaw` interpolation** | `FromSqlRaw($"...{input}...")` | Parameterize. Or use Dapper with `@params`. |
| **Repository over `DbContext`** | Wrapping `DbContext` in a generic `Repository<T>` to "abstract EF" | Use `IApplicationDbContext` directly. Repos are for aggregates with non-trivial loading only. |
| **Repository returning `IQueryable`** | Leaks persistence concerns; defeats the purpose | Return aggregates. |
| **`ToList()` early** | Materializes before filters/projection | Keep the query as `IQueryable` until the end. |
| **Missing `AsNoTracking()` on read** | Tracking entities you'll never modify | Add `AsNoTracking()`. |
| **N+1** | `foreach` calling a method that hits the DB | Single query with `Include` / projection / batching. |

## Web / API anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **Endpoint logic** | Business rules inside a Minimal API handler lambda | Dispatch via MediatR. Endpoint is < 15 lines. |
| **Custom controller base** | `BaseApiController` hiding cross-cutting concerns | Use Minimal APIs + endpoint filters / pipeline behaviors. |
| **Anonymous response** | `return Results.Ok(new { id, status })` | Use a typed response DTO. |
| **Status code via JSON** | `return Ok(new { success = false, code = "FORBIDDEN" })` | Return the actual HTTP status (403 with `ProblemDetails`). |
| **Missing OpenAPI metadata** | Endpoint without `.Produces` / `.ProducesProblem` for every status | Declare every status. |

## Error-handling anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **Catch-and-null** | `catch (Exception) { return null; }` | Let it bubble, or convert at the layer with the contract authority. |
| **Catch-rethrow-with-log** | Same exception logged at three layers | The global handler logs once. Don't catch unless you can do something useful. |
| **Try-catch around domain rule** | Wrapping `domainObject.DoSomething()` to catch a "validation" exception | Domain methods return `Result`; check `IsFailure`. |
| **Throw for expected failure** | `throw new NotFoundException(...)` for "user not found" | Return `Result.Failure(Error.NotFound(...))`. |

## Test anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **Mocked `DbContext`** | Substituting `DbContext` to "unit test" a query | Integration test with Testcontainers, or hand-rolled `IApplicationDbContext` fake for shape. |
| **Log assertion** | Asserting on log output as a behavioral contract | Assert on outcomes; logs are observability. |
| **`Thread.Sleep`** | Used for synchronization | `await Task.Delay`, polling helper, or rendezvous. |
| **Skipped test** | `[Skip("flaky")]` without an issue link | Fix or delete. |
| **Moq usage** | Mixing test doubles | Use NSubstitute exclusively. |
| **Raw `Assert.*`** | `Assert.Equal(...)` | Use FluentAssertions. |

## Naming anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **`Helper`, `Util`, `Manager`, `Processor`** | Names that say nothing about the type's role | Name by domain concept and operation. |
| **`Service` for a use case** | `PlaceOrderService` instead of `PlaceOrderHandler` | Handler. |
| **`Common`, `Shared`, `Misc`** | Catch-all folders | Organize by feature or responsibility. |

## Dependency anti-patterns

| Smell | Description | Fix |
|---|---|---|
| **Pinned-by-accident** | Package on an old version with no rationale | Bump or document why pinned. |
| **Single-use library** | NuGet for one helper function | Write the function. |
| **Two libraries doing the same job** | AutoMapper + Mapster, or Moq + NSubstitute | Pick one; remove the other. |

---

# Hard rules — the build or PR check fails on these

## Architecture (architecture tests in CI)
- Domain has no framework deps (`using Microsoft.*`, `MediatR`, `EF Core`, etc.).
- Application has no Infrastructure deps.
- Web has no Infrastructure types in handlers/endpoints (composition root excepted).
- Handlers are `internal sealed`.
- Aggregates have private setters.

## Async hygiene (analyzers, warnings as errors)
- `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` on `Task`.
- `async void` outside event handlers (VSTHRD100).
- Missing `CancellationToken` forwarding (CA2016).

## Security (build and CI gates)
- New endpoints without `RequireAuthorization(...)` unless explicitly justified as public.
- `[Authorize(Roles = "...")]` strings — use policies.
- Tenant id sourced from request body — source from `ICurrentUser`.
- Custom JWT validators.
- `BinaryFormatter`, `XmlSerializer` on untrusted input.
- Disabled SSL certificate validation.
- MD5/SHA-1/plain SHA-256 for passwords.
- `Random.*` for security-sensitive randomness.
- Symmetric JWT signing keys committed.
- `dotnet list package --vulnerable --include-transitive` returning Critical/High.
- GPL/AGPL licensed dependencies.

## Persistence
- `EnsureCreated()` anywhere.
- Editing a migration after merge.
- `DropColumn` + `AddColumn` of the same logical concept in one migration.
- Lazy loading enabled.

# Adding a new dependency — vetting checklist

A new NuGet package needs **all** of:
- A real, immediate need in this PR (not "we might use it later").
- Publisher with > 100k total downloads, > 1 year maintained history, OR Microsoft / well-known OSS.
- License compatible (no GPL-2.0/3.0/AGPL).
- No existing package in the codebase covers the use case.
- An update to the relevant `.github/instructions/*.instructions.md` if it introduces a new pattern (background jobs, mapping, messaging), with rationale in the PR description.

## Locked dependencies (require explicit team agreement to bump major)
- `MediatR` (license model)
- `FluentValidation` (major version)
- `Microsoft.EntityFrameworkCore` (major)
- Any package powering the persistence outbox / messaging.

## Forbidden packages

| Package | Why | Use instead |
|---|---|---|
| `Newtonsoft.Json` (in new code) | Replaced by `System.Text.Json` | `System.Text.Json` |
| `AutoMapper` | Mapping policy — manual mapping is the standard | Manual mapping or Mapster |
| `Moq` | Testing stack standard is NSubstitute | NSubstitute |
| Any package using `BinaryFormatter` | Security | `System.Text.Json` |
| `EntityFramework` (EF6) | Legacy | `Microsoft.EntityFrameworkCore` |

## Per-layer dependency constraints

| Project | Allowed deps |
|---|---|
| `src/Domain` | `System.*` BCL only. **Zero** NuGet packages. |
| `src/Application` | MediatR, FluentValidation, `Microsoft.Extensions.Logging.Abstractions`, Mapster (if used). No EF Core. No HttpClient. No ASP.NET. |
| `src/Infrastructure` | Persistence (EF Core, Npgsql, Dapper), caching (`HybridCache`, StackExchange.Redis), HTTP (`IHttpClientFactory`, Polly), messaging, Hangfire/Quartz. |
| `src/Web` | ASP.NET Core, Asp.Versioning, OpenAPI, Scalar, Serilog/OpenTelemetry hosting. |

When you spot one of these in a review, name it. "This is a 'God handler'" is faster than "this method does too many things and could be split because…"
