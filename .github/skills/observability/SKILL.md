---
name: observability
description: Add structured logging (Serilog), metrics, distributed traces (OpenTelemetry), correlation, and health checks to ASP.NET Core 9 code. Use when instrumenting a new feature, wiring telemetry in `Program.cs`, debugging missing or noisy telemetry, auditing log hygiene for PII/secrets, or reviewing custom `ActivitySource` / `Meter` usage.
---

# Observability

Activates when code touches logs, metrics, traces, health checks, or `Program.cs` telemetry wiring. This is the canonical observability reference — three signals, one trace, no vendor lock-in.

## Three signals, one trace

- **Logs** — Serilog → stdout JSON → OTel Collector.
- **Metrics** — `System.Diagnostics.Metrics` → OTLP exporter.
- **Traces** — `System.Diagnostics.ActivitySource` → OTLP exporter.

All three carry the same `traceparent`. Correlation is automatic — `ASP.NET Core` and `HttpClient` instrumentation propagate it, and `Enrich.WithSpan()` stamps `TraceId` / `SpanId` onto every log.

The collector fans out to vendors (Tempo/Jaeger, Prometheus, Loki, etc.). **Apps never know the vendor.**

## Logging — Serilog wiring

One configuration in `Program.cs`:

```csharp
builder.Host.UseSerilog((ctx, services, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .Enrich.WithProcessId()
    .Enrich.WithThreadId()
    .Enrich.WithCorrelationId()
    .Enrich.WithSpan()
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .WriteTo.OpenTelemetry());
```

- Sinks: **Console (compact JSON)** for stdout, **OpenTelemetry** for the collector. **No file sinks** in containerized envs.
- Health check noise off: `MinimumLevel.Override("Microsoft.AspNetCore.Diagnostics.HealthChecks", LogEventLevel.Warning)`.

## Log levels

| Level | When |
|---|---|
| `Trace` | Local dev only. Never in production. |
| `Debug` | Disabled in production by default. Enable per-namespace via runtime config. |
| `Information` | Successful operations, lifecycle events. The baseline. |
| `Warning` | Recovered errors, deprecations, suspicious conditions. |
| `Error` | Failed operations the system cannot recover from at this layer. |
| `Critical` | Process-level failures requiring immediate attention. |

## Structured logging — every log line

```csharp
_logger.LogInformation(
    "Order {OrderId} placed by {CustomerId} totaling {Amount} {Currency}",
    order.Id, order.CustomerId, order.Total.Amount, order.Total.Currency);
```

- Properties **PascalCase**, scalar where possible. Avoid logging full DTOs.
- Use `LoggerMessage` source generators for hot paths — `LogInformation("...")` allocates a `params` array per call:

```csharp
[LoggerMessage(EventId = 1001, Level = LogLevel.Information, Message = "Order {OrderId} placed by {CustomerId}")]
public partial void OrderPlaced(OrderId orderId, CustomerId customerId);
```

- **Never** `LogInformation($"Order {order.Id} placed")` — interpolation breaks structured logging and allocates.

## What NOT to log

- Secrets, tokens, API keys, passwords, anything in an `Authorization` header.
- Full request/response bodies for `/auth/*`, `/payments/*`, anything with PII.
- PII (email, phone, full name, IP) without redaction; align with privacy policy.
- Stack traces in user-visible responses (yes in logs, no in API output).

## Distributed tracing — OpenTelemetry wiring

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(serviceName: "orders-api", serviceVersion: ThisAssembly.Git.SemVer))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddNpgsql()
        .AddRedisInstrumentation()
        .AddHangfireInstrumentation()
        .AddSource("Orders.Api")
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddMeter("Orders.Api")
        .AddOtlpExporter())
    .WithLogging(l => l.AddOtlpExporter());
```

Service name from `OTEL_SERVICE_NAME` env var or `Resource.Builder().AddService(...)`. Exporter is **OTLP** to the collector — never a vendor SDK.

## Custom Activities

```csharp
private static readonly ActivitySource Source = new("Orders.Api");

public async Task<Result<OrderId>> Handle(PlaceOrderCommand request, CancellationToken ct)
{
    using var activity = Source.StartActivity("Orders.Application.PlaceOrder");
    activity?.SetTag("enduser.id", request.CustomerId);
    // ...
    if (result.IsFailure) activity?.SetStatus(ActivityStatusCode.Error, result.Error.Code);
    return result;
}
```

- **One `ActivitySource` per service** (`new ActivitySource("Orders.Api")`). Don't proliferate sources.
- Span names: `Service.Layer.Operation` — `Orders.Application.PlaceOrder`, `Orders.Infrastructure.Outbox.Publish`.
- Tags follow **OTel semantic conventions** (`db.system`, `db.statement`, `messaging.system`, `enduser.id`, `http.*`). Don't invent ad-hoc keys when a semantic convention exists.
- Record exceptions: `activity?.RecordException(ex); activity?.SetStatus(ActivityStatusCode.Error, ex.Message);`.

## Metrics

```csharp
private static readonly Meter Meter = new("Orders.Api", "1.0.0");
private static readonly Counter<long> OrdersPlaced = Meter.CreateCounter<long>("orders.placed.count");
private static readonly Histogram<double> OrderProcessingMs = Meter.CreateHistogram<double>("orders.processing.duration_ms");
```

- **One `Meter` per bounded context.**
- Names: `<service>.<resource>.<unit>` — `orders.placed.count`, `orders.processing.duration_ms`.
- **Counters** for events, **histograms** for durations, **observable gauges** for sampled values.
- Tag dimensions sparingly — high cardinality blows up storage.
- Built-in metrics are already enabled via the wiring above: `aspnet_core.*`, `http.server.*`, `db.*`, `dotnet.gc.*`, `process.*`.

## Correlation

- `traceparent` propagated automatically via `ASP.NET Core` and `HttpClient` instrumentation.
- All logs auto-include `TraceId` and `SpanId` via `Enrich.WithSpan()`.
- Background work picks up the parent context via `Activity.Current` captured at enqueue and restored at dequeue.

## Health checks

- `/health/live` — process is up.
- `/health/ready` — checks DB, cache, broker. **503** if any dependency is down.
- Reported as a metric: `health.check.status{check="postgres", result="healthy"}`.
- Excluded from log noise (see Serilog override above).

## Sampling

- **Production**: parent-based, **10%** ratio by default; **100% on errors**. Configured via `OTEL_TRACES_SAMPLER`.
- **Local dev**: 100%.
- Never sample at the app level via random gates — let OTel sampling handle it.

## SLOs (default targets)

- API p95 latency: **< 300ms** read, **< 800ms** write.
- Error rate (5xx): **< 0.1%** over rolling 5 min.
- Saturation: CPU **< 70%**, GC pause p99 **< 50ms**.

When a handler approaches its budget, the `PerformanceBehavior` logs a warning with the handler name and elapsed time.

## Forbidden

- `Console.WriteLine` outside of `Program.cs` startup.
- `Trace.WriteLine`, `Debug.WriteLine`.
- `try { ... } catch (Exception ex) { _logger.LogError(ex, "..."); throw; }` — the global exception handler logs once. Re-logging at every layer creates noise.
- Custom log scopes that duplicate `Activity` attributes.
- Sampling at the app level via random gates.
- File sinks in containerized environments.
- Logging full `HttpRequest` / `HttpContext` / DTO payloads.
- Vendor SDKs in app code (Datadog, AppInsights, etc.) — export OTLP, let the collector adapt.

## Output when adding instrumentation

```
Logs added:
  - <event> in <file> — Level: <X>, Properties: <list>
Activities added:
  - <span name> in <file> — Tags: <list>
Metrics added:
  - <metric name> (counter|histogram|gauge) in <file>
PII / secret check: <pass | issues>
```
