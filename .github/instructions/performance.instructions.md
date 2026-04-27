---
name: 'Performance conventions'
description: 'Allocation, async, caching, and database performance rules applied to all C# code.'
applyTo: '**/*.cs'
---

# Performance conventions

The system targets:
- API p95 < 300ms (read), < 800ms (write).
- < 50 allocations per request on hot paths.
- GC pause p99 < 50ms.
- DB round-trips per request: 1 ideal, 2–3 acceptable, > 5 needs justification.

## Database — the #1 source of perf bugs

- **Always** `AsNoTracking()` on reads.
- **Always** project to DTO with `Select(...)` — never `ToListAsync().Select(...)`.
- **No N+1** — when iterating, the query has loaded everything you need (`Include`, `Select` projection, or batched query).
- **Limit included collections** with `AsSplitQuery()` for cardinality > 1.
- **Use `Take(n)`** on every query that could return unbounded results.
- **Index every FK** and every `WHERE`/`ORDER BY` column on hot queries.
- **Use Dapper** for read-heavy hot paths where EF translation is suboptimal — measured, not guessed.

## Async hygiene

- `cancellationToken` flows through every awaited call.
- Never block on async (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`). The build fails on these via analyzer.
- Avoid `Task.Run` to escape sync context — handlers are already async.
- Prefer `await foreach` over materializing `IAsyncEnumerable` to a list.
- `Task.WhenAll` for independent calls; never sequential awaits when you can parallelize.

## Allocations

- Use `string.Create`, `ArrayPool<T>`, `MemoryPool<T>`, `Span<T>`, `ReadOnlySpan<T>` on hot paths.
- `StringBuilder` for any concatenation in a loop.
- Prefer `ReadOnlySpan<char>` parsing over `Substring`.
- Use `LoggerMessage` source generators on hot logging paths — `LogInformation("...")` allocates an array for parameters.
- Avoid LINQ on hot paths (allocates closures and enumerators). `for` / `foreach` over arrays.
- Use `record struct` for small immutable values to avoid heap allocation.

## Collections

- Pre-size `List<T>`, `Dictionary<K,V>` when count is known: `new List<T>(capacity: 100)`.
- Use `FrozenDictionary<K,V>` / `FrozenSet<T>` for read-mostly lookup tables.
- `ImmutableArray<T>` for shared immutable data.

## Caching

- `HybridCache` (in-memory L1 + Redis L2) by default.
- Cache **read-through**, never write-through (write-through invalidation is too easy to get wrong).
- Every cache entry has an explicit TTL. Default 5 minutes, max 1 hour for general data.
- Tag-based invalidation triggered by command handlers via `IInvalidateCache`.
- Cache stampede protection: `HybridCache` handles it; for `IDistributedCache` directly, use a `SemaphoreSlim` keyed on cache key.

## HTTP clients

- `IHttpClientFactory` typed clients only. Never `new HttpClient()`.
- Connection limits tuned via `SocketsHttpHandler.MaxConnectionsPerServer`.
- Timeout configured at the client and at the Polly pipeline (defense in depth).
- Use compressed responses where the upstream supports it.

## Serialization

- `System.Text.Json` only. Never `Newtonsoft.Json` in new code.
- Source-generated `JsonSerializerContext` for hot serialization paths (response DTOs).
- `JsonSerializerOptions` reused (cache as singleton). Never `new JsonSerializerOptions()` per call.

## Background work

- Long-running CPU work uses `IHostedService` with a `Channel<T>` queue, **not** an HTTP request.
- Recurring jobs use Hangfire with explicit lock timeouts and idempotency.
- Never `Task.Run(() => ...)` then forget — fire-and-forget swallows exceptions and dies on app shutdown.

## Threading

- Avoid `lock`, prefer `SemaphoreSlim` for async-friendly mutual exclusion.
- Use the new `Lock` type (.NET 9) for sync-only critical sections.
- Never `Thread.Sleep` in async code — `await Task.Delay`.

## Measuring before optimizing

- Suspected hot path: write a `BenchmarkDotNet` benchmark in `tests/Performance.Tests/` first.
- Suspected memory pressure: capture with `dotnet-trace` and `PerfView`.
- Suspected DB issue: enable `EnableSensitiveDataLogging` locally + capture `pg_stat_statements`.
- Don't merge a perf change without a before/after number.

## Forbidden patterns

- `IEnumerable<T>` returned from a hot method (forces re-enumeration).
- `Lazy<T>` used to avoid an `await` (just `await` it).
- `async Task` wrapping a synchronous method.
- `string.Format` / interpolation for log messages (breaks structured logging and allocates).
- Multiple `await` calls on independent operations sequentially when `Task.WhenAll` is available.
- Converting `IQueryable` to `IEnumerable` mid-pipeline (silently drops to client-side eval).
