---
description: 'Generate xUnit + FluentAssertions tests (unit, integration, or architecture) for the file or feature in context.'
agent: 'agent'
tools: ['search/codebase', 'search/usages', 'edit/files']
argument-hint: '<unit|integration|architecture> for <file or feature>'
---

# Generate tests

## Decide the test type first

| If you are testing... | Use |
|---|---|
| A pure aggregate, entity, or value object | **Unit** in `Domain.UnitTests` |
| A MediatR handler | **Unit** in `Application.UnitTests` |
| An HTTP endpoint or full request flow | **Integration** in `Web.IntegrationTests` |
| A dependency-direction or naming rule | **Architecture** in `Architecture.Tests` |
| A throughput / allocation budget | **Performance** in `Performance.Tests` (BenchmarkDotNet) |

If you are tempted to mock a database, **stop** — that is an integration test using Testcontainers, not a unit test.

## Conventions

From [.github/instructions/testing.instructions.md](../instructions/testing.instructions.md):

- `<Sut>Tests` class, `MethodUnderTest_Scenario_ExpectedOutcome` method names.
- Arrange-Act-Assert with blank lines.
- One behavior per test. `[Theory]` + `[InlineData]` over copy-paste.
- FluentAssertions everywhere: `result.Should().BeTrue()`, never `Assert.True(...)`.
- NSubstitute for doubles. **No Moq.**
- Builders for arrange (`OrderBuilder`, `CustomerBuilder`).
- Pass `TestContext.Current.CancellationToken` to async calls.

## Coverage targets per type

### Unit (handler / aggregate)
- Happy path.
- Every `Result.Failure` branch (one test each).
- Cancellation propagation (pass cancelled token).
- For aggregates: every invariant and every state transition.

### Integration (endpoint)
- 200/201 happy path.
- Authentication failure (401).
- Authorization failure (403).
- Validation failure (400 with `ProblemDetails.errors`).
- Domain failure (409, 404 — whichever applies).
- Idempotency (replay returns the same response).

### Architecture
- Layer dependency rules from [anti-patterns](../instructions/anti-patterns.instructions.md).

## Process

1. **Read the SUT** and list every public path through it.
2. **Identify the test type** (unit/integration/architecture).
3. **Find the existing test file** for this feature, or its closest sibling.
4. **Match the file's style** (builder usage, arrange section structure, naming).
5. **Generate one test per behavior**, in the order: happy → failure modes → edge cases.
6. **Verify** that no test asserts on log output or shared mutable state.

## Output

```
Test class: <Name>Tests
Type: <unit|integration|architecture>
Tests added: <count>
Coverage: <branches/paths covered>
```
