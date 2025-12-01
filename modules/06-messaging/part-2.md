# 06 — Messaging, Queues & Event-Driven Systems

## Part 2 / 4 — Queues and Message Brokers

---

## Why This Matters

Message brokers are the backbone of async systems. They provide:
- Durability (messages survive restarts)
- Delivery guarantees (at-least-once, exactly-once)
- Routing (topics, subscriptions, filters)
- Backpressure (queues buffer when consumers are slow)

But misconfigured brokers cause:
- Lost messages
- Infinite retry loops
- Dead letter queue explosions
- System-wide outages

---

## Core Concept #6 — Message Broker Fundamentals

### Why This Matters

Understanding broker primitives helps you design reliable systems:

| Concept | Azure Service Bus | RabbitMQ |
|---------|-------------------|----------|
| Point-to-point | Queue | Queue |
| Pub/Sub | Topic + Subscriptions | Exchange + Queues |
| Dead letters | Dead-letter queue | Dead-letter exchange |
| Delayed delivery | Scheduled messages | TTL + DLX |
| Ordering | Sessions | Single consumer |

### Queue vs Topic

```
QUEUE (Point-to-Point):
One message → One consumer

Producer ──► [Queue] ──► Consumer A
                    ──► Consumer B (competing)

Use for: Commands, work distribution

TOPIC (Pub/Sub):
One message → Multiple subscribers

                    ┌──► [Sub 1] ──► Payment Service
Producer ──► [Topic]├──► [Sub 2] ──► Shipping Service
                    └──► [Sub 3] ──► Analytics Service

Use for: Events, broadcasting
```

### Setting Up Azure Service Bus

```csharp
// Program.cs — Configure Service Bus
builder.Services.AddAzureClients(clientBuilder =>
{
    clientBuilder.AddServiceBusClient(
        builder.Configuration.GetConnectionString("ServiceBus"));
});

// Or with MassTransit (recommended abstraction)
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderPlacedConsumer>();
    x.AddConsumer<PaymentProcessedConsumer>();

    x.UsingAzureServiceBus((context, cfg) =>
    {
        cfg.Host(builder.Configuration.GetConnectionString("ServiceBus"));

        cfg.SubscriptionEndpoint<OrderPlacedEvent>(
            "order-notifications",  // Subscription name
            e =>
            {
                e.ConfigureConsumer<OrderPlacedConsumer>(context);
            });

        cfg.ConfigureEndpoints(context);
    });
});
```

### Setting Up RabbitMQ

```csharp
// docker-compose.yml
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: orderflow
      RABBITMQ_DEFAULT_PASS: orderflow

// Program.cs — Configure RabbitMQ with MassTransit
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderPlacedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("orderflow");
            h.Password("orderflow");
        });

        cfg.ConfigureEndpoints(context);
    });
});
```

---

## Core Concept #7 — Retry Policies

### Why This Matters

Transient failures happen:
- Network blips
- Database connection pool exhaustion
- Downstream service temporarily unavailable

Retry policies handle these automatically. But bad retry policies cause:
- Thundering herd (all retries at same time)
- Infinite loops (non-transient errors keep retrying)
- Resource exhaustion

### What Goes Wrong Without This

**War story — The Retry Storm:**
Payment service goes down for 5 minutes. 10,000 messages queue up. Service comes back. All 10,000 messages retry simultaneously. Service goes down again from load. Repeat for hours.

### Implementing Retry with Backoff

```csharp
// MassTransit retry configuration
cfg.SubscriptionEndpoint<OrderPlacedEvent>("payments", e =>
{
    // Immediate retries for transient failures
    e.UseMessageRetry(r =>
    {
        r.Intervals(
            TimeSpan.FromMilliseconds(100),
            TimeSpan.FromMilliseconds(500),
            TimeSpan.FromSeconds(1),
            TimeSpan.FromSeconds(5),
            TimeSpan.FromSeconds(30));

        // Only retry transient exceptions
        r.Handle<TimeoutException>();
        r.Handle<HttpRequestException>();
        r.Ignore<ValidationException>();  // Don't retry validation errors
    });

    // Delayed redelivery for persistent failures
    e.UseDelayedRedelivery(r =>
    {
        r.Intervals(
            TimeSpan.FromMinutes(1),
            TimeSpan.FromMinutes(5),
            TimeSpan.FromMinutes(30));
    });

    e.ConfigureConsumer<PaymentConsumer>(context);
});
```

### Exponential Backoff with Jitter

```csharp
public class RetryPolicy
{
    private static readonly Random Jitter = new();

    public static TimeSpan GetDelay(int attemptNumber, TimeSpan baseDelay)
    {
        // Exponential: 1s, 2s, 4s, 8s, 16s...
        var exponentialDelay = TimeSpan.FromTicks(
            baseDelay.Ticks * (long)Math.Pow(2, attemptNumber - 1));

        // Cap at 5 minutes
        var cappedDelay = TimeSpan.FromTicks(
            Math.Min(exponentialDelay.Ticks, TimeSpan.FromMinutes(5).Ticks));

        // Add jitter (±25%) to prevent thundering herd
        var jitterFactor = 0.75 + (Jitter.NextDouble() * 0.5);
        return TimeSpan.FromTicks((long)(cappedDelay.Ticks * jitterFactor));
    }
}

// Usage in consumer
public class OrderConsumer : IConsumer<OrderPlacedEvent>
{
    public async Task Consume(ConsumeContext<OrderPlacedEvent> context)
    {
        var attemptNumber = context.GetRetryAttempt();

        _logger.LogInformation(
            "Processing order {OrderId}, attempt {Attempt}",
            context.Message.OrderId,
            attemptNumber);

        try
        {
            await ProcessOrderAsync(context.Message);
        }
        catch (TransientException ex) when (attemptNumber < 5)
        {
            var delay = RetryPolicy.GetDelay(attemptNumber, TimeSpan.FromSeconds(1));
            _logger.LogWarning(ex,
                "Transient failure, scheduling retry in {Delay}",
                delay);

            throw;  // MassTransit handles retry
        }
    }
}
```

### Circuit Breaker Pattern

```csharp
// Polly circuit breaker for downstream calls
builder.Services.AddHttpClient<IPaymentGateway, PaymentGateway>()
    .AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (result, duration) =>
            {
                Log.Warning("Circuit breaker opened for {Duration}", duration);
            },
            onReset: () =>
            {
                Log.Information("Circuit breaker reset");
            });
}
```

---

## Core Concept #8 — Dead Letter Queues

### Why This Matters

Some messages can never be processed:
- Invalid data (missing required fields)
- Business rule violations
- Unrecoverable errors

These "poison messages" need somewhere to go. Without DLQ, they:
- Block the queue forever
- Retry infinitely
- Consume resources uselessly

### What Goes Wrong Without This

**War story — The Poison Message:**
Malformed JSON in one message. Consumer throws, message requeues. Consumer throws again. Repeat infinitely. Queue stops processing. 50,000 messages pile up behind one bad message.

### DLQ Configuration

```csharp
// Azure Service Bus — DLQ is automatic
// Messages go to DLQ after max delivery count

// MassTransit — Configure error handling
cfg.ReceiveEndpoint("order-processing", e =>
{
    // After all retries exhausted, move to error queue
    e.ConfigureConsumer<OrderConsumer>(context);

    // Messages that fail all retries go to: order-processing_error
    // Messages that can't be deserialized go to: order-processing_skipped
});

// Custom DLQ handling
public class OrderConsumer : IConsumer<OrderPlacedEvent>
{
    public async Task Consume(ConsumeContext<OrderPlacedEvent> context)
    {
        try
        {
            await ProcessAsync(context.Message);
        }
        catch (ValidationException ex)
        {
            // Don't retry validation errors — send to DLQ immediately
            _logger.LogError(ex,
                "Validation failed for order {OrderId}, moving to DLQ",
                context.Message.OrderId);

            // Move to fault queue with error details
            await context.Send(new DeadLetterMessage<OrderPlacedEvent>
            {
                OriginalMessage = context.Message,
                Error = ex.Message,
                FailedAt = DateTime.UtcNow,
                Consumer = nameof(OrderConsumer)
            });

            // Don't throw — message is handled
            return;
        }
    }
}
```

### DLQ Monitoring and Replay

```csharp
// Background service to monitor DLQ
public class DlqMonitorService : BackgroundService
{
    private readonly ServiceBusClient _client;
    private readonly ILogger<DlqMonitorService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var receiver = _client.CreateReceiver(
            "order-processing",
            new ServiceBusReceiverOptions
            {
                SubQueue = SubQueue.DeadLetter
            });

        while (!stoppingToken.IsCancellationRequested)
        {
            var messages = await receiver.PeekMessagesAsync(100, stoppingToken);

            if (messages.Count > 0)
            {
                _logger.LogWarning(
                    "DLQ has {Count} messages, oldest from {Time}",
                    messages.Count,
                    messages.First().EnqueuedTime);

                // Alert if DLQ is growing
                await _alertService.SendAlertAsync(new DlqAlert
                {
                    Queue = "order-processing",
                    MessageCount = messages.Count,
                    OldestMessage = messages.First().EnqueuedTime
                });
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}

// DLQ Replay Tool
public class DlqReplayService
{
    public async Task<ReplayResult> ReplayMessagesAsync(
        string queueName,
        int maxMessages,
        Func<ServiceBusReceivedMessage, bool>? filter = null)
    {
        var receiver = _client.CreateReceiver(
            queueName,
            new ServiceBusReceiverOptions { SubQueue = SubQueue.DeadLetter });

        var sender = _client.CreateSender(queueName);
        var replayed = 0;
        var failed = 0;

        while (replayed + failed < maxMessages)
        {
            var message = await receiver.ReceiveMessageAsync(TimeSpan.FromSeconds(5));
            if (message == null) break;

            if (filter != null && !filter(message))
            {
                await receiver.AbandonMessageAsync(message);
                continue;
            }

            try
            {
                // Send back to main queue
                await sender.SendMessageAsync(new ServiceBusMessage(message.Body)
                {
                    ContentType = message.ContentType,
                    CorrelationId = message.CorrelationId,
                    ApplicationProperties = { ["ReplayedFrom"] = "DLQ" }
                });

                // Complete (remove from DLQ)
                await receiver.CompleteMessageAsync(message);
                replayed++;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to replay message {Id}", message.MessageId);
                await receiver.AbandonMessageAsync(message);
                failed++;
            }
        }

        return new ReplayResult { Replayed = replayed, Failed = failed };
    }
}
```

---

## Core Concept #9 — Competing Consumers

### Why This Matters

Single consumer = single point of failure and throughput limit.
Multiple consumers = parallel processing and resilience.

But competing consumers introduce:
- Message ordering challenges
- Duplicate processing risks
- Resource contention

### Scaling Consumers

```csharp
// MassTransit — Configure concurrency
cfg.ReceiveEndpoint("order-processing", e =>
{
    // Process up to 16 messages concurrently
    e.PrefetchCount = 32;  // Fetch ahead for efficiency
    e.ConcurrentMessageLimit = 16;

    // Or use partitioning for ordered processing
    e.UseMessagePartitioner(p => p.Message.CustomerId);

    e.ConfigureConsumer<OrderConsumer>(context);
});

// Kubernetes scaling based on queue depth
// Using KEDA (Kubernetes Event-Driven Autoscaling)
```

```yaml
# keda-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: order-processing
        messageCount: "100"  # Scale up when queue > 100 messages
        connectionFromEnv: SERVICE_BUS_CONNECTION
```

### Message Ordering with Sessions

```csharp
// Azure Service Bus sessions for ordered delivery
// All messages with same SessionId go to same consumer

public async Task PublishOrderEventsAsync(Order order)
{
    var events = new[]
    {
        new OrderCreatedEvent { OrderId = order.Id, SessionId = order.Id.ToString() },
        new OrderValidatedEvent { OrderId = order.Id, SessionId = order.Id.ToString() },
        new OrderPaidEvent { OrderId = order.Id, SessionId = order.Id.ToString() }
    };

    foreach (var evt in events)
    {
        await _sender.SendMessageAsync(new ServiceBusMessage(
            JsonSerializer.Serialize(evt))
        {
            SessionId = evt.SessionId  // Guarantees FIFO within session
        });
    }
}

// Consumer with session support
cfg.ReceiveEndpoint("order-processing", e =>
{
    e.RequiresSession = true;
    e.MaxConcurrentSessions = 10;
    e.ConfigureConsumer<OrderConsumer>(context);
});
```

---

## Core Concept #10 — Security

### Why This Matters

Message brokers handle sensitive data:
- Payment information
- Personal data
- Business-critical commands

Unsecured brokers are a data breach waiting to happen.

### Azure Service Bus Security

```csharp
// Managed Identity (recommended for Azure)
builder.Services.AddAzureClients(clientBuilder =>
{
    clientBuilder.AddServiceBusClientWithNamespace(
        "orderflow.servicebus.windows.net")
        .WithCredential(new DefaultAzureCredential());
});

// Role assignments (via Bicep/Terraform)
// - Azure Service Bus Data Sender
// - Azure Service Bus Data Receiver

// SAS tokens (for external clients)
var connectionString = "Endpoint=...;SharedAccessKeyName=...;SharedAccessKey=...";
// Store in Key Vault, rotate regularly
```

### RabbitMQ Security

```csharp
// TLS configuration
cfg.Host("rabbitmq.internal", 5671, "/", h =>
{
    h.Username("orderflow");
    h.Password(configuration["RabbitMQ:Password"]);

    h.UseSsl(s =>
    {
        s.Protocol = SslProtocols.Tls12;
        s.CertificatePath = "/certs/client.pfx";
        s.CertificatePassphrase = configuration["RabbitMQ:CertPassword"];
    });
});

// Virtual hosts for isolation
cfg.Host("rabbitmq.internal", "/production", h => { /* ... */ });
cfg.Host("rabbitmq.internal", "/staging", h => { /* ... */ });
```

---

## Hands-On: OrderFlow Message Broker Setup

### Task 1 — Local Development Setup

Create Docker Compose with:
- RabbitMQ with management UI
- Service Bus Emulator (or use RabbitMQ)

### Task 2 — Configure Retry Policies

Implement retry with:
- Immediate retries (3x)
- Delayed redelivery (1min, 5min, 30min)
- Circuit breaker for downstream calls

### Task 3 — DLQ Monitoring

Build:
- Background service monitoring DLQ depth
- API endpoint to inspect DLQ messages
- Replay tool for recovered messages

### Task 4 — Load Testing

Test consumer scaling:
- Generate 10,000 messages
- Observe auto-scaling
- Measure throughput and latency

---

## Deliverables

1. **Docker Compose** for local broker setup
2. **Retry configuration** with exponential backoff and jitter
3. **DLQ tools** for monitoring and replay
4. **Load test results** with scaling observations

---

## Resources

### Must-Read
- [Azure Service Bus Documentation](https://learn.microsoft.com/en-us/azure/service-bus-messaging/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials)
- [MassTransit Documentation](https://masstransit.io/documentation/concepts)

### Videos
- [Nick Chapsas: MassTransit Deep Dive](https://www.youtube.com/watch?v=4FFjLTKtfDQ)
- [Hussein Nasser: Message Queues Explained](https://www.youtube.com/watch?v=oUJbuFMyBDk)
- [Raw Coding: Azure Service Bus](https://www.youtube.com/watch?v=Lz7NqEQwjnw)

### Tools
- [Azure Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer)
- [RabbitMQ Management UI](http://localhost:15672)
- [KEDA](https://keda.sh/) — Kubernetes autoscaling

---

## Reflection Questions

1. When would you use a queue vs a topic?
2. How do you prevent thundering herd on retry?
3. What's the trade-off between prefetch and memory usage?
4. How do you handle messages that should never be retried?

---

**Next:** [Part 3 — The Outbox Pattern](./part-3.md)
