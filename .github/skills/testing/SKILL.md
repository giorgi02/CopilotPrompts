---
name: testing
description: Write xUnit + FluentAssertions tests for this codebase — unit (Domain/Application), integration (Web with Testcontainers), and architecture tests. Use when adding tests for new behavior, designing a test strategy for a feature, or improving an existing test suite.
---

# Testing skill

Activates when generating or restructuring tests. Mirrors [.github/instructions/testing.instructions.md](../../instructions/testing.instructions.md).

## Pick the right test type

| What you are verifying | Type | Project |
|---|---|---|
| Aggregate / value object invariants | Unit | `Domain.UnitTests` |
| Handler `Result` branches | Unit | `Application.UnitTests` |
| HTTP request flow (status, headers, body) | Integration (Testcontainers) | `Web.IntegrationTests` |
| Layer dependency rule | Architecture | `Architecture.Tests` |
| Throughput / allocations | Performance (BenchmarkDotNet) | `Performance.Tests` |
| OpenAPI surface change | Snapshot (Verify) | `Contracts.Tests` |

If you reach for a database mock to write a "unit test", you are probably writing the wrong test type.

## Frameworks (locked)

- xUnit, FluentAssertions, NSubstitute (no Moq), Bogus (via builders), Verify, Testcontainers, NetArchTest.

## Naming

- `<Sut>Tests` class.
- `MethodUnderTest_Scenario_ExpectedOutcome` method (underscores allowed in tests).

## Shape — every test

```csharp
[Fact]
public async Task Handle_WhenCustomerInactive_ReturnsForbidden()
{
    // Arrange
    var customer = new CustomerBuilder().Inactive().Build();
    var sut = new PlaceOrderHandlerBuilder().WithCustomer(customer).Build();
    var command = new PlaceOrderCommand(customer.Id, [...]);

    // Act
    var result = await sut.Handle(command, TestContext.Current.CancellationToken);

    // Assert
    result.IsFailure.Should().BeTrue();
    result.Error.Should().Be(CustomerErrors.Inactive);
}
```

- Three blocks separated by blank lines.
- One behavior per test. Use `[Theory]` + `[InlineData]` over duplicated `[Fact]`s.
- Always pass `TestContext.Current.CancellationToken`.
- Use builders, not raw constructors.

## Builders pattern

`tests/Common/Builders/<Aggregate>Builder.cs`:

```csharp
public sealed class OrderBuilder
{
    private CustomerId _customerId = new(Guid.CreateVersion7());
    private List<OrderItem> _items = [new(...) ];

    public OrderBuilder ForCustomer(CustomerId id) { _customerId = id; return this; }
    public OrderBuilder WithItems(params OrderItem[] items) { _items = [..items]; return this; }
    public Order Build() => Order.Place(_customerId, _items, FakeTime.UtcNow).Value;
}
```

- One `Build()` returns a valid aggregate.
- Fluent setters for the bits that matter to the test.
- No reflection. No `internal` access tricks.

## Coverage targets

### Unit (handler)
- Happy path.
- Every `Result.Failure` branch.
- Cancellation propagation.

### Unit (aggregate)
- Each invariant (positive and negative).
- Each state transition.
- Each domain event raised.

### Integration (endpoint)
- 200/201 happy path.
- 401 unauthenticated.
- 403 unauthorized.
- 400 validation.
- Domain failure (404, 409).
- Idempotency replay (when applicable).

### Architecture (always present)
- Domain has no framework deps.
- Application has no Infrastructure deps.
- Web has no Infrastructure deps (only DI extensions).
- Handlers are `internal sealed`.
- Aggregates have private setters.

## Integration test fixture

```csharp
public sealed class WebFixture : IAsyncLifetime
{
    public PostgreSqlContainer Postgres { get; } = new PostgreSqlBuilder().WithImage("postgres:16-alpine").Build();
    public RedisContainer Redis { get; } = new RedisBuilder().WithImage("redis:7-alpine").Build();
    public WebApplicationFactory<Program> Factory { get; private set; } = default!;

    public async ValueTask InitializeAsync()
    {
        await Postgres.StartAsync();
        await Redis.StartAsync();
        Factory = new WebApplicationFactory<Program>().WithWebHostBuilder(b =>
            b.ConfigureAppConfiguration((_, cfg) =>
                cfg.AddInMemoryCollection(new Dictionary<string, string?>
                {
                    ["ConnectionStrings:Database"] = Postgres.GetConnectionString(),
                    ["ConnectionStrings:Redis"]    = Redis.GetConnectionString(),
                })));
    }
    public async ValueTask DisposeAsync()
    {
        Factory.Dispose();
        await Postgres.DisposeAsync();
        await Redis.DisposeAsync();
    }
}
```

## Forbidden

- Mocking `DbContext`. Use a real Postgres via Testcontainers or test the handler with an in-memory `IApplicationDbContext` fake.
- `Thread.Sleep`.
- Tests that assert on log output.
- Tests sharing mutable static state.
- `[Skip]` without an issue link.

## Output when generating

```
Test class: <Name>Tests
Type: <unit | integration | architecture>
Cases:
  - <method name> — <what it covers>
Builders used: <list>
Doubles used: <list>
```
