# 06 — Messaging, Queues & Event-Driven Systems

## Part 4 / 4 — Operating Message Systems

---

## Why This Matters

Building a messaging system is one thing. Operating it is another:
- How do you know if messages are being processed?
- What happens when consumers fall behind?
- How do you debug a failed message from last Tuesday?
- What's your plan when the broker goes down?

Production readiness means observability, scaling, and incident response.

---

## Core Concept #15 — Distributed Tracing

### Why This Matters

In synchronous systems, a request flows through one path. In messaging systems:
- A single request can trigger dozens of messages
- Messages spawn more messages
- Processing happens across multiple services, minutes apart

Without tracing, debugging is impossible.

### What Goes Wrong Without This

**War story — The Phantom Failure:**
Customer reports order not delivered. Order exists in database. Payment was charged. Shipping service has no record. Team spends 4 hours searching logs across 5 services. Eventually discover message was dropped due to deserialization error. No trace connecting the events.

### OpenTelemetry Setup

```csharp
// Program.cs — Configure OpenTelemetry
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(serviceName: "OrderFlow.Api"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("MassTransit")  // MassTransit spans
        .AddSource("OrderFlow")     // Custom spans
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://jaeger:4317");
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddMeter("MassTransit")
        .AddMeter("OrderFlow")
        .AddOtlpExporter());

// Custom activity source for business operations
public static class Telemetry
{
    public static readonly ActivitySource Source = new("OrderFlow");
}
```

### Tracing Message Flow

```csharp
public class OrderService
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        using var activity = Telemetry.Source.StartActivity("CreateOrder");
        activity?.SetTag("order.customer_id", request.CustomerId);

        var order = new Order { /* ... */ };

        activity?.SetTag("order.id", order.Id);
        activity?.SetTag("order.total", order.TotalAmount);

        // Outbox message will inherit trace context
        await SaveOrderAndEventAsync(order);

        activity?.SetStatus(ActivityStatusCode.Ok);
        return order;
    }
}

// Consumer receives trace context automatically with MassTransit
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        // Span is automatically created as child of publisher's span
        using var activity = Telemetry.Source.StartActivity("ProcessOrderCreated");
        activity?.SetTag("order.id", context.Message.OrderId);

        try
        {
            await SendEmailAsync(context.Message);
            activity?.SetStatus(ActivityStatusCode.Ok);
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

### Trace Visualization

```
Trace: abc-123-def-456

├─ [HTTP] POST /api/orders                              15:00:00.000  250ms
│   ├─ [Span] CreateOrder                               15:00:00.010   50ms
│   │   ├─ order.id: guid-1
│   │   └─ order.customer_id: cust-1
│   └─ [DB] SaveChangesAsync                            15:00:00.060  180ms
│
├─ [Message] OrderCreatedEvent → payment-service        15:00:00.500  120ms
│   ├─ [Span] ProcessPayment                            15:00:00.510  100ms
│   │   └─ payment.status: success
│   └─ [HTTP] POST stripe.com/charges                   15:00:00.520   80ms
│
├─ [Message] PaymentProcessedEvent → shipping-service   15:00:01.000   50ms
│   └─ [Span] CreateShipment                            15:00:01.010   40ms
│       └─ shipment.tracking: TRK123
│
└─ [Message] ShipmentCreatedEvent → notification        15:00:01.500   30ms
    └─ [Span] SendEmail                                 15:00:01.510   20ms
        └─ email.to: customer@example.com
```

---

## Core Concept #16 — Metrics and Alerting

### Why This Matters

Metrics tell you:
- Is the system healthy right now?
- Is it degrading over time?
- When should we be paged?

### Essential Metrics

```csharp
// Custom metrics
public class MessagingMetrics
{
    private static readonly Meter Meter = new("OrderFlow.Messaging");

    // Counters
    public static readonly Counter<long> MessagesPublished =
        Meter.CreateCounter<long>("messaging.messages.published");

    public static readonly Counter<long> MessagesConsumed =
        Meter.CreateCounter<long>("messaging.messages.consumed");

    public static readonly Counter<long> MessagesFailed =
        Meter.CreateCounter<long>("messaging.messages.failed");

    // Histograms
    public static readonly Histogram<double> ProcessingDuration =
        Meter.CreateHistogram<double>("messaging.processing.duration", "ms");

    public static readonly Histogram<double> MessageAge =
        Meter.CreateHistogram<double>("messaging.message.age", "ms");

    // Gauges (via observable)
    public static readonly ObservableGauge<long> QueueDepth =
        Meter.CreateObservableGauge<long>("messaging.queue.depth",
            () => GetQueueDepths());
}

// In consumer
public class MetricsConsumerFilter<T> : IFilter<ConsumeContext<T>>
    where T : class
{
    public async Task Send(ConsumeContext<T> context, IPipe<ConsumeContext<T>> next)
    {
        var stopwatch = Stopwatch.StartNew();
        var messageAge = DateTime.UtcNow - context.SentTime ?? TimeSpan.Zero;

        try
        {
            await next.Send(context);

            MessagingMetrics.MessagesConsumed.Add(1,
                new KeyValuePair<string, object?>("message_type", typeof(T).Name),
                new KeyValuePair<string, object?>("status", "success"));
        }
        catch (Exception)
        {
            MessagingMetrics.MessagesFailed.Add(1,
                new KeyValuePair<string, object?>("message_type", typeof(T).Name));
            throw;
        }
        finally
        {
            MessagingMetrics.ProcessingDuration.Record(
                stopwatch.ElapsedMilliseconds,
                new KeyValuePair<string, object?>("message_type", typeof(T).Name));

            MessagingMetrics.MessageAge.Record(
                messageAge.TotalMilliseconds,
                new KeyValuePair<string, object?>("message_type", typeof(T).Name));
        }
    }
}
```

### Alerting Rules

```yaml
# Prometheus alerting rules
groups:
  - name: messaging
    rules:
      # DLQ is growing
      - alert: DeadLetterQueueGrowing
        expr: increase(messaging_dlq_depth[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "DLQ depth increasing"
          description: "{{ $labels.queue }} has {{ $value }} new DLQ messages"

      # Processing is too slow
      - alert: SlowMessageProcessing
        expr: histogram_quantile(0.99, messaging_processing_duration_bucket) > 30000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Message processing is slow"
          description: "P99 processing time is {{ $value }}ms"

      # Consumer is falling behind
      - alert: ConsumerLag
        expr: messaging_queue_depth > 10000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Consumer is falling behind"
          description: "Queue {{ $labels.queue }} has {{ $value }} pending messages"

      # High failure rate
      - alert: HighMessageFailureRate
        expr: |
          rate(messaging_messages_failed[5m])
          / rate(messaging_messages_consumed[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High message failure rate"
          description: "{{ $value | humanizePercentage }} of messages are failing"
```

### Dashboard Panels

```
┌────────────────────────────────────────────────────────────────────┐
│                    Message Processing Dashboard                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Messages/sec          Processing Time (P99)      Success Rate     │
│  ┌──────────────┐     ┌──────────────┐           ┌──────────────┐ │
│  │     1,234    │     │    45ms      │           │   99.8%      │ │
│  │    ▲ 12%     │     │    ▼ 5ms     │           │   ─ stable   │ │
│  └──────────────┘     └──────────────┘           └──────────────┘ │
│                                                                    │
│  Queue Depth                DLQ Messages           Consumer Lag    │
│  ┌──────────────┐     ┌──────────────┐           ┌──────────────┐ │
│  │     156      │     │      3       │           │   2.3s       │ │
│  │   ▼ normal   │     │   ▲ warning  │           │   ─ OK       │ │
│  └──────────────┘     └──────────────┘           └──────────────┘ │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Messages Processed Over Time                                │  │
│  │                     ___                                      │  │
│  │            ___     /   \___                                 │  │
│  │      ___  /   \___/        \___                             │  │
│  │  ___/   \/                      \___                        │  │
│  │  ───────────────────────────────────────────────────────    │  │
│  │  00:00    06:00    12:00    18:00    00:00                  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Core Concept #17 — Scaling Consumers

### Why This Matters

Message throughput varies:
- Black Friday: 100x normal traffic
- Batch jobs: Sudden spikes
- Normal operation: Steady flow

Consumers need to scale up and down automatically.

### Kubernetes Autoscaling with KEDA

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-processor
  template:
    metadata:
      labels:
        app: order-processor
    spec:
      containers:
        - name: order-processor
          image: orderflow/order-processor:latest
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          env:
            - name: SERVICE_BUS_CONNECTION
              valueFrom:
                secretKeyRef:
                  name: servicebus-secret
                  key: connection-string

---
# keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1
  maxReplicaCount: 20
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: order-processing
        namespace: orderflow-bus
        messageCount: "50"  # Target 50 messages per pod
      authenticationRef:
        name: azure-servicebus-auth

---
# Scale based on multiple triggers
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: multi-trigger-scaler
spec:
  scaleTargetRef:
    name: order-processor
  triggers:
    # Primary: Queue depth
    - type: azure-servicebus
      metadata:
        queueName: order-processing
        messageCount: "50"
    # Secondary: CPU (for processing-heavy work)
    - type: cpu
      metadata:
        type: Utilization
        value: "70"
```

### Consumer Concurrency Tuning

```csharp
// Program.cs — Tune concurrency based on workload
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderConsumer>();

    x.UsingAzureServiceBus((context, cfg) =>
    {
        cfg.Host(connectionString);

        cfg.ReceiveEndpoint("order-processing", e =>
        {
            // CPU-bound work: Lower concurrency
            e.PrefetchCount = 8;
            e.ConcurrentMessageLimit = 4;

            // I/O-bound work: Higher concurrency
            // e.PrefetchCount = 64;
            // e.ConcurrentMessageLimit = 32;

            // Memory-heavy work: Conservative
            // e.PrefetchCount = 4;
            // e.ConcurrentMessageLimit = 2;

            e.ConfigureConsumer<OrderConsumer>(context);
        });
    });
});

// Runtime tuning based on environment
var isHighMemoryPod = Environment.GetEnvironmentVariable("HIGH_MEMORY") == "true";
var concurrency = isHighMemoryPod ? 32 : 8;
```

---

## Core Concept #18 — Incident Response

### Why This Matters

Messaging systems fail in specific ways:
- Consumer crashes
- Broker unavailable
- DLQ explosion
- Poison message blocking queue

Having runbooks ready means faster recovery.

### Runbook: DLQ Spike

```markdown
# Runbook: Dead Letter Queue Spike

## Symptoms
- Alert: DeadLetterQueueGrowing
- DLQ depth > 100 messages
- Consumer logs showing repeated failures

## Immediate Actions (< 5 minutes)

1. **Assess Severity**
   ```bash
   # Check DLQ depth
   az servicebus queue show --name order-processing-dlq \
     --namespace orderflow-bus --query messageCount

   # Check active consumer pods
   kubectl get pods -l app=order-processor
   ```

2. **Check Consumer Logs**
   ```bash
   kubectl logs -l app=order-processor --tail=100 | grep -i error
   ```

3. **Sample DLQ Messages**
   ```bash
   # Use Service Bus Explorer or:
   dotnet run --project tools/DlqInspector -- peek 10
   ```

## Diagnosis

### Pattern: All Same Error
- **Cause**: Bug in consumer code or downstream service down
- **Action**: Fix bug or wait for downstream recovery, then replay

### Pattern: All Same Message Type
- **Cause**: Schema mismatch or missing handler
- **Action**: Deploy fix, then replay

### Pattern: Random Failures
- **Cause**: Transient errors, retry exhausted
- **Action**: Increase retry count, add circuit breaker

### Pattern: Single Poison Message
- **Cause**: Invalid data that can never process
- **Action**: Move to permanent dead letter, fix data source

## Recovery

1. **Fix Root Cause** (deploy fix if needed)

2. **Replay DLQ Messages**
   ```bash
   dotnet run --project tools/DlqReplayer -- \
     --queue order-processing \
     --batch-size 100 \
     --delay-ms 100
   ```

3. **Monitor Recovery**
   - Watch DLQ depth decrease
   - Check success rate returns to normal
   - Verify downstream systems received messages

## Post-Incident

- [ ] Document root cause
- [ ] Add test case for failure mode
- [ ] Update alerting thresholds if needed
- [ ] Schedule post-mortem if significant impact
```

### Runbook: Consumer Lag

```markdown
# Runbook: Consumer Falling Behind

## Symptoms
- Alert: ConsumerLag
- Queue depth growing
- Message age increasing

## Immediate Actions

1. **Check Consumer Health**
   ```bash
   kubectl get pods -l app=order-processor
   kubectl top pods -l app=order-processor
   ```

2. **Force Scale Up**
   ```bash
   kubectl scale deployment order-processor --replicas=10
   ```

3. **Check for Slow Processing**
   ```bash
   # Look for slow operations in traces
   # Query Jaeger/Zipkin for slow spans
   ```

## Diagnosis

### Pattern: Sudden Spike
- **Cause**: Traffic surge or batch job
- **Action**: Scale up, wait for backlog to clear

### Pattern: Gradual Increase
- **Cause**: Processing getting slower over time
- **Action**: Profile consumer, optimize slow paths

### Pattern: Single Pod Processing
- **Cause**: Session affinity or single consumer
- **Action**: Check session configuration, add consumers

## Recovery

1. **Scale Appropriately**
   - Match consumer count to throughput needed
   - Consider processing time per message

2. **Optimize if Needed**
   - Add indexes to database queries
   - Cache frequently accessed data
   - Batch downstream calls

3. **Monitor Catch-Up**
   - Queue depth should decrease steadily
   - Processing time should stay consistent
```

---

## Core Concept #19 — Chaos Engineering

### Why This Matters

The only way to know your system survives failures is to cause them intentionally.

### Testing Failure Modes

```csharp
// Integration test: Consumer crashes mid-processing
[Fact]
public async Task Message_Is_Reprocessed_After_Consumer_Crash()
{
    // Arrange
    var processCount = 0;
    var crashOnFirst = true;

    _consumer.OnProcess = async (message) =>
    {
        processCount++;
        if (crashOnFirst)
        {
            crashOnFirst = false;
            throw new Exception("Simulated crash");
        }
        await Task.CompletedTask;
    };

    // Act
    await _bus.PublishAsync(new TestMessage { Id = Guid.NewGuid() });

    // Wait for reprocessing
    await Task.Delay(TimeSpan.FromSeconds(5));

    // Assert
    Assert.Equal(2, processCount);  // Processed twice
}

// Integration test: Broker unavailable during publish
[Fact]
public async Task Outbox_Handles_Broker_Unavailability()
{
    // Arrange: Kill the broker
    await _brokerContainer.StopAsync();

    // Act: Create order (should save to outbox)
    var order = await _orderService.CreateOrderAsync(new CreateOrderRequest { /* ... */ });

    // Restart broker
    await _brokerContainer.StartAsync();

    // Wait for outbox processor to catch up
    await Task.Delay(TimeSpan.FromSeconds(10));

    // Assert: Message was eventually published
    var consumedMessage = await _testConsumer.WaitForMessageAsync<OrderCreatedEvent>(
        m => m.OrderId == order.Id,
        timeout: TimeSpan.FromSeconds(30));

    Assert.NotNull(consumedMessage);
}
```

### Toxiproxy for Network Chaos

```csharp
// docker-compose.yml
services:
  toxiproxy:
    image: ghcr.io/shopify/toxiproxy
    ports:
      - "8474:8474"

// Test with simulated network issues
[Fact]
public async Task Consumer_Handles_Network_Latency()
{
    // Add latency to broker connection
    await _toxiproxy.AddToxicAsync(new LatencyToxic
    {
        Name = "broker-latency",
        Proxy = "broker",
        Toxicity = 1.0,
        Attributes = new { latency = 1000, jitter = 500 }
    });

    // Run test
    var stopwatch = Stopwatch.StartNew();
    await _bus.PublishAsync(new TestMessage());
    var result = await _consumer.WaitForMessageAsync(TimeSpan.FromSeconds(30));
    stopwatch.Stop();

    // Assert message was processed despite latency
    Assert.NotNull(result);
    Assert.True(stopwatch.ElapsedMilliseconds > 1000);
}
```

---

## Hands-On: OrderFlow Production Readiness

### Task 1 — Add Distributed Tracing

Implement:
- OpenTelemetry configuration
- Custom spans for business operations
- Correlation ID propagation
- Jaeger visualization

### Task 2 — Create Dashboards

Build Grafana dashboards showing:
- Message throughput
- Processing latency (P50, P95, P99)
- Queue depth
- DLQ messages
- Consumer health

### Task 3 — Configure Alerting

Set up alerts for:
- DLQ growth
- High failure rate
- Consumer lag
- Slow processing

### Task 4 — Write Runbooks

Create runbooks for:
- DLQ spike
- Consumer lag
- Broker unavailability
- Poison message

### Task 5 — Chaos Testing

Implement tests for:
- Consumer crash and recovery
- Broker outage with outbox
- Network latency

---

## Deliverables

1. **Tracing setup** with example traces
2. **Dashboard** with key metrics
3. **Alerting rules** with appropriate thresholds
4. **Runbooks** for common incidents
5. **Chaos tests** proving resilience

---

## Resources

### Must-Read
- [OpenTelemetry .NET Documentation](https://opentelemetry.io/docs/instrumentation/net/)
- [KEDA Documentation](https://keda.sh/docs/)
- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)

### Videos
- [Jimmy Bogard: Distributed Tracing](https://www.youtube.com/watch?v=M8FCz8t0lsE)
- [KEDA: Kubernetes Event-Driven Autoscaling](https://www.youtube.com/watch?v=kKpUVxDzMlI)
- [Netflix: Chaos Engineering](https://www.youtube.com/watch?v=CZ3wIuvmHeM)

### Tools
- [Jaeger](https://www.jaegertracing.io/) — Distributed tracing
- [Grafana](https://grafana.com/) — Dashboards
- [KEDA](https://keda.sh/) — Kubernetes autoscaling
- [Toxiproxy](https://github.com/Shopify/toxiproxy) — Chaos testing

---

## Reflection Questions

1. What metrics would you alert on first for a new messaging system?
2. How do you balance thorough tracing with performance overhead?
3. What's the difference between scaling by queue depth vs processing time?
4. How often should you run chaos tests?

---

## Module Summary

You've learned the complete messaging stack:

1. **Foundations** — Commands, events, idempotency
2. **Brokers** — Queues, retries, DLQs
3. **Outbox** — Transactional publishing, deduplication
4. **Operations** — Tracing, metrics, scaling, incidents

**Key Takeaways:**
- Messaging trades simplicity for reliability and scale
- The outbox pattern is essential for consistency
- Every consumer must be idempotent
- You can't operate what you can't observe
- Test failure modes before production does

---

**Next Module:** [07 — Testing Strategies](../07-testing/README.md)
