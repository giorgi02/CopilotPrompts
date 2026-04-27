---
name: performance-engineering
description: Find and fix performance issues — DB N+1, allocations, async hygiene, caching, HTTP timeouts. Use when investigating latency, allocations, or throughput problems, when reviewing hot paths, or when adding caching to a read endpoint.
---

# Performance engineering

Activates when investigating latency, allocations, or throughput, or when reviewing a hot path.

## Targets

- API p95: < 300ms read, < 800ms write.
- DB round-trips per request: 1 ideal, 2–3 acceptable.
- Allocations per request on hot paths: < 50.
- GC pause p99: < 50ms.

## Investigation order — check the cheap stuff first

### 1. Database (almost always the answer)
- Missing `AsNoTracking()`?
- Loading entities then mapping in memory? Use `Select(...)` projection.
- N+1? Look for `foreach` calling a method that hits the DB.
- `Include` of a 1-to-many causing a Cartesian explosion? Use `AsSplitQuery()`.
- Missing index on `WHERE` / `ORDER BY` / FK column?
- Unbounded query missing `Take(n)`?
- Lazy loading enabled? It must not be.

### 2. Async
- `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` anywhere?
- `cancellationToken` propagated end-to-end?
- Sequential awaits on independent calls — use `Task.WhenAll`?
- `async void` outside event handlers?
- `Task.Run` on already-async paths?

### 3. Allocations
- `LogInformation($"...")` interpolation? Use structured logging or `LoggerMessage`.
- LINQ chain on a hot path? Replace with `for`/`foreach`.
- `string` concat in a loop? `StringBuilder`.
- `new HttpClient()`? Typed client via factory.
- `JsonSerializer.Serialize(...)` with fresh options each call? Cache `JsonSerializerOptions`.

### 4. Caching
- Hot read endpoint with no cache? Add `HybridCache` with explicit TTL.
- Cache without TTL? Always TTL.
- Stampede risk? `HybridCache` handles it; for `IDistributedCache`, semaphore by key.

### 5. HTTP
- `IHttpClientFactory` typed client used? Never `new HttpClient()`.
- Polly resilience pipeline registered (timeout → retry → circuit breaker)?
- Outbound timeout configured at both client and pipeline?

## Tools

- **Profiling**: `dotnet-trace`, `PerfView`, JetBrains dotTrace.
- **Allocation**: `dotnet-counters`, `dotnet-gcmon`.
- **DB**: `EF.CoreEnableSensitiveDataLogging` locally, `pg_stat_statements` on Postgres.
- **Benchmarks**: `BenchmarkDotNet` in `tests/Performance.Tests/`.
- **HTTP load**: `k6`, `bombardier`.

## Don't optimize without measurement

For any "this should be faster" hypothesis:
1. Write a benchmark that exercises the path.
2. Capture before / after.
3. Only ship the change if before > after by a meaningful margin.

## Common false positives

- Single-call benchmarks of LINQ vs for-loop on cold paths — irrelevant if the call frequency is low.
- Async overhead on calls that already do I/O — tiny, irrelevant.
- "Removing a closure" on code that runs once per request.

Reach for the slowest, hottest path first. The 1% that runs 99% of the time.

## Caching design

- Read-through, never write-through. Writes evict, reads populate.
- Cache key: `"<area>:<entity>:<id>"`, lower kebab-case. Centralize in `CacheKeys`.
- TTL explicit, default 5 min, max 1 hour for general data.
- Tag-based invalidation triggered by command handlers via `IInvalidateCache`.
- Cache stampede protection on slow upstreams.

## Output when investigating

```
Hot path identified: <code path>
Suspected cause: <one-line>
Measurement: <before> → <after>  (units: ms p95, allocations/req, etc.)
Fix applied: <what + file:line>
Side effects on other paths: <list or "none">
Benchmark added: <yes — file | no — measured in prod>
```
