# 06 — Messaging, Queues & Event-Driven Systems

## Part 1 / 4 — Messaging Foundations

---

## Why This Matters

HTTP is synchronous: client waits for response. For many operations, this is exactly wrong:
- User shouldn't wait 30 seconds while you process their order
- Your service shouldn't fail if the email service is down
- Multiple systems need to react to the same event
- Operations need to retry automatically on failure

Messaging decouples systems in time and space. But it requires a different mental model.

---

## Core Concept #1 — When to Use Messaging

### Why This Matters

Messaging isn't always better than HTTP. Each has its place:

| Use HTTP When | Use Messaging When |
|---------------|-------------------|
| Client needs response immediately | Client doesn't need to wait |
| Operation is fast (<500ms) | Operation is slow or unreliable |
| Failure should return to user | Failure should retry automatically |
| Simple request/response | Multiple systems need to react |
| Strong consistency required | Eventual consistency acceptable |

### What Goes Wrong Without This

**War story — The Synchronous Email:**
Order API calls email service synchronously. Email service goes down. Orders fail. Customers can't buy. Email is not critical path — order should succeed regardless.

**War story — The Fire-and-Forget:**
Team uses messaging for everything. User registration publishes "UserCreated" event. UI shows "Registration complete!" Event handler fails. User never gets confirmation email. No retry. User thinks registration worked but account isn't set up.

### Decision Framework

```csharp
// ❌ WRONG: Synchronous email in order flow
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
{
    var order = await _orderService.CreateAsync(request);

    // If email fails, entire order fails!
    await _emailService.SendOrderConfirmationAsync(order);

    return Created($"/orders/{order.Id}", order);
}

// ✅ RIGHT: Async email via messaging
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
{
    var order = await _orderService.CreateAsync(request);

    // Publish event — email will be sent asynchronously
    await _messageBus.PublishAsync(new OrderCreatedEvent
    {
        OrderId = order.Id,
        CustomerId = order.CustomerId,
        CustomerEmail = order.CustomerEmail,
        Items = order.Items,
        TotalAmount = order.TotalAmount,
        CreatedAt = order.CreatedAt
    });

    return Created($"/orders/{order.Id}", order);
}
```

---

## Core Concept #2 — Commands vs Events

### Why This Matters

Commands and events look similar (both are messages) but have fundamentally different semantics:

| Aspect | Command | Event |
|--------|---------|-------|
| Meaning | "Do this" | "This happened" |
| Tense | Imperative | Past tense |
| Sender knows receivers | Yes | No |
| Receiver count | Exactly one | Zero or more |
| Can be rejected | Yes | No (already happened) |
| Examples | PlaceOrder, SendEmail | OrderPlaced, EmailSent |

### What Goes Wrong Without This

**Anti-pattern — Command disguised as event:**
```csharp
// ❌ WRONG: "SendEmail" is a command, not an event
public record EmailShouldBeSentEvent(string To, string Subject, string Body);

// What if two services subscribe? User gets duplicate emails.
// What if no service subscribes? Email never sent, no error.
```

**Anti-pattern — Event disguised as command:**
```csharp
// ❌ WRONG: Event contains instructions
public record OrderCreatedEvent(
    Guid OrderId,
    bool ShouldSendEmail,      // Don't tell consumers what to do
    bool ShouldNotifyWarehouse,
    bool ShouldUpdateAnalytics);
```

### Correct Patterns

```csharp
// Commands — imperative, single receiver
public record PlaceOrderCommand(
    Guid CustomerId,
    List<OrderItem> Items,
    ShippingAddress Address,
    PaymentInfo Payment)
{
    public Guid CommandId { get; init; } = Guid.NewGuid();
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
}

public record SendEmailCommand(
    string To,
    string Subject,
    string Body,
    Guid? CorrelationId = null);

// Events — past tense, multiple receivers
public record OrderPlacedEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    List<OrderItemDto> Items,
    DateTime PlacedAt)
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public int Version { get; init; } = 1;
}

// Notification (special type — informational, no action expected)
public record LowInventoryNotification(
    Guid ProductId,
    int CurrentStock,
    int ReorderThreshold);
```

### Event Flow Example

```
Order API                  Payment Service           Shipping Service
    │                            │                          │
    │  OrderPlacedEvent          │                          │
    ├───────────────────────────►│                          │
    │                            │                          │
    │  OrderPlacedEvent          │                          │
    ├─────────────────────────────────────────────────────►│
    │                            │                          │
    │                      PaymentProcessedEvent            │
    │◄───────────────────────────┤                          │
    │                            │                          │
    │                            │    PaymentProcessedEvent │
    │                            ├─────────────────────────►│
    │                            │                          │
    │                      ShipmentCreatedEvent             │
    │◄─────────────────────────────────────────────────────┤
```

---

## Core Concept #3 — Message Contracts

### Why This Matters

Messages are contracts between services. They need:
- Clear schemas (what fields exist)
- Versioning (how to evolve without breaking consumers)
- Metadata (correlation IDs, timestamps, source)

### Contract Design

```csharp
// Base class for all events
public abstract record IntegrationEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
    public string CorrelationId { get; init; } = string.Empty;
    public string CausationId { get; init; } = string.Empty;
    public int Version { get; init; } = 1;
    public string Source { get; init; } = string.Empty;
}

// Concrete event
public record OrderPlacedEvent : IntegrationEvent
{
    public required Guid OrderId { get; init; }
    public required Guid CustomerId { get; init; }
    public required string CustomerEmail { get; init; }
    public required decimal TotalAmount { get; init; }
    public required string Currency { get; init; }
    public required List<OrderItemDto> Items { get; init; }
    public required AddressDto ShippingAddress { get; init; }
}

public record OrderItemDto(
    Guid ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice);

public record AddressDto(
    string Line1,
    string? Line2,
    string City,
    string PostalCode,
    string Country);
```

### Versioning Strategies

```csharp
// Strategy 1: Additive changes (backward compatible)
// V1
public record OrderPlacedEvent_V1(Guid OrderId, Guid CustomerId, decimal Total);

// V2 — Add optional field (V1 consumers still work)
public record OrderPlacedEvent_V2(
    Guid OrderId,
    Guid CustomerId,
    decimal Total,
    string? CustomerEmail);  // New field, nullable for backward compatibility

// Strategy 2: Version in message type
[MessageVersion(2)]
public record OrderPlacedEvent { /* V2 schema */ }

// Consumer handles multiple versions
public class OrderPlacedHandler :
    IHandleMessages<OrderPlacedEvent>,
    IHandleMessages<OrderPlacedEvent_V1>  // Legacy support
{
    public Task Handle(OrderPlacedEvent message, IMessageContext context) { /* ... */ }
    public Task Handle(OrderPlacedEvent_V1 message, IMessageContext context)
    {
        // Convert to current version
        var current = new OrderPlacedEvent { /* map fields */ };
        return Handle(current, context);
    }
}

// Strategy 3: Envelope with version
public record MessageEnvelope<T>(
    T Payload,
    string Type,
    int Version,
    string CorrelationId,
    DateTime Timestamp);
```

---

## Core Concept #4 — Idempotent Handlers

### Why This Matters

Messages can be delivered more than once:
- Network hiccup during acknowledgment
- Consumer crashes after processing but before ack
- Retry after transient failure

Your handlers MUST handle duplicates safely.

### What Goes Wrong Without This

**War story — Double Charge:**
Payment handler processes message, charges customer, then crashes before acknowledging. Message broker redelivers. Handler charges customer again. Customer is charged twice.

### Idempotency Patterns

```csharp
// Pattern 1: Idempotency key table
public class IdempotentMessageHandler<TMessage>
    where TMessage : IntegrationEvent
{
    private readonly IProcessedMessageRepository _processedMessages;

    public async Task HandleAsync(TMessage message, Func<Task> handler)
    {
        // Check if already processed
        if (await _processedMessages.ExistsAsync(message.EventId))
        {
            _logger.LogInformation(
                "Message {EventId} already processed, skipping",
                message.EventId);
            return;
        }

        // Process
        await handler();

        // Mark as processed
        await _processedMessages.MarkAsProcessedAsync(
            message.EventId,
            typeof(TMessage).Name,
            DateTime.UtcNow);
    }
}

// Processed messages table
public class ProcessedMessage
{
    public Guid MessageId { get; set; }  // PK
    public string MessageType { get; set; }
    public DateTime ProcessedAt { get; set; }
    public DateTime ExpiresAt { get; set; }  // Cleanup old records
}
```

```csharp
// Pattern 2: Business state check
public class OrderPaymentHandler
{
    public async Task HandleAsync(ProcessPaymentCommand command)
    {
        var order = await _orderRepository.GetByIdAsync(command.OrderId);

        // Check if already paid (idempotent by design)
        if (order.PaymentStatus == PaymentStatus.Paid)
        {
            _logger.LogInformation(
                "Order {OrderId} already paid, skipping",
                command.OrderId);
            return;
        }

        // Process payment
        var result = await _paymentGateway.ChargeAsync(
            order.CustomerId,
            order.TotalAmount);

        // Update order
        order.PaymentStatus = result.Success
            ? PaymentStatus.Paid
            : PaymentStatus.Failed;

        await _orderRepository.UpdateAsync(order);
    }
}
```

```csharp
// Pattern 3: Optimistic concurrency
public class InventoryHandler
{
    public async Task HandleAsync(ReserveInventoryCommand command)
    {
        var product = await _productRepository.GetByIdAsync(command.ProductId);

        // Check if this specific reservation already exists
        if (product.Reservations.Any(r => r.ReservationId == command.ReservationId))
        {
            return;  // Already reserved
        }

        // Reserve with version check
        product.Reservations.Add(new Reservation
        {
            ReservationId = command.ReservationId,
            Quantity = command.Quantity,
            ExpiresAt = DateTime.UtcNow.AddMinutes(15)
        });

        try
        {
            await _productRepository.UpdateAsync(product, expectedVersion: product.Version);
        }
        catch (ConcurrencyException)
        {
            // Retry with fresh data
            throw new RetryableException("Concurrent modification, retrying");
        }
    }
}
```

---

## Core Concept #5 — Correlation and Causation

### Why This Matters

In distributed systems, you need to trace message flow:
- Which HTTP request started this chain?
- Which message caused this message?
- How are all these messages related?

### Implementing Correlation

```csharp
public interface IMessageContext
{
    string CorrelationId { get; }
    string CausationId { get; }
    string MessageId { get; }
}

// Set correlation ID from HTTP request
public class CorrelationMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers["X-Correlation-Id"] = correlationId;

        await next(context);
    }
}

// Publish event with correlation
public class OrderService
{
    private readonly ICorrelationContext _correlation;
    private readonly IMessageBus _bus;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order { /* ... */ };
        await _repository.CreateAsync(order);

        await _bus.PublishAsync(new OrderPlacedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            // ... other fields

            CorrelationId = _correlation.CorrelationId,  // From HTTP request
            CausationId = _correlation.CorrelationId,    // This is the root cause
            Source = "OrderService"
        });

        return order;
    }
}

// Handler publishes derived event with causation chain
public class PaymentHandler
{
    public async Task HandleAsync(OrderPlacedEvent orderPlaced)
    {
        var payment = await _paymentGateway.ChargeAsync(/* ... */);

        await _bus.PublishAsync(new PaymentProcessedEvent
        {
            PaymentId = payment.Id,
            OrderId = orderPlaced.OrderId,

            CorrelationId = orderPlaced.CorrelationId,  // Same root
            CausationId = orderPlaced.EventId,          // Caused by OrderPlaced
            Source = "PaymentService"
        });
    }
}
```

### Tracing Message Flow

```
Original HTTP Request (Correlation: abc-123)
    │
    └─► OrderPlacedEvent (ID: evt-1, Correlation: abc-123, Causation: abc-123)
            │
            ├─► PaymentProcessedEvent (ID: evt-2, Correlation: abc-123, Causation: evt-1)
            │       │
            │       └─► EmailSentEvent (ID: evt-4, Correlation: abc-123, Causation: evt-2)
            │
            └─► ShipmentCreatedEvent (ID: evt-3, Correlation: abc-123, Causation: evt-1)
```

---

## Hands-On: OrderFlow Messaging Design

### Task 1 — Event Storming

Map OrderFlow's order lifecycle:
1. What events occur?
2. What commands trigger them?
3. What services react to each event?

### Task 2 — Define Message Contracts

Create C# records for:
- `OrderPlacedEvent`
- `PaymentProcessedEvent`
- `ShipmentCreatedEvent`
- `OrderCompletedEvent`

Include metadata (correlation ID, version, timestamp).

### Task 3 — Implement Idempotent Handler

Build a notification handler that:
- Receives `OrderPlacedEvent`
- Sends confirmation email
- Handles duplicate deliveries safely

### Task 4 — Correlation Tracing

Implement correlation ID propagation from:
- HTTP request
- Through event publishing
- To downstream handlers

---

## Deliverables

1. **Event flow diagram** for OrderFlow's order lifecycle
2. **Message contracts** with versioning strategy documented
3. **Idempotent handler** with processed message tracking
4. **Correlation ID flow** across services

---

## Resources

### Must-Read
- [Martin Fowler: What do you mean by Event-Driven?](https://martinfowler.com/articles/201701-event-driven.html)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [MassTransit Documentation](https://masstransit.io/documentation/concepts)

### Videos
- [Udi Dahan: Avoid a Failed SOA](https://www.youtube.com/watch?v=p2r8v2P6Q8Y)
- [Derek Comartin: Event-Driven Architecture Mistakes](https://www.youtube.com/watch?v=GBTdnfD6s5Q)
- [Nick Chapsas: Message Queues in .NET](https://www.youtube.com/watch?v=F7GTmxBmlaU)

### Books
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Chapter 11
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [Building Event-Driven Microservices](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/)

---

## Reflection Questions

1. When would synchronous HTTP be better than messaging?
2. How do you handle message ordering requirements?
3. What happens if an event handler throws?
4. How do you test event-driven systems?

---

**Next:** [Part 2 — Queues and Message Brokers](./part-2.md)
