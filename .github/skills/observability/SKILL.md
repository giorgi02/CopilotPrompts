---
name: observability
description: Add structured logging (Serilog), metrics, traces (OpenTelemetry), and correlation to code. Use when instrumenting a new feature, debugging missing telemetry, or auditing log hygiene for PII/secrets.
---

# Observability

Activates when adding telemetry to code or reviewing telemetry in a PR. See [.github/instructions/observability.instructions.md](../../instructions/observability.instructions.md).

## Three signals, one trace

- **Logs** (Serilog → stdout JSON → OTel Collector).
- **Metrics** (`System.Diagnostics.Metrics` → OTel exporter).
- **Traces** (`System.Diagnostics.ActivitySource` → OTel exporter).

All three carry the same `traceparent`. Correlation is automatic.

## Logging — every log line

- Structured. Properties PascalCase, scalar where possible.
- `LoggerMessage` source generator for hot paths.
- No interpolation in the message: `LogInformation("Order {OrderId} placed", orderId)`, never `LogInformation($"Order {orderId} placed")`.
- Levels: `Information` for lifecycle / success, `Warning` for recoverable issues, `Error` for unrecoverable at this layer, `Critical` for process-level.

```csharp
[LoggerMessage(EventId = 1001, Level = LogLevel.Information, Message = "Order {OrderId} placed by {CustomerId}")]
public partial void OrderPlaced(OrderId orderId, CustomerId customerId);
```

## What NOT to log

- Secrets, tokens, API keys, anything from an `Authorization` header.
- Full request bodies for `/auth/*`, `/payments/*`.
- PII — email, phone, full name, IP — without redaction; align with privacy policy.
- Exception messages or stack traces in user-visible responses (yes in logs, no in API output).

## Custom Activities

```csharp
private static readonly ActivitySource Source = new("Orders.Api");

public async Task<Result<OrderId>> Handle(PlaceOrderCommand request, CancellationToken ct)
{
    using var activity = Source.StartActivity("Orders.Application.PlaceOrder");
    activity?.SetTag("customer.id", request.CustomerId);
    // ...
    if (result.IsFailure) activity?.SetStatus(ActivityStatusCode.Error, result.Error.Code);
    return result;
}
```

- Span name: `Service.Layer.Operation`.
- Tags follow OTel semantic conventions where they exist (`db.system`, `messaging.system`, `enduser.id`).
- Record exceptions: `activity?.RecordException(ex); activity?.SetStatus(ActivityStatusCode.Error, ex.Message);`.

## Metrics

```csharp
private static readonly Meter Meter = new("Orders.Api", "1.0.0");
private static readonly Counter<long> OrdersPlaced = Meter.CreateCounter<long>("orders.placed.count");
private static readonly Histogram<double> OrderProcessingMs = Meter.CreateHistogram<double>("orders.processing.duration_ms");
```

- Names: `<service>.<resource>.<unit>`.
- Counters for events, histograms for durations, observable gauges for sampled values.
- Tag dimensions sparingly — high cardinality blows up storage.

## Built-in instrumentation (configured once in Program.cs)

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(serviceName: "orders-api", serviceVersion: ThisAssembly.Git.SemVer))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddNpgsql()
        .AddRedisInstrumentation()
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

## Health checks

- `/health/live` and `/health/ready`.
- `/health/ready` checks DB, cache, broker. 503 if any down.
- Reported as a metric for alerting.

## SLOs (defaults)

- API p95 latency: < 300ms read, < 800ms write.
- Error rate (5xx): < 0.1% over 5 min.
- GC pause p99: < 50ms.

`PerformanceBehavior` warns when a handler exceeds its budget.

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
