---
name: 'Worker Service rules'
description: 'BackgroundService hosts, cron-scheduled jobs, queue consumers, graceful shutdown, and dispatch-only worker discipline for .NET 10 Worker projects.'
applyTo: 'src/Worker/**/*.cs,src/Workers/**/*.cs'
---

# Worker Service rules

The Worker project is a **composition root** that hosts long-running background work — cron-scheduled jobs, queue/outbox consumers, and pollers. It targets **.NET 10** (`Microsoft.NET.Sdk.Worker`). Like the Web project, the Worker dispatches to MediatR and **contains no business logic**.

For dispatch, validation, and `Result` conventions, see [application.instructions.md](./application.instructions.md). For Hangfire fire-and-forget jobs invoked from request handlers, see the Application rules — this file governs **dedicated host processes** built on `BackgroundService` / `IHostedService` / `IHostedLifecycleService`.

## Hard constraints

- Worker depends on **Application**. It depends on **Infrastructure only inside `Program.cs`** for DI wiring — never from a `BackgroundService`, cron job, or consumer.
- No `DbContext`, `IApplicationDbContext`, repository, or `NpgsqlDataSource` in `src/Worker` outside `Program.cs`. Background work loads aggregates by dispatching MediatR commands/queries.
- No business rules. Workers **schedule** and **dispatch** — handlers do the work, return `Result`.
- Every long-running operation accepts `CancellationToken` and propagates it to `ISender.Send`. Honour `stoppingToken` from `BackgroundService.ExecuteAsync` so SIGTERM / `IHostApplicationLifetime.StopApplication()` shuts down cleanly within `HostOptions.ShutdownTimeout`.
- Time is `TimeProvider`. **Never** `DateTime.UtcNow`, `DateTime.Now`, or `Stopwatch.StartNew()` for scheduling. Cron evaluation and delays use `TimeProvider.GetUtcNow()` and the `Task.Delay(delay, TimeProvider, ct)` overload so `FakeTimeProvider` can drive tests deterministically.

## Project layout

```
src/Worker/
  Jobs/
    SendOverdueInvoicesJob.cs       ← CronJob, one file per scheduled job
    ReconcilePaymentsJob.cs
  Consumers/
    OrdersOutboxConsumer.cs         ← BackgroundService draining a Channel<T> / queue
  Scheduling/
    CronJob.cs                      ← base class wrapping the cron loop
    CronSchedule.cs                 ← Cronos.CronExpression wrapper
    ICronJob.cs
  HealthChecks/
    JobHeartbeatHealthCheck.cs
  Extensions/
    DependencyInjection.cs          ← AddWorkerServices() + AddCronJob<T>()
  Program.cs
```

One job per file. Group by feature folder when a domain area has more than two jobs.

## Cron jobs — the canonical pattern

Cron parsing uses **`Cronos`** (`Cronos.CronExpression`). It supports both 5-field (`min hour day month dow`) and 6-field (`sec min hour day month dow`) expressions. Use the 6-field form only when seconds-precision is genuinely required — minute precision covers >95% of real schedules.

```csharp
internal sealed partial class SendOverdueInvoicesJob(
    ISender sender,
    IOptionsMonitor<JobsOptions> options,
    ILogger<SendOverdueInvoicesJob> logger)
    : CronJob(logger)
{
    public override CronSchedule Schedule =>
        CronSchedule.Parse(options.CurrentValue.SendOverdueInvoices.Cron);

    protected override async Task ExecuteOccurrenceAsync(DateTimeOffset firedAt, CancellationToken ct)
    {
        var command = new SendOverdueInvoicesCommand(firedAt);
        var result = await sender.Send(command, ct);

        if (result.IsFailure)
            JobFailed(Logger, nameof(SendOverdueInvoicesJob), result.Error.Code, result.Error.Message);
    }

    [LoggerMessage(EventId = 4001, Level = LogLevel.Warning,
        Message = "Job {Job} failed with {ErrorCode}: {ErrorMessage}")]
    private static partial void JobFailed(ILogger logger, string job, string errorCode, string errorMessage);
}
```

- `internal sealed`. Cron jobs are wired by `AddCronJob<T>()`, not consumed directly.
- Constructor injection only. Primary constructors preferred.
- `ExecuteOccurrenceAsync` runs **once per scheduled occurrence**. The base class owns the loop.
- The job dispatches a MediatR command; the handler does the work and returns `Result`. The worker logs the structured failure — it does **not** throw to convert.

## CronJob base class — what it must guarantee

A correct `CronJob` base implementation:

1. Computes the **next UTC occurrence** via `CronSchedule.GetNextOccurrence(TimeProvider.GetUtcNow())`.
2. Sleeps via `Task.Delay(delay, TimeProvider, stoppingToken)` — never the two-argument overload, so `FakeTimeProvider.Advance(...)` works in tests.
3. Wraps each occurrence in its own scope: `IServiceScopeFactory.CreateAsyncScope()` — handlers are scoped; do not capture them in the singleton job instance.
4. Catches `Exception` at the per-occurrence boundary, logs structured (`Job`, `OccurrenceUtc`, `DurationMs`, `ExceptionType`), and continues. **One failed occurrence must not tear down the host.**
5. Treats `OperationCanceledException` propagated from `stoppingToken` as graceful shutdown — re-throw, do not log as an error.
6. Records every occurrence to an in-memory heartbeat (`LastOccurrenceUtc`, `LastOutcome`) read by `JobHeartbeatHealthCheck`.

## Registration

```csharp
services.AddCronJob<SendOverdueInvoicesJob>();
services.AddCronJob<ReconcilePaymentsJob>();
```

`AddCronJob<T>` registers the job as a singleton (so the heartbeat is observable) **and** as `IHostedService`. Do not call `AddHostedService<T>` for cron jobs directly — go through the extension so discovery, heartbeats, and metrics stay consistent.

## Distributed scheduling — at most once per occurrence

When the Worker runs in **more than one replica** (the default in production), an unguarded cron job fires on every replica. Pick **one** of:

- **Database lease** via the Application abstraction `IDistributedLock` (Postgres advisory lock, key = `cron:<JobName>:<OccurrenceUtc:O>`). Acquire before `ExecuteOccurrenceAsync`; if not acquired, skip with a debug log.
- **Hangfire `RecurringJob`** — when the job fits Hangfire's model and the dashboard is desired. The Worker hosts the Hangfire server; recurring jobs are registered from `Program.cs`.

`RecurringJob` and `CronJob` are mutually exclusive for a given logical job. Pick one and put a one-line comment on the chosen registration explaining why.

## Queue / outbox consumers

A consumer is a `BackgroundService` that drains a queue. Required shape:

- Bounded `Channel<T>` with a documented capacity. The producer is the Application abstraction (`IOutboxQueue.Enqueue(...)`); the consumer lives here.
- `await foreach (var msg in channel.Reader.ReadAllAsync(stoppingToken))` is the loop. Never `while (true) { TryRead; Task.Delay; }`.
- Per-message scope, per-message try/catch, structured log on failure, dead-letter on poison messages after N attempts.
- Backpressure: when the downstream is failing (Polly circuit broken), pause with backoff rather than draining into a black hole.

## Composition root — `Program.cs`

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services
    .AddDomain()
    .AddApplication()
    .AddInfrastructure(builder.Configuration)
    .AddWorkerServices(builder.Configuration);

builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(30);
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost;
});

var host = builder.Build();
await host.RunAsync();
```

- One `AddXxx` extension per layer. Worker-specific wiring (cron jobs, consumers, options binding) lives in `Extensions/DependencyInjection.cs`.
- `Program.cs` is the **only** place Worker references Infrastructure.
- `BackgroundServiceExceptionBehavior.StopHost` makes silent host bugs loud — the orchestrator restarts the pod rather than running a zombie process.
- Configuration binding via `IOptions<T>` with `ValidateOnStart()` + `ValidateDataAnnotations()`. The options validator parses every cron expression (`CronExpression.Parse(...)`) — fail fast on a bad schedule, never at the first scheduled fire-time.

## Configuration shape

```jsonc
{
  "Jobs": {
    "SendOverdueInvoices": {
      "Cron": "30 2 * * *",          // 02:30 UTC daily (5-field)
      "Enabled": true
    },
    "ReconcilePayments": {
      "Cron": "0 */15 * * * *",      // every 15 min on the second 0 (6-field)
      "Enabled": true
    }
  }
}
```

- Schedules live in configuration, not hard-coded constants. `CronJob.Schedule` reads from `IOptionsMonitor<JobsOptions>` so operations can reschedule by config without a redeploy.
- `Enabled: false` short-circuits `ExecuteOccurrenceAsync` to a no-op debug log. Use it to disable a job per environment without removing the registration.
- All cron expressions are evaluated in **UTC**. Per-job timezone overrides go through `CronSchedule.Parse(expr, TimeZoneInfo)` — never adjust `DateTime` values manually.

## Observability

- `Activity` per occurrence: `ActivitySource.StartActivity($"Worker.Cron.{JobName}", ActivityKind.Internal)`. Tags: `cron.expression`, `cron.fired_at`, `cron.next_at`.
- Metrics: counter `worker.cron.runs_total{job, outcome}`, histogram `worker.cron.duration_ms{job}`, gauge `worker.cron.last_success_age_seconds{job}`.
- Structured logs via `LoggerMessage` source generators. Standard properties: `Job`, `OccurrenceUtc`, `DurationMs`, `Outcome`.
- The heartbeat health check reads the last-success-age gauge. If a job has missed its expected window by > 2× its interval, `/health/ready` reports degraded.

## Health checks

- Worker exposes a minimal HTTP listener (or sidecar) for `/health/live` and `/health/ready` only — no application endpoints.
- `/health/live` — process is up.
- `/health/ready` — dependency pings (Postgres, Redis, message broker) tagged `"ready"` plus `JobHeartbeatHealthCheck` for cron freshness.

## Testing

- **Unit-test cron jobs** with `FakeTimeProvider`: advance time across an occurrence boundary, assert `ISender.Send` was called once with the expected command.
- **Integration tests** use `Host.CreateApplicationBuilder` with the real DI graph but `FakeTimeProvider` and Testcontainers for Postgres/Redis.
- Never test "the job runs eventually" with `Thread.Sleep` or real wall-clock waits. If you cannot make it deterministic with `FakeTimeProvider`, the job has hidden timing dependencies — fix the design.
- Coverage gate (80% line / 70% branch) applies to Application/Domain only; the Worker project itself is composition and is exempt.

## Forbidden

- Business logic, validation rules, or domain decisions inside a `BackgroundService` or `CronJob`. The job dispatches and logs — that is all.
- `DbContext`, `IApplicationDbContext`, repositories, or `NpgsqlDataSource` referenced anywhere in `src/Worker` outside `Program.cs`.
- `DateTime.UtcNow` / `DateTime.Now` / `Stopwatch` for scheduling. Use `TimeProvider`.
- `Thread.Sleep`. `Task.Delay(timeout)` or `Task.Delay(timeout, ct)` without the `TimeProvider` overload — the three-argument form is mandatory for testability.
- `async void` methods (event handlers excepted). Unhandled `async void` exceptions tear the host down outside `BackgroundServiceExceptionBehavior`.
- Catching `OperationCanceledException` and swallowing it during shutdown — re-throw so the host can complete cleanly.
- Registering a cron job that mutates state in two replicas without a distributed lock or Hangfire — every cron job must be **at most once per occurrence, cluster-wide**.
- Hard-coded cron expressions in code — schedules belong in `IOptions`-bound configuration, validated at startup.
- Coupling a cron job and a queue consumer in the same class — cron drains a schedule, consumers drain a queue. Different lifecycles, different failure modes, different files.
- Logging the full payload of a dispatched command (PII risk). Log identifiers and outcomes only.
