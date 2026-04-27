---
description: 'Targeted performance review of a code path or endpoint, with concrete remediation suggestions.'
agent: 'agent'
tools: ['search/codebase', 'search/usages']
argument-hint: '<file, method, or endpoint to review>'
---

# Performance review

Apply the rules in [performance instructions](../instructions/performance.instructions.md) and [performance guardrails in anti-patterns](../instructions/anti-patterns.instructions.md). Focus on **measured, plausible** wins. Don't chase micro-optimizations that don't move the SLO needle.

## Scan in this order

### 1. Database
- Missing `AsNoTracking()` on read queries.
- Loading entities then mapping in memory instead of `Select(...)` projection.
- N+1 — any `foreach` that calls a method which queries the DB.
- `Include` on large collections without `AsSplitQuery()`.
- Missing index on a hot `WHERE` / `ORDER BY` column.
- Unbounded query (no `Take`).
- Lazy loading enabled (it should not be).

### 2. Async
- `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`.
- Missing `cancellationToken` propagation.
- Sequential `await` calls that are independent (use `Task.WhenAll`).
- `async void` outside event handlers.
- `Task.Run` on already-async paths.

### 3. Allocations
- `LogInformation($"...")` — interpolation in log messages.
- LINQ chains in hot paths.
- `string` concatenation in loops.
- `new HttpClient()` instead of typed client via factory.
- `JsonSerializer.Serialize(...)` with a fresh `JsonSerializerOptions` per call.

### 4. Caching
- A hot read endpoint with no caching.
- Cache without explicit TTL.
- Cache stampede risk (no protection on a slow upstream).

### 5. HTTP
- Missing `IHttpClientFactory`.
- No timeout configured on outbound HTTP.
- No Polly resilience pipeline (retry + circuit breaker).

## Output

```
## Performance review of <scope>

### Findings (sorted by impact)

1. **<one-line problem>** — <file>:<line>
   Impact: <DB hits saved | allocations saved | latency improvement (estimated)>
   Fix: <concrete change>

2. ...

### Wins not worth pursuing
- <thing that looks like a problem but isn't worth the change>

### Suggested benchmark
- If you want to verify the largest finding, write a `BenchmarkDotNet` benchmark in `tests/Performance.Tests/` that exercises <method>.
```

## Honesty requirement

- If the code is already efficient, say so. Don't fabricate findings.
- If you can't tell whether a change matters without measurement, say "needs measurement" and propose a benchmark — don't recommend the change blind.
