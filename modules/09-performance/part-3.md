# Part 3 — Observability Deep Dive

## Why This Matters

Monitoring tells you **something is wrong**. Observability tells you **why**.

```
Monitoring: "API latency is high"
Observability: "API latency is high because the order-service is
               waiting on inventory-service, which is blocked on
               a slow database query in tenant-123's partition"
```

In production, you can't attach a debugger. Your observability system IS your debugger.

---

## What Goes Wrong Without This

### The Phantom Slowdown

```
Alert: "Homepage taking 5 seconds to load"
Team: "Which service?"
Logs: 47 different services involved
Each service: "Not me, I'm fast"
Reality: 200ms + 200ms + 200ms... across 20 services = 4 seconds

Without distributed tracing, you're blind to the cascade.
```

### The Mystery Errors

```
Customer: "My order failed"
Support: "When?"
Customer: "Yesterday around 3pm"
Engineer: *searches logs*

Logs from Service A: Nothing unusual
Logs from Service B: "Error processing order" (no context)
Logs from Service C: No logs at all

Three hours later: Found it was a timeout in Service D,
triggered by a deployment in Service E, which caused
resource exhaustion in Service F.
```

### The Alert Storm

```
2:03 AM - Alert: Database latency high
2:03 AM - Alert: API error rate elevated
2:03 AM - Alert: Payment service unhealthy
2:03 AM - Alert: Order processing delayed
2:03 AM - Alert: Cache hit rate dropped
2:04 AM - Alert: Worker queue growing
2:04 AM - Alert: Memory usage high
... 47 more alerts ...

On-call engineer: "What's actually broken?"
```

---

## The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│                      OBSERVABILITY                          │
├─────────────────┬─────────────────┬─────────────────────────┤
│     LOGS        │    METRICS      │       TRACES            │
├─────────────────┼─────────────────┼─────────────────────────┤
│ What happened   │ What's the      │ How did the request     │
│ at this moment  │ aggregate state │ flow through services   │
├─────────────────┼─────────────────┼─────────────────────────┤
│ High cardinality│ Low cardinality │ High cardinality        │
│ High volume     │ Low volume      │ Sampled volume          │
├─────────────────┼─────────────────┼─────────────────────────┤
│ Debug specific  │ Alert on trends │ Understand              │
│ issues          │                 │ dependencies            │
└─────────────────┴─────────────────┴─────────────────────────┘
```

---

## OpenTelemetry Setup

### Why OpenTelemetry?

- **Vendor-neutral**: Works with any backend (Jaeger, Zipkin, Datadog, etc.)
- **Standardized**: One API for traces, metrics, and logs
- **Automatic instrumentation**: HTTP, database, messaging out of the box
- **Future-proof**: Industry standard, wide adoption

### Installing OpenTelemetry

```bash
# Core packages
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Extensions.Hosting

# Automatic instrumentation
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add package OpenTelemetry.Instrumentation.StackExchangeRedis

# Exporters
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore
```

### Basic Configuration

```csharp
// Program.cs
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

// Define service resource
var serviceName = "OrderFlow.Api";
var serviceVersion = typeof(Program).Assembly.GetName().Version?.ToString() ?? "1.0.0";

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: serviceName,
            serviceVersion: serviceVersion,
            serviceInstanceId: Environment.MachineName)
        .AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment"] = builder.Environment.EnvironmentName,
            ["host.name"] = Environment.MachineName
        }))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation(options =>
        {
            options.RecordException = true;
            options.Filter = context =>
                !context.Request.Path.StartsWithSegments("/health");
        })
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation(options =>
        {
            options.SetDbStatementForText = true;
            options.SetDbStatementForStoredProcedure = true;
        })
        .AddSource("OrderFlow.*")  // Custom activity sources
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"]!);
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddMeter("OrderFlow.*")  // Custom meters
        .AddPrometheusExporter());

// Configure logging to include trace context
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
    logging.ParseStateValues = true;
    logging.AddOtlpExporter(options =>
    {
        options.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"]!);
    });
});
```

---

## Distributed Tracing

### Understanding Trace Context

```
Trace: End-to-end request journey
├── Span: API Gateway (100ms total)
│   ├── Span: Authentication (10ms)
│   └── Span: Route to Service (5ms)
├── Span: Order Service (80ms total)
│   ├── Span: Validate Request (5ms)
│   ├── Span: Check Inventory (30ms)  ← HTTP call
│   └── Span: Save Order (40ms)       ← Database
└── Span: Inventory Service (25ms)
    └── Span: Query Stock (20ms)      ← Database
```

### Custom Instrumentation

```csharp
using System.Diagnostics;

public class OrderService
{
    // Create an ActivitySource for your service
    private static readonly ActivitySource ActivitySource = new("OrderFlow.OrderService");

    private readonly IInventoryClient _inventoryClient;
    private readonly OrderDbContext _context;
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(CreateOrderCommand command)
    {
        // Start a custom span
        using var activity = ActivitySource.StartActivity("CreateOrder");

        // Add attributes (tags) for context
        activity?.SetTag("order.customer_id", command.CustomerId);
        activity?.SetTag("order.line_count", command.Lines.Count);
        activity?.SetTag("order.tenant_id", command.TenantId);

        try
        {
            // Child span for validation
            using (var validationActivity = ActivitySource.StartActivity("ValidateOrder"))
            {
                await ValidateOrderAsync(command);
                validationActivity?.SetTag("validation.passed", true);
            }

            // Child span for inventory check
            using (var inventoryActivity = ActivitySource.StartActivity("CheckInventory"))
            {
                var availability = await _inventoryClient.CheckAvailabilityAsync(
                    command.Lines.Select(l => l.ProductId).ToList());

                inventoryActivity?.SetTag("inventory.all_available", availability.AllAvailable);

                if (!availability.AllAvailable)
                {
                    activity?.SetStatus(ActivityStatusCode.Error, "Insufficient inventory");
                    throw new InsufficientInventoryException(availability.UnavailableItems);
                }
            }

            // Child span for persistence
            using (var saveActivity = ActivitySource.StartActivity("SaveOrder"))
            {
                var order = new Order(command);
                _context.Orders.Add(order);
                await _context.SaveChangesAsync();

                saveActivity?.SetTag("order.id", order.Id);
                activity?.SetTag("order.id", order.Id);

                // Add an event (annotation)
                activity?.AddEvent(new ActivityEvent("OrderCreated", tags: new ActivityTagsCollection
                {
                    { "order.total", order.Total }
                }));

                return order;
            }
        }
        catch (Exception ex)
        {
            // Record the exception
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }
}
```

### Trace Context Propagation

```csharp
// Propagate trace context through message queues
public class OrderCreatedPublisher
{
    private readonly IBus _bus;

    public async Task PublishAsync(OrderCreatedEvent @event)
    {
        // MassTransit automatically propagates trace context
        await _bus.Publish(@event);
    }
}

// For manual propagation (e.g., custom HTTP calls)
public class InventoryClient
{
    private readonly HttpClient _client;

    public async Task<AvailabilityResponse> CheckAvailabilityAsync(List<int> productIds)
    {
        var request = new HttpRequestMessage(HttpMethod.Post, "/api/availability")
        {
            Content = JsonContent.Create(productIds)
        };

        // Trace context is automatically injected by OpenTelemetry HTTP instrumentation
        var response = await _client.SendAsync(request);
        return await response.Content.ReadFromJsonAsync<AvailabilityResponse>()!;
    }
}

// For background jobs, manually restore context
public class OrderProcessingConsumer : IConsumer<OrderCreated>
{
    private static readonly ActivitySource ActivitySource = new("OrderFlow.Worker");

    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        // MassTransit restores the trace context automatically
        using var activity = ActivitySource.StartActivity("ProcessOrder");
        activity?.SetTag("order.id", context.Message.OrderId);

        // Your processing logic here
    }
}
```

---

## Metrics

### The Four Golden Signals

Google's SRE book defines these as the essential metrics:

1. **Latency**: Time to service a request
2. **Traffic**: Requests per second
3. **Errors**: Rate of failed requests
4. **Saturation**: How "full" the service is

### Custom Metrics Implementation

```csharp
using System.Diagnostics.Metrics;

public class OrderMetrics
{
    private readonly Counter<long> _ordersCreated;
    private readonly Counter<long> _ordersFailed;
    private readonly Histogram<double> _orderProcessingDuration;
    private readonly ObservableGauge<int> _pendingOrders;
    private readonly OrderDbContext _context;

    public OrderMetrics(IMeterFactory meterFactory, OrderDbContext context)
    {
        _context = context;

        var meter = meterFactory.Create("OrderFlow.Orders");

        _ordersCreated = meter.CreateCounter<long>(
            "orderflow.orders.created",
            unit: "{orders}",
            description: "Number of orders created");

        _ordersFailed = meter.CreateCounter<long>(
            "orderflow.orders.failed",
            unit: "{orders}",
            description: "Number of failed order creations");

        _orderProcessingDuration = meter.CreateHistogram<double>(
            "orderflow.orders.processing_duration",
            unit: "ms",
            description: "Order processing duration in milliseconds");

        _pendingOrders = meter.CreateObservableGauge(
            "orderflow.orders.pending",
            () => GetPendingOrderCount(),
            unit: "{orders}",
            description: "Current number of pending orders");
    }

    public void RecordOrderCreated(string tenantId, string region)
    {
        _ordersCreated.Add(1,
            new KeyValuePair<string, object?>("tenant_id", tenantId),
            new KeyValuePair<string, object?>("region", region));
    }

    public void RecordOrderFailed(string tenantId, string reason)
    {
        _ordersFailed.Add(1,
            new KeyValuePair<string, object?>("tenant_id", tenantId),
            new KeyValuePair<string, object?>("failure_reason", reason));
    }

    public void RecordProcessingDuration(double durationMs, string orderType)
    {
        _orderProcessingDuration.Record(durationMs,
            new KeyValuePair<string, object?>("order_type", orderType));
    }

    private int GetPendingOrderCount()
    {
        return _context.Orders.Count(o => o.Status == OrderStatus.Pending);
    }
}

// Register as singleton
services.AddSingleton<OrderMetrics>();
```

### Using Metrics in Services

```csharp
public class OrderService
{
    private readonly OrderMetrics _metrics;
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(CreateOrderCommand command)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            var order = await CreateOrderInternalAsync(command);

            stopwatch.Stop();
            _metrics.RecordOrderCreated(command.TenantId, command.Region);
            _metrics.RecordProcessingDuration(stopwatch.ElapsedMilliseconds, command.OrderType);

            return order;
        }
        catch (ValidationException ex)
        {
            _metrics.RecordOrderFailed(command.TenantId, "validation");
            throw;
        }
        catch (InsufficientInventoryException ex)
        {
            _metrics.RecordOrderFailed(command.TenantId, "inventory");
            throw;
        }
        catch (Exception ex)
        {
            _metrics.RecordOrderFailed(command.TenantId, "unknown");
            throw;
        }
    }
}
```

---

## Structured Logging

### Why Structured Logs?

```csharp
// Bad: Unstructured log
_logger.LogInformation($"Order {orderId} created for customer {customerId} with total ${total}");
// Result: "Order 123 created for customer 456 with total $99.99"
// Can't query: "Show me all orders over $100" or "All orders for customer 456"

// Good: Structured log
_logger.LogInformation(
    "Order created: {OrderId} for {CustomerId} with total {Total}",
    orderId, customerId, total);
// Result: { "OrderId": 123, "CustomerId": 456, "Total": 99.99, "Message": "Order created..." }
// Now you can query!
```

### Logging Best Practices

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(CreateOrderCommand command)
    {
        // Use scopes for context that applies to multiple log entries
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["TenantId"] = command.TenantId,
            ["CustomerId"] = command.CustomerId,
            ["CorrelationId"] = Activity.Current?.Id ?? Guid.NewGuid().ToString()
        }))
        {
            _logger.LogDebug("Starting order creation with {LineCount} items", command.Lines.Count);

            try
            {
                var order = await ProcessOrderAsync(command);

                // Include relevant data, but not sensitive data
                _logger.LogInformation(
                    "Order {OrderId} created successfully. Total: {OrderTotal}, Items: {ItemCount}",
                    order.Id,
                    order.Total,
                    order.Lines.Count);

                return order;
            }
            catch (ValidationException ex)
            {
                // Log warning for expected failures
                _logger.LogWarning(ex,
                    "Order validation failed: {ValidationErrors}",
                    string.Join(", ", ex.Errors));
                throw;
            }
            catch (Exception ex)
            {
                // Log error for unexpected failures
                _logger.LogError(ex,
                    "Unexpected error creating order. ProductIds: {ProductIds}",
                    command.Lines.Select(l => l.ProductId));
                throw;
            }
        }
    }
}
```

### Correlation IDs

```csharp
// Middleware to ensure correlation ID exists
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationIdHeader = "X-Correlation-Id";

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Activity.Current?.Id
            ?? Guid.NewGuid().ToString();

        context.Response.Headers[CorrelationIdHeader] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}
```

---

## Dashboards

### Grafana Dashboard Design

```json
{
  "title": "OrderFlow API Overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_server_request_duration_seconds_count[5m])) by (http_route)",
          "legendFormat": "{{http_route}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_server_request_duration_seconds_count{http_status_code=~\"5..\"}[5m])) / sum(rate(http_server_request_duration_seconds_count[5m])) * 100",
          "legendFormat": "Error %"
        }
      ]
    },
    {
      "title": "P95 Latency by Endpoint",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_server_request_duration_seconds_bucket[5m])) by (le, http_route))",
          "legendFormat": "{{http_route}}"
        }
      ]
    },
    {
      "title": "Orders Created",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(increase(orderflow_orders_created[24h]))",
          "legendFormat": "24h Total"
        }
      ]
    }
  ]
}
```

### Essential Dashboard Panels

```yaml
# dashboard-config.yaml
dashboards:
  api-overview:
    row-1-traffic:
      - requests_per_second:
          query: sum(rate(http_requests_total[5m]))
      - active_connections:
          query: sum(http_server_active_requests)
      - throughput_bytes:
          query: sum(rate(http_response_bytes_total[5m]))

    row-2-latency:
      - p50_latency:
          query: histogram_quantile(0.50, sum(rate(http_duration_bucket[5m])) by (le))
      - p95_latency:
          query: histogram_quantile(0.95, sum(rate(http_duration_bucket[5m])) by (le))
      - p99_latency:
          query: histogram_quantile(0.99, sum(rate(http_duration_bucket[5m])) by (le))

    row-3-errors:
      - error_rate:
          query: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
      - errors_by_type:
          query: sum(rate(http_requests_total{status=~"5.."}[5m])) by (status)

    row-4-resources:
      - cpu_usage:
          query: process_cpu_seconds_total
      - memory_usage:
          query: process_resident_memory_bytes
      - gc_collections:
          query: rate(dotnet_gc_collections_total[5m])

  business-metrics:
    row-1:
      - orders_per_minute:
          query: sum(rate(orderflow_orders_created[1m])) * 60
      - order_value:
          query: avg(orderflow_order_total)
      - pending_orders:
          query: orderflow_orders_pending
```

---

## Alerting

### Alert Design Principles

1. **Alert on symptoms, not causes**: "Error rate > 1%" not "Database CPU > 80%"
2. **Use error budgets**: Alert when burning through SLO too fast
3. **Page only for user impact**: Everything else goes to a ticket
4. **Include runbook links**: What should on-call do?

### Prometheus Alerting Rules

```yaml
# alerts/orderflow.yaml
groups:
  - name: orderflow-slo
    interval: 30s
    rules:
      # Error budget burn rate alert
      - alert: OrderFlowHighErrorRate
        expr: |
          (
            sum(rate(http_server_request_duration_seconds_count{job="orderflow-api",http_status_code=~"5.."}[5m]))
            /
            sum(rate(http_server_request_duration_seconds_count{job="orderflow-api"}[5m]))
          ) > 0.01
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "OrderFlow API error rate above 1%"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"
          runbook_url: "https://runbooks.example.com/orderflow/high-error-rate"

      # Latency SLO burn rate
      - alert: OrderFlowHighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_server_request_duration_seconds_bucket{job="orderflow-api"}[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "OrderFlow API p95 latency above 500ms"
          description: "P95 latency is {{ $value | humanizeDuration }}"
          runbook_url: "https://runbooks.example.com/orderflow/high-latency"

      # Queue depth growing
      - alert: OrderFlowQueueBacklog
        expr: orderflow_orders_pending > 1000
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "OrderFlow pending orders queue growing"
          description: "{{ $value }} orders pending for more than 10 minutes"
          runbook_url: "https://runbooks.example.com/orderflow/queue-backlog"

  - name: orderflow-saturation
    rules:
      # Database connection pool exhaustion
      - alert: OrderFlowDbPoolExhausted
        expr: |
          (npgsql_connections_busy / npgsql_connections_max) > 0.9
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Database connection pool > 90% utilized"
          runbook_url: "https://runbooks.example.com/orderflow/db-pool"

      # Memory pressure
      - alert: OrderFlowHighMemory
        expr: |
          (process_resident_memory_bytes / container_memory_limit_bytes) > 0.85
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "OrderFlow memory usage > 85%"
```

### Alert Routing (Alertmanager)

```yaml
# alertmanager.yaml
global:
  slack_api_url: 'https://hooks.slack.com/services/xxx'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'xxx'
        severity: critical

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warning'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

---

## Hands-On Exercise: Instrument OrderFlow

### Step 1: Add OpenTelemetry

```bash
# Add packages
dotnet add OrderFlow.Api package OpenTelemetry.Extensions.Hosting
dotnet add OrderFlow.Api package OpenTelemetry.Instrumentation.AspNetCore
dotnet add OrderFlow.Api package OpenTelemetry.Instrumentation.Http
dotnet add OrderFlow.Api package OpenTelemetry.Exporter.Console
```

### Step 2: Create Custom Instrumentation

```csharp
// Create OrderFlow.Telemetry/TelemetryConstants.cs
public static class TelemetryConstants
{
    public const string ServiceName = "OrderFlow";

    public static class ActivitySources
    {
        public const string Orders = "OrderFlow.Orders";
        public const string Inventory = "OrderFlow.Inventory";
        public const string Payments = "OrderFlow.Payments";
    }

    public static class Meters
    {
        public const string Orders = "OrderFlow.Orders";
        public const string Business = "OrderFlow.Business";
    }
}
```

### Step 3: Trace a Complete Request

Add instrumentation and capture a trace showing:
1. HTTP request received
2. Validation performed
3. Database queries
4. External service calls
5. Response sent

### Step 4: Create a Dashboard

Build a Grafana dashboard with:
- Request rate by endpoint
- Error rate over time
- P50/P95/P99 latency
- Orders created per minute
- Pending orders gauge

---

## Common Observability Anti-Patterns

### Anti-Pattern: Logging Everything

```csharp
// Anti-pattern: Verbose logging in hot path
public async Task<Order> GetOrderAsync(int id)
{
    _logger.LogDebug("Entering GetOrderAsync");
    _logger.LogDebug("Parameter id = {Id}", id);
    _logger.LogDebug("Checking cache...");

    // 10 more log lines...

    _logger.LogDebug("Exiting GetOrderAsync");
    return order;
}

// Better: Log meaningful events only
public async Task<Order> GetOrderAsync(int id)
{
    var order = await _cache.GetOrCreateAsync($"order:{id}",
        async () => await _repository.GetOrderAsync(id));

    if (order == null)
        _logger.LogWarning("Order {OrderId} not found", id);

    return order!;
}
```

### Anti-Pattern: High Cardinality Labels

```csharp
// Anti-pattern: User ID as metric label (millions of unique values!)
_ordersCreated.Add(1,
    new KeyValuePair<string, object?>("user_id", userId));  // DON'T

// Better: Use bounded labels
_ordersCreated.Add(1,
    new KeyValuePair<string, object?>("customer_tier", "premium"),
    new KeyValuePair<string, object?>("region", "us-west"));
```

---

## Deliverables

1. **OpenTelemetry Setup**: Working configuration with exporters
2. **Custom Instrumentation**: Activity sources and meters for key operations
3. **Dashboard**: Grafana dashboard covering the four golden signals
4. **Alert Rules**: Prometheus alerts for SLO violations
5. **Runbook**: Document how to investigate alerts

---

## Reflection Questions

1. What's the difference between monitoring and observability?
2. When would you use logs vs metrics vs traces?
3. How do you balance observability detail vs cost/overhead?
4. What makes a good alert?

---

## Resources

### OpenTelemetry
- [OpenTelemetry .NET Documentation](https://opentelemetry.io/docs/languages/net/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Jimmy Bogard - OpenTelemetry in .NET](https://www.youtube.com/watch?v=nFU-hcHyl2s)

### Observability Platforms
- [Grafana Documentation](https://grafana.com/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)

### Books and Guides
- [Observability Engineering](https://www.oreilly.com/library/view/observability-engineering/9781492076438/) by Charity Majors
- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Distributed Tracing in Practice](https://www.oreilly.com/library/view/distributed-tracing-in/9781492056621/)

---

**Next:** [Part 4 — Troubleshooting & Tuning Playbook](./part-4.md)
