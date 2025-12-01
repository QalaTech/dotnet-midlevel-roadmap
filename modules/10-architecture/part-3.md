# Part 3 — Service Boundaries & Microservices

## Why This Matters

"Let's split into microservices!" is one of the most expensive decisions a team can make. Done wrong, you get:

- Same problems, now distributed
- Network latency where function calls worked
- Partial failures everywhere
- Debugging nightmares
- Deployment complexity multiplied

Done right, you get:
- Independent team velocity
- Isolated scaling
- Technology flexibility
- Fault isolation

**The key question isn't "should we use microservices?" It's "are we ready for the trade-offs?"**

---

## What Goes Wrong Without This

### The Distributed Monolith

```
Before: Monolith
├── OrderService calls InventoryService (function call)
├── InventoryService calls PaymentService (function call)
└── Deploy: 1 deployment, 1 database, works fine

After: "Microservices"
├── OrderService calls InventoryService (HTTP + retry + timeout)
├── InventoryService calls PaymentService (HTTP + retry + timeout)
├── Services share the same database
├── Changes require deploying all three
└── Same coupling, but now with:
    - Network failures
    - Inconsistent deployments
    - 3x infrastructure
    - Distributed transactions

"We made our monolith worse by adding network boundaries"
```

### The Data Sharing Disaster

```
Team A: "Let's just query their database directly, it's faster"

Six months later:
- Team B changes their schema
- Team A's queries break in production
- No one knows who depends on what
- Emergency meeting at 2 AM

Root cause: Shared database = shared coupling
```

### The Saga That Never Ends

```
User: "I cancelled my order but was still charged!"

Order Service: Order status = Cancelled ✓
Payment Service: Payment = Processed ✓ (timeout during cancellation)
Inventory Service: Stock = Released ✓
Shipping Service: Label = Created ✓ (async, already queued)

Four services, four different states, angry customer.
```

---

## When to Split (The Real Checklist)

### Don't Split If:

- [ ] Team size < 8 (Conway's Law: structure follows team)
- [ ] You can't deploy services independently
- [ ] Services will share a database
- [ ] Most features span multiple services
- [ ] You don't have DevOps/SRE capacity
- [ ] "Because Netflix does it"

### Consider Splitting If:

- [ ] Different parts need different scaling
- [ ] Different teams need independent deployment
- [ ] Different parts need different tech stacks
- [ ] Fault isolation is critical
- [ ] You've already modularized the monolith successfully

### The Test: Can You Answer These?

```
Before splitting, you should be able to answer:

1. What data does each service own? (Clear ownership)
2. How do services communicate? (Sync vs async)
3. What happens when Service B is down? (Failure modes)
4. How do you trace a request across services? (Observability)
5. How do you maintain data consistency? (Transactions/Sagas)
6. Who gets paged when each service fails? (Team ownership)
```

---

## API Gateway Patterns

### Why an API Gateway?

```
Without Gateway:
┌─────────┐     ┌──────────────┐
│ Client  │ ──► │ Order Service│
└─────────┘     └──────────────┘
     │          ┌──────────────┐
     └────────► │ User Service │
     │          └──────────────┘
     └────────► │Payment Service│
                └──────────────┘

Problems:
- Client knows about all services
- Each service handles auth
- No centralized rate limiting
- Service discovery in client

With Gateway:
┌─────────┐     ┌─────────┐     ┌──────────────┐
│ Client  │ ──► │ Gateway │ ──► │ Order Service│
└─────────┘     └─────────┘     ├──────────────┤
                                │ User Service │
                                ├──────────────┤
                                │Payment Service│
                                └──────────────┘

Benefits:
- Single entry point
- Centralized auth/rate limiting
- Request routing
- Response aggregation
- Service discovery
```

### YARP (Yet Another Reverse Proxy) Setup

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapReverseProxy();

app.Run();
```

```json
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" }
        ]
      },
      "inventory-route": {
        "ClusterId": "inventory-cluster",
        "Match": {
          "Path": "/api/inventory/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" }
        ]
      }
    },
    "Clusters": {
      "orders-cluster": {
        "Destinations": {
          "orders-primary": {
            "Address": "http://orders-service:8080"
          }
        },
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:30",
            "Timeout": "00:00:10",
            "Path": "/health"
          }
        }
      },
      "inventory-cluster": {
        "Destinations": {
          "inventory-primary": {
            "Address": "http://inventory-service:8080"
          }
        }
      }
    }
  }
}
```

### Gateway with Rate Limiting and Caching

```csharp
// Program.cs - Enhanced gateway
var builder = WebApplication.CreateBuilder(args);

// Add rate limiting
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            context.User.Identity?.Name ?? context.Request.Headers["X-Client-Id"].ToString() ?? "anonymous",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
});

// Add output caching
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder
        .Expire(TimeSpan.FromMinutes(5))
        .Tag("api"));

    options.AddPolicy("products", builder => builder
        .Expire(TimeSpan.FromHours(1))
        .SetVaryByHeader("Accept-Language")
        .Tag("products"));
});

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.UseRateLimiter();
app.UseOutputCache();
app.MapReverseProxy();

app.Run();
```

---

## Saga Patterns

### The Problem: Distributed Transactions

```
Traditional transaction (monolith):
BEGIN TRANSACTION
  1. Create Order
  2. Reserve Inventory
  3. Charge Payment
  4. Create Shipment
COMMIT (or ROLLBACK everything)

Distributed "transaction" (microservices):
1. Order Service: Create Order ✓
2. Inventory Service: Reserve Stock (network timeout!)
   └── Did it succeed? Who knows?
3. Payment Service: Never reached
4. Shipment Service: Never reached

Result: Inconsistent state across services
```

### Saga: Orchestration vs Choreography

```
ORCHESTRATION (Central coordinator)
┌─────────────────────────────────────────────────────────────┐
│                    SAGA ORCHESTRATOR                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Create Order ─────────► Order Service                   │
│                   ◄───────── Order Created                  │
│                                                             │
│  2. Reserve Stock ────────► Inventory Service               │
│                   ◄───────── Stock Reserved                 │
│                                                             │
│  3. Process Payment ──────► Payment Service                 │
│                   ◄───────── Payment Processed              │
│                                                             │
│  4. Create Shipment ──────► Shipping Service                │
│                   ◄───────── Shipment Created               │
│                                                             │
│  On Failure: Send compensating commands                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘


CHOREOGRAPHY (Events drive workflow)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│ Order Service                                               │
│    │                                                        │
│    └── publishes: OrderPlaced                               │
│              │                                              │
│              ├───────────────┐                              │
│              ▼               ▼                              │
│    Inventory Service    Payment Service                     │
│         │                    │                              │
│         │ StockReserved      │ PaymentProcessed             │
│         │                    │                              │
│         └────────┬───────────┘                              │
│                  │                                          │
│                  ▼                                          │
│            Shipping Service                                 │
│                  │                                          │
│                  └── ShipmentCreated                        │
│                                                             │
│ On Failure: Each service publishes failure event            │
│             Subscribers handle compensation                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### When to Use Which?

| Aspect | Orchestration | Choreography |
|--------|---------------|--------------|
| **Complexity** | Centralized, easier to understand | Distributed, harder to trace |
| **Coupling** | Orchestrator knows all services | Services only know events |
| **Failure handling** | Central point handles all failures | Each service handles its failures |
| **Best for** | Complex workflows, strict ordering | Simple flows, loose coupling |
| **Risk** | Single point of failure | Event spaghetti |

### Implementing a Saga Orchestrator

```csharp
// Application/Sagas/OrderFulfillmentSaga.cs
public class OrderFulfillmentSaga :
    MassTransitStateMachine<OrderFulfillmentState>
{
    public State OrderCreated { get; private set; }
    public State StockReserved { get; private set; }
    public State PaymentProcessed { get; private set; }
    public State Completed { get; private set; }
    public State Failed { get; private set; }

    public Event<OrderPlaced> OrderPlacedEvent { get; private set; }
    public Event<StockReserved> StockReservedEvent { get; private set; }
    public Event<StockReservationFailed> StockFailedEvent { get; private set; }
    public Event<PaymentProcessed> PaymentProcessedEvent { get; private set; }
    public Event<PaymentFailed> PaymentFailedEvent { get; private set; }

    public OrderFulfillmentSaga()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderPlacedEvent, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => StockReservedEvent, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => StockFailedEvent, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentProcessedEvent, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentFailedEvent, x => x.CorrelateById(c => c.Message.OrderId));

        Initially(
            When(OrderPlacedEvent)
                .Then(context =>
                {
                    context.Saga.OrderId = context.Message.OrderId;
                    context.Saga.CustomerId = context.Message.CustomerId;
                    context.Saga.Total = context.Message.Total;
                    context.Saga.Items = context.Message.Items;
                })
                .Send(context => new ReserveStock
                {
                    OrderId = context.Message.OrderId,
                    Items = context.Message.Items
                })
                .TransitionTo(OrderCreated));

        During(OrderCreated,
            When(StockReservedEvent)
                .Send(context => new ProcessPayment
                {
                    OrderId = context.Saga.OrderId,
                    CustomerId = context.Saga.CustomerId,
                    Amount = context.Saga.Total
                })
                .TransitionTo(StockReserved),

            When(StockFailedEvent)
                .Send(context => new CancelOrder
                {
                    OrderId = context.Saga.OrderId,
                    Reason = "Insufficient stock"
                })
                .TransitionTo(Failed));

        During(StockReserved,
            When(PaymentProcessedEvent)
                .Send(context => new CreateShipment
                {
                    OrderId = context.Saga.OrderId,
                    Items = context.Saga.Items
                })
                .TransitionTo(Completed),

            When(PaymentFailedEvent)
                // Compensating action: release reserved stock
                .Send(context => new ReleaseStock
                {
                    OrderId = context.Saga.OrderId,
                    Items = context.Saga.Items
                })
                .Send(context => new CancelOrder
                {
                    OrderId = context.Saga.OrderId,
                    Reason = $"Payment failed: {context.Message.Reason}"
                })
                .TransitionTo(Failed));
    }
}

public class OrderFulfillmentState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; }
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal Total { get; set; }
    public List<OrderItem> Items { get; set; }
}
```

### Choreography with Domain Events

```csharp
// Order Service publishes
public record OrderPlacedIntegrationEvent(
    Guid OrderId,
    Guid CustomerId,
    List<OrderLineItem> Items,
    decimal Total);

// Inventory Service subscribes and publishes
public class OrderPlacedConsumer : IConsumer<OrderPlacedIntegrationEvent>
{
    private readonly IInventoryService _inventory;
    private readonly IPublishEndpoint _publisher;

    public async Task Consume(ConsumeContext<OrderPlacedIntegrationEvent> context)
    {
        var result = await _inventory.TryReserveStockAsync(
            context.Message.Items,
            context.Message.OrderId);

        if (result.Success)
        {
            await _publisher.Publish(new StockReservedIntegrationEvent
            {
                OrderId = context.Message.OrderId,
                ReservationId = result.ReservationId
            });
        }
        else
        {
            await _publisher.Publish(new StockReservationFailedIntegrationEvent
            {
                OrderId = context.Message.OrderId,
                Reason = result.FailureReason,
                UnavailableItems = result.UnavailableItems
            });
        }
    }
}

// Order Service subscribes to failure and compensates
public class StockReservationFailedConsumer : IConsumer<StockReservationFailedIntegrationEvent>
{
    private readonly IOrderService _orders;

    public async Task Consume(ConsumeContext<StockReservationFailedIntegrationEvent> context)
    {
        await _orders.CancelOrderAsync(
            context.Message.OrderId,
            $"Insufficient stock: {context.Message.Reason}");
    }
}
```

---

## Data Consistency Across Services

### The Problem: No Distributed Transactions

```
Traditional: ACID guarantees
- Atomicity: All or nothing
- Consistency: Data always valid
- Isolation: Transactions don't interfere
- Durability: Committed = persisted

Microservices: BASE
- Basically Available: System works most of the time
- Soft state: State may change over time (eventual consistency)
- Eventual consistency: Given time, all replicas converge
```

### Patterns for Consistency

**1. Outbox Pattern** - Reliably publish events

```csharp
// Instead of:
// 1. Save Order (might fail)
// 2. Publish Event (might fail)
// Result: Inconsistent state

// Use Outbox:
public class OrderDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<OutboxMessage> OutboxMessages { get; set; }

    public async Task SaveOrderWithEventAsync(Order order, OrderPlacedEvent @event)
    {
        using var transaction = await Database.BeginTransactionAsync();

        Orders.Add(order);

        // Store event in same transaction
        OutboxMessages.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = @event.GetType().FullName,
            Payload = JsonSerializer.Serialize(@event),
            CreatedAt = DateTime.UtcNow,
            ProcessedAt = null
        });

        await SaveChangesAsync();
        await transaction.CommitAsync();

        // Background process will publish and mark as processed
    }
}

// Background service publishes outbox messages
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var messages = await _db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(ct);

            foreach (var message in messages)
            {
                await _publisher.Publish(message.Type, message.Payload);
                message.ProcessedAt = DateTime.UtcNow;
            }

            await _db.SaveChangesAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
    }
}
```

**2. Inbox Pattern** - Idempotent message handling

```csharp
public class IdempotentConsumer<TMessage> : IConsumer<TMessage>
    where TMessage : class, IIntegrationEvent
{
    private readonly InboxDbContext _inbox;
    private readonly IConsumer<TMessage> _inner;

    public async Task Consume(ConsumeContext<TMessage> context)
    {
        var messageId = context.Message.MessageId;

        // Check if already processed
        if (await _inbox.ProcessedMessages.AnyAsync(m => m.Id == messageId))
        {
            return; // Already handled, skip
        }

        // Process
        await _inner.Consume(context);

        // Mark as processed
        _inbox.ProcessedMessages.Add(new ProcessedMessage
        {
            Id = messageId,
            ProcessedAt = DateTime.UtcNow
        });
        await _inbox.SaveChangesAsync();
    }
}
```

---

## Hands-On Exercise: Split a Module

### Step 1: Choose a Module to Extract

Review OrderFlow's modular monolith:
- Orders Module (core)
- Inventory Module (candidate for extraction)
- Payments Module (external dependency)

### Step 2: Define the Service Contract

```yaml
# inventory-service-contract.yaml
service: Inventory
version: 1.0

# Synchronous API
endpoints:
  - path: /api/inventory/{productId}/availability
    method: GET
    response:
      available: integer
      reserved: integer

  - path: /api/inventory/reserve
    method: POST
    request:
      orderId: uuid
      items: array of {productId, quantity}
    response:
      reservationId: uuid
      success: boolean

# Async Events (Published)
events:
  - name: StockReserved
    payload: {orderId, reservationId, items}

  - name: StockReservationFailed
    payload: {orderId, reason, unavailableItems}

  - name: StockReleased
    payload: {orderId, reservationId}

# Async Events (Subscribed)
subscriptions:
  - name: OrderCancelled
    action: Release any reserved stock
```

### Step 3: Implement the Saga

Create an order fulfillment saga that:
1. Creates order
2. Reserves inventory (with timeout)
3. Processes payment
4. Handles failures with compensation

### Step 4: Document the Split

```markdown
# ADR 005: Extract Inventory Service

## Status
Proposed

## Context
The Inventory module has:
- Different scaling needs (read-heavy)
- Different team ownership (warehouse team)
- Different deployment cadence (inventory updates daily)

## Decision
Extract Inventory into a separate service communicating via:
- REST API for synchronous availability checks
- Message queue for async reservations

## Consequences
### Positive
- Warehouse team can deploy independently
- Can scale inventory reads separately
- Clear ownership boundary

### Negative
- Network latency for availability checks
- Saga complexity for order fulfillment
- Additional infrastructure

## Migration Plan
1. Week 1-2: Add API gateway, keep monolith
2. Week 3-4: Extract Inventory service, run in parallel
3. Week 5: Route traffic to new service
4. Week 6: Remove Inventory module from monolith
```

---

## Deliverables

1. **Service Contract**: API and event definitions for one service
2. **Saga Implementation**: Orchestration or choreography for a workflow
3. **Gateway Configuration**: YARP setup routing to services
4. **Split Proposal**: ADR documenting a module extraction

---

## Reflection Questions

1. What problems does microservices solve that a modular monolith doesn't?
2. When is choreography better than orchestration?
3. How do you handle a saga that's been running for hours?
4. What's the cost of eventual consistency to users?

---

## Resources

### Microservices
- [Building Microservices](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/) by Sam Newman
- [Microservices Patterns](https://microservices.io/patterns/) by Chris Richardson
- [Martin Fowler - Microservices](https://martinfowler.com/articles/microservices.html)

### Sagas
- [Chris Richardson - Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Caitie McCaffrey - Sagas](https://www.youtube.com/watch?v=xDuwrtwYHu8)
- [MassTransit Saga Documentation](https://masstransit.io/documentation/patterns/saga)

### API Gateway
- [YARP Documentation](https://microsoft.github.io/reverse-proxy/)
- [Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/)

---

**Next:** [Part 4 — Resilience Patterns](./part-4.md)
