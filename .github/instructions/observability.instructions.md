---
name: 'Observability conventions'
description: 'Logging (Serilog), metrics, distributed tracing (OpenTelemetry), and correlation conventions.'
applyTo: '**/*.cs,src/Web/Program.cs,src/Web/appsettings*.json'
---

# Observability conventions

The system has three signals: **traces, metrics, logs**. They share correlation via `traceparent`. Every operation is observable end-to-end.

## Logging — Serilog

- One configuration in `Program.cs` via `builder.Host.UseSerilog((ctx, services, cfg) => cfg.ReadFrom.Configuration(ctx.Configuration).ReadFrom.Services(services).Enrich.FromLogContext()...)`.
- Sinks: **Console (JSON)** for stdout, **OpenTelemetry** for collector. No file sinks in containerized envs.
- Enrichers: `WithMachineName`, `WithEnvironmentName`, `WithProcessId`, `WithThreadId`, `WithCorrelationId`, `WithSpan`.
- Format: `Serilog.Formatting.Compact.RenderedCompactJsonFormatter` to stdout.

## Log levels

- `Trace` — never in production. Local dev only.
- `Debug` — disabled in production by default. Enable per-namespace via runtime config.
- `Information` — successful operations, lifecycle events. Should be the baseline.
- `Warning` — recovered errors, deprecations, suspicious conditions.
- `Error` — failed operations the system cannot recover from at this layer.
- `Critical` — process-level failures requiring immediate attention.

## Structured logging — always

```csharp
_logger.LogInformation(
    "Order {OrderId} placed by {CustomerId} totaling {Amount} {Currency}",
    order.Id, order.CustomerId, order.Total.Amount, order.Total.Currency);
```

- Properties are PascalCase, scalar where possible. Avoid logging full DTOs.
- Use `LoggerMessage` source generators for hot paths:
  ```csharp
  [LoggerMessage(EventId = 1001, Level = LogLevel.Information, Message = "Order {OrderId} placed by {CustomerId}")]
  public partial void OrderPlaced(OrderId orderId, CustomerId customerId);
  ```
- Never `LogInformation($"Order {order.Id} placed")` (breaks structure, allocates).

## What NOT to log

- Secrets, tokens, API keys, passwords, anything in an `Authorization` header.
- Full request/response bodies for `/auth`, `/payments`, anything with PII.
- Stack traces in user-visible responses (yes in logs, no in API output).

## Distributed tracing — OpenTelemetry

- `AddOpenTelemetry().WithTracing(...).WithMetrics(...).WithLogging(...)` in `Program.cs`.
- Instrumentation: `AddAspNetCoreInstrumentation`, `AddHttpClientInstrumentation`, `AddEntityFrameworkCoreInstrumentation`, `AddNpgsql`, `AddRedisInstrumentation`, `AddHangfireInstrumentation`.
- Exporter: **OTLP** to the collector. The collector fans out to vendors (Tempo/Jaeger, Prometheus, Loki, etc.). Apps **never** know the vendor.
- Service name from `OTEL_SERVICE_NAME` or `Resource.Builder().AddService("orders-api", serviceVersion: ThisAssembly.Git.SemVer)`.

## Custom Activities

- Source name: one per service (`new ActivitySource("Orders.Api")`).
- Span names: `Service.Layer.Operation` — `Orders.Application.PlaceOrder`, `Orders.Infrastructure.Outbox.Publish`.
- Tags use OTel semantic conventions: `db.system`, `db.statement`, `messaging.system`, `enduser.id`, etc. Do not invent ad-hoc keys when a semantic convention exists.
- Record exceptions: `activity?.RecordException(ex); activity?.SetStatus(ActivityStatusCode.Error, ex.Message);`.

## Metrics

- One `Meter` per bounded context (`new Meter("Orders.Api", "1.0.0")`).
- Names: `<service>.<resource>.<unit>` — `orders.placed.count`, `orders.processing.duration_ms`.
- Use **counters** for events, **histograms** for durations, **observable gauges** for sampled values.
- Built-in metrics enabled: `aspnet_core.*`, `http.server.*`, `db.*`, `dotnet.gc.*`, `process.*`.

## Correlation

- `traceparent` header propagated by default via `ASP.NET Core` and `HttpClient` instrumentation.
- All logs auto-include `TraceId` and `SpanId` via `Enrich.WithSpan()`.
- Background work picks up the parent context via `Activity.Current` captured at enqueue time and restored at dequeue.

## Health checks

- Exposed at `/health/live` and `/health/ready`.
- Reported as a metric: `health.check.status{check="postgres", result="healthy"}`.
- Excluded from logging noise via `Serilog` `MinimumLevel.Override("Microsoft.AspNetCore.Diagnostics.HealthChecks", LogEventLevel.Warning)`.

## Sampling

- Production: parent-based, ratio 10% by default; 100% on errors. Configured via `OTEL_TRACES_SAMPLER`.
- Local dev: 100%.

## SLOs (default targets)

- API p95 latency < 300ms for read endpoints, < 800ms for writes.
- Error rate (5xx) < 0.1% over rolling 5min.
- Saturation: CPU < 70%, GC pause p99 < 50ms.

When a handler approaches its budget, the `PerformanceBehavior` logs a warning with the handler name and elapsed time.

## Forbidden

- `Console.WriteLine` outside of `Program.cs` startup.
- `Trace.WriteLine`, `Debug.WriteLine`.
- `try { ... } catch (Exception ex) { _logger.LogError(ex, "..."); throw; }` — the global exception handler logs once. Re-logging at every layer creates noise.
- Custom log scopes that duplicate `Activity` attributes.
- Sampling at the app level via random gates — let OTel sampling handle it.
