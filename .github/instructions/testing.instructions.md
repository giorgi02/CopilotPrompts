---
name: 'Testing conventions'
description: 'xUnit, FluentAssertions, NSubstitute, and Testcontainers conventions for unit, integration, and architecture tests.'
applyTo: '**/tests/**/*.cs,**/*.Tests/**/*.cs,**/*.IntegrationTests/**/*.cs,**/*.ArchitectureTests/**/*.cs'
---

# Testing conventions

## The pyramid (target ratios)

- **Unit tests** (Domain, Application handlers): ~70% of test count, sub-second.
- **Integration tests** (Web → Application → real DB via Testcontainers): ~25%.
- **Architecture tests** (NetArchTest, dependency rules): ~3%.
- **End-to-end** (full system in docker-compose, real broker): ~2%, only for critical user journeys.

If unit tests get too slow or too brittle, you are testing infrastructure with the wrong tool — push the test up the pyramid.

## Frameworks (locked)

- **xUnit** — test runner.
- **FluentAssertions** — assertions. `result.Should().BeXxx()` only; never raw `Assert.Equal`.
- **NSubstitute** — test doubles. **Do not introduce Moq.**
- **Testcontainers** — real Postgres, Redis, RabbitMQ in integration tests. **No SQLite-as-substitute.**
- **Bogus** — realistic data builders. Wrap in `*Builder` classes per aggregate.
- **Verify** — snapshot tests for OpenAPI, large DTOs, generated SQL.
- **NetArchTest** or **ArchUnitNET** — architecture rules.

## Test project layout

```
tests/
  Domain.UnitTests/
  Application.UnitTests/
  Web.IntegrationTests/
  Architecture.Tests/
  Contracts.Tests/      ← API contract / OpenAPI snapshot
  Performance.Tests/    ← BenchmarkDotNet, opt-in via category
```

One test project per source project. Mirror namespaces.

## Naming

- Test class: `<SystemUnderTest>Tests` (e.g. `PlaceOrderHandlerTests`).
- Test method: `MethodUnderTest_Scenario_ExpectedOutcome`. Underscores allowed in tests.
  - Good: `Handle_WhenCustomerInactive_ReturnsForbidden`
  - Bad: `Test1`, `HappyPath`

## Structure

```csharp
[Fact]
public async Task Handle_WhenCustomerInactive_ReturnsForbidden()
{
    // Arrange
    var customer = new CustomerBuilder().Inactive().Build();
    var handler = new PlaceOrderHandlerBuilder().WithCustomer(customer).Build();
    var command = new PlaceOrderCommand(customer.Id, items: [...]);

    // Act
    var result = await handler.Handle(command, TestContext.Current.CancellationToken);

    // Assert
    result.IsFailure.Should().BeTrue();
    result.Error.Should().Be(CustomerErrors.Inactive);
}
```

- One behavior per test. Use `[Theory]` + `[InlineData]` for parameterized cases.
- Always pass `TestContext.Current.CancellationToken` to async calls.
- Builders for arrange. Never construct aggregates with raw constructors in tests beyond simple value objects.

## Application handler tests (unit)

- **No DB.** Substitute `IApplicationDbContext` with an in-memory list-backed fake or use `EntityFrameworkCore.InMemory` only for `DbSet<T>` shape — never for SQL semantics.
- Substitute external collaborators with NSubstitute.
- Use `FakeTimeProvider` from `Microsoft.Extensions.TimeProvider.Testing`.
- Assert against `Result` and against domain events raised on aggregates.

## Domain tests (unit)

- No mocks. Domain has no dependencies. Test invariants directly on aggregates and value objects.
- Drive with a `SystemUnderTest()` factory or a builder.

## Integration tests

- One `WebApplicationFactory<Program>` per test class via `IClassFixture`.
- Postgres + Redis from Testcontainers, started once per test run via an `IAsyncLifetime` fixture using `xunit.v3` `[Collection]`.
- Each test gets a fresh DB schema OR a transaction-rollback wrapper. Pick one strategy per project, document it.
- Authenticate via a `TestAuthHandler` that issues a fake bearer with arbitrary claims.
- Tests hit real HTTP via `HttpClient`. Assert on status code, headers, and JSON shape.

## Architecture tests (mandatory)

A baseline set in `Architecture.Tests`:

```csharp
[Fact]
public void Domain_ShouldNot_DependOn_Anything_ButBcl()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .Should()
        .NotHaveDependencyOnAny("Microsoft", "MediatR", "FluentValidation", "Npgsql")
        .GetResult();
    result.IsSuccessful.Should().BeTrue(string.Join("\n", result.FailingTypeNames ?? []));
}
```

Cover:
- Domain → no framework deps.
- Application → no Infrastructure deps.
- Web → no Infrastructure deps (only via DI extensions).
- Handlers are `internal sealed`.
- Aggregates have private setters.

## Contract / snapshot tests

- Snapshot the OpenAPI document with **Verify**. PRs that change the API surface must commit the updated snapshot — reviewers see the diff.
- Snapshot generated EF SQL for read-heavy queries.

## Performance tests (opt-in)

- `BenchmarkDotNet`, gated by `[Trait("Category","Performance")]` and excluded from default CI runs.
- Run on a separate workflow with stable hardware tags.

## Forbidden

- `Thread.Sleep` for waiting — use `await Task.Delay` with a hard timeout via `WaitForAsync` helpers.
- `[Skip]` attributes without an issue link.
- Tests that assert log output (brittle) — assert on outcomes.
- Tests that share mutable static state.
- `var sut = new XxxHandler(Substitute.For<...>(), Substitute.For<...>(), ...)` — use builders.
