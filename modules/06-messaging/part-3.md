# 06 — Messaging, Queues & Event-Driven Systems

## Part 3 / 4 — The Outbox Pattern

---

## Why This Matters

Publishing events to a message broker while updating your database creates a dangerous situation: the dual-write problem. If either operation fails, your system becomes inconsistent.

The outbox pattern solves this by treating event publishing as a database operation first, ensuring atomicity with your business data.

---

## Core Concept #11 — The Dual-Write Problem

### Why This Matters

Two writes to different systems cannot be atomic:

```csharp
// ❌ DANGEROUS: Dual write
public async Task CreateOrderAsync(Order order)
{
    // Write 1: Database
    await _dbContext.Orders.AddAsync(order);
    await _dbContext.SaveChangesAsync();

    // Write 2: Message broker
    await _messageBus.PublishAsync(new OrderCreatedEvent { OrderId = order.Id });
    // What if this fails? Order exists but event is lost!
}
```

### What Goes Wrong Without This

**Scenario 1 — Event Lost:**
```
1. Order saved to database ✓
2. Application crashes before publishing event
3. Order exists but no one knows about it
4. Customer never gets confirmation email
5. Inventory never reserved
6. Order sits in limbo forever
```

**Scenario 2 — Phantom Event:**
```
1. Order saved to database ✓
2. Event published ✓
3. Database transaction rolls back (constraint violation)
4. Order doesn't exist but event was sent
5. Payment service charges for non-existent order
6. Accounting nightmare
```

**War story — The Lost Orders:**
E-commerce site processes 10,000 orders/day. Database save succeeds, then network hiccup loses 0.1% of events. 10 orders/day never get processed. Nobody notices for weeks. Discovered when customers complain about missing shipments. Manual reconciliation takes 2 weeks.

---

## Core Concept #12 — The Outbox Pattern

### Why This Matters

The outbox pattern ensures events are never lost:
1. Save business data AND event in same transaction
2. Background process publishes events from outbox
3. Mark event as published

### How It Works

```
┌────────────────────────────────────────────────────────────────────┐
│                         Application                                 │
│                                                                    │
│  ┌─────────────────┐      ┌─────────────────┐                     │
│  │  Order Service  │      │ Outbox Processor│                     │
│  └────────┬────────┘      └────────┬────────┘                     │
│           │                        │                               │
│           │ BEGIN TRANSACTION      │                               │
│           │ INSERT Order           │                               │
│           │ INSERT OutboxMessage   │                               │
│           │ COMMIT                 │                               │
│           ▼                        │                               │
│  ┌─────────────────┐              │                               │
│  │    Database     │◄─────────────┤ Poll for unpublished          │
│  │                 │              │                               │
│  │  Orders         │              │                               │
│  │  OutboxMessages │──────────────┤ Read message                  │
│  └─────────────────┘              │                               │
│                                   │                               │
│                                   ▼                               │
│                          Publish to broker                        │
│                                   │                               │
│                                   ▼                               │
│                          Mark as published                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────┐
                         │  Message Broker │
                         └─────────────────┘
```

### Implementation

```csharp
// Outbox message entity
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public int Attempts { get; set; }
    public string? Error { get; set; }
    public string? CorrelationId { get; set; }
}

// EF Core configuration
public class OutboxMessageConfiguration : IEntityTypeConfiguration<OutboxMessage>
{
    public void Configure(EntityTypeBuilder<OutboxMessage> builder)
    {
        builder.ToTable("OutboxMessages");
        builder.HasKey(m => m.Id);

        builder.Property(m => m.Type).HasMaxLength(500).IsRequired();
        builder.Property(m => m.Payload).IsRequired();
        builder.Property(m => m.CorrelationId).HasMaxLength(100);
        builder.Property(m => m.Error).HasMaxLength(2000);

        // Index for efficient polling
        builder.HasIndex(m => new { m.ProcessedAt, m.CreatedAt })
            .HasFilter("[ProcessedAt] IS NULL")
            .HasDatabaseName("IX_OutboxMessages_Unprocessed");
    }
}
```

### Writing to Outbox

```csharp
public class OrderService
{
    private readonly OrderFlowDbContext _context;
    private readonly ICorrelationContext _correlation;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = request.CustomerId,
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice
            }).ToList(),
            TotalAmount = request.Items.Sum(i => i.Quantity * i.UnitPrice),
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };

        // Create event
        var orderCreatedEvent = new OrderCreatedEvent
        {
            EventId = Guid.NewGuid(),
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            TotalAmount = order.TotalAmount,
            Items = order.Items.Select(i => new OrderItemDto(
                i.ProductId, i.Quantity, i.UnitPrice)).ToList(),
            CreatedAt = order.CreatedAt,
            CorrelationId = _correlation.CorrelationId
        };

        // Create outbox message
        var outboxMessage = new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = typeof(OrderCreatedEvent).AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(orderCreatedEvent),
            CreatedAt = DateTime.UtcNow,
            CorrelationId = _correlation.CorrelationId
        };

        // ATOMIC: Both saved in same transaction
        _context.Orders.Add(order);
        _context.OutboxMessages.Add(outboxMessage);
        await _context.SaveChangesAsync();

        return order;
    }
}
```

### Outbox Processor

```csharp
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IMessageBus _messageBus;
    private readonly ILogger<OutboxProcessor> _logger;

    private readonly TimeSpan _pollingInterval = TimeSpan.FromSeconds(1);
    private readonly int _batchSize = 100;
    private readonly int _maxAttempts = 5;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessOutboxMessagesAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing outbox messages");
            }

            await Task.Delay(_pollingInterval, stoppingToken);
        }
    }

    private async Task ProcessOutboxMessagesAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();

        // Get unprocessed messages
        var messages = await context.OutboxMessages
            .Where(m => m.ProcessedAt == null)
            .Where(m => m.Attempts < _maxAttempts)
            .OrderBy(m => m.CreatedAt)
            .Take(_batchSize)
            .ToListAsync(ct);

        foreach (var message in messages)
        {
            try
            {
                // Deserialize and publish
                var eventType = Type.GetType(message.Type);
                if (eventType == null)
                {
                    _logger.LogError("Unknown message type: {Type}", message.Type);
                    message.Error = $"Unknown type: {message.Type}";
                    message.Attempts++;
                    continue;
                }

                var @event = JsonSerializer.Deserialize(message.Payload, eventType);
                await _messageBus.PublishAsync(@event!, ct);

                // Mark as processed
                message.ProcessedAt = DateTime.UtcNow;

                _logger.LogInformation(
                    "Published outbox message {MessageId} of type {Type}",
                    message.Id, message.Type);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Failed to publish outbox message {MessageId}, attempt {Attempt}",
                    message.Id, message.Attempts + 1);

                message.Attempts++;
                message.Error = ex.Message;
            }
        }

        await context.SaveChangesAsync(ct);
    }
}
```

---

## Core Concept #13 — Outbox Optimizations

### Why This Matters

The basic outbox pattern works but has performance concerns:
- Polling creates database load
- Large payloads bloat the table
- Old messages accumulate

### Polling vs Push

```csharp
// Option 1: Database polling (simple, universal)
// Already shown above — works with any database

// Option 2: Change Data Capture (SQL Server)
// Uses database transaction log — no polling needed
// More complex setup, better performance

// Option 3: PostgreSQL LISTEN/NOTIFY
public class PostgresOutboxListener : BackgroundService
{
    private readonly NpgsqlConnection _connection;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _connection.OpenAsync(stoppingToken);

        _connection.Notification += async (_, e) =>
        {
            if (e.Channel == "outbox_message_created")
            {
                var messageId = Guid.Parse(e.Payload);
                await ProcessMessageAsync(messageId);
            }
        };

        await using var cmd = new NpgsqlCommand("LISTEN outbox_message_created", _connection);
        await cmd.ExecuteNonQueryAsync(stoppingToken);

        // Keep connection alive
        while (!stoppingToken.IsCancellationRequested)
        {
            await _connection.WaitAsync(stoppingToken);
        }
    }
}

// Trigger to notify on insert
// CREATE FUNCTION notify_outbox_insert() RETURNS TRIGGER AS $$
// BEGIN
//   PERFORM pg_notify('outbox_message_created', NEW.id::text);
//   RETURN NEW;
// END;
// $$ LANGUAGE plpgsql;
//
// CREATE TRIGGER outbox_insert_trigger
// AFTER INSERT ON outbox_messages
// FOR EACH ROW EXECUTE FUNCTION notify_outbox_insert();
```

### Message Cleanup

```csharp
public class OutboxCleanupService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly TimeSpan _retentionPeriod = TimeSpan.FromDays(7);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var context = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();

            var cutoff = DateTime.UtcNow - _retentionPeriod;

            // Delete old processed messages
            var deleted = await context.OutboxMessages
                .Where(m => m.ProcessedAt != null)
                .Where(m => m.ProcessedAt < cutoff)
                .ExecuteDeleteAsync(stoppingToken);

            if (deleted > 0)
            {
                _logger.LogInformation("Cleaned up {Count} old outbox messages", deleted);
            }

            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

### Distributed Outbox Processing

```csharp
// Multiple processors with row-level locking
private async Task ProcessOutboxMessagesAsync(CancellationToken ct)
{
    using var scope = _scopeFactory.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();

    // Use transaction with row lock to prevent duplicate processing
    await using var transaction = await context.Database.BeginTransactionAsync(ct);

    var messages = await context.OutboxMessages
        .FromSqlRaw(@"
            SELECT TOP 100 *
            FROM OutboxMessages WITH (UPDLOCK, READPAST)
            WHERE ProcessedAt IS NULL
              AND Attempts < @maxAttempts
            ORDER BY CreatedAt",
            new SqlParameter("@maxAttempts", _maxAttempts))
        .ToListAsync(ct);

    foreach (var message in messages)
    {
        // Process message...
    }

    await context.SaveChangesAsync(ct);
    await transaction.CommitAsync(ct);
}
```

---

## Core Concept #14 — Consumer Deduplication (Inbox Pattern)

### Why This Matters

Even with outbox, consumers may receive duplicates:
- Network issues during acknowledgment
- Consumer restart during processing
- Message broker redelivery

Consumers must be idempotent.

### Inbox Pattern

```csharp
// Inbox message entity
public class InboxMessage
{
    public Guid MessageId { get; set; }  // PK, from EventId
    public string Type { get; set; } = string.Empty;
    public DateTime ProcessedAt { get; set; }
}

// Consumer with inbox deduplication
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly OrderFlowDbContext _context;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrderCreatedConsumer> _logger;

    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var message = context.Message;

        // Check if already processed
        var exists = await _context.InboxMessages
            .AnyAsync(m => m.MessageId == message.EventId);

        if (exists)
        {
            _logger.LogInformation(
                "Message {EventId} already processed, skipping",
                message.EventId);
            return;
        }

        // Process the message
        await using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            // Record in inbox FIRST (before processing)
            _context.InboxMessages.Add(new InboxMessage
            {
                MessageId = message.EventId,
                Type = typeof(OrderCreatedEvent).Name,
                ProcessedAt = DateTime.UtcNow
            });

            // Save inbox entry
            await _context.SaveChangesAsync();

            // Now process (if this fails, message is still marked as processed)
            await _emailService.SendOrderConfirmationAsync(
                message.CustomerEmail,
                message.OrderId,
                message.TotalAmount);

            await transaction.CommitAsync();
        }
        catch (DbUpdateException ex) when (IsDuplicateKeyException(ex))
        {
            // Race condition — another consumer processed first
            _logger.LogWarning(
                "Duplicate processing detected for {EventId}",
                message.EventId);
            await transaction.RollbackAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;  // Will be retried
        }
    }

    private static bool IsDuplicateKeyException(DbUpdateException ex)
    {
        return ex.InnerException is SqlException sqlEx &&
               sqlEx.Number == 2627;  // Unique constraint violation
    }
}
```

### Combined Outbox + Inbox

```csharp
// Full pattern: Outbox for publishing, Inbox for consuming
public class InventoryReservationHandler : IConsumer<OrderCreatedEvent>
{
    private readonly InventoryDbContext _context;

    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var message = context.Message;

        await using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            // 1. Check inbox (deduplication)
            if (await _context.InboxMessages.AnyAsync(m => m.MessageId == message.EventId))
            {
                _logger.LogInformation("Already processed {EventId}", message.EventId);
                return;
            }

            // 2. Record in inbox
            _context.InboxMessages.Add(new InboxMessage
            {
                MessageId = message.EventId,
                Type = nameof(OrderCreatedEvent),
                ProcessedAt = DateTime.UtcNow
            });

            // 3. Business logic
            foreach (var item in message.Items)
            {
                var product = await _context.Products.FindAsync(item.ProductId);
                product.ReservedQuantity += item.Quantity;
            }

            // 4. Create outbox message for result
            var reservedEvent = new InventoryReservedEvent
            {
                EventId = Guid.NewGuid(),
                OrderId = message.OrderId,
                CorrelationId = message.CorrelationId,
                CausationId = message.EventId.ToString(),
                ReservedAt = DateTime.UtcNow
            };

            _context.OutboxMessages.Add(new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = typeof(InventoryReservedEvent).AssemblyQualifiedName!,
                Payload = JsonSerializer.Serialize(reservedEvent),
                CreatedAt = DateTime.UtcNow,
                CorrelationId = message.CorrelationId
            });

            // 5. Commit everything atomically
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch (DbUpdateException ex) when (IsDuplicateKeyException(ex))
        {
            await transaction.RollbackAsync();
            // Already processed — success
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;  // Retry
        }
    }
}
```

---

## Hands-On: OrderFlow Outbox Implementation

### Task 1 — Create Outbox Infrastructure

Add to OrderFlow:
- `OutboxMessage` entity and configuration
- `InboxMessage` entity and configuration
- Database migrations

### Task 2 — Implement Transactional Publishing

Modify `OrderService.CreateOrderAsync` to:
- Save order and outbox message in same transaction
- Include correlation ID

### Task 3 — Build Outbox Processor

Create background service that:
- Polls for unprocessed messages
- Publishes to message broker
- Handles failures with retry
- Cleans up old messages

### Task 4 — Implement Consumer with Inbox

Create `OrderNotificationHandler` that:
- Checks inbox for duplicates
- Sends email notification
- Records in inbox
- Handles race conditions

### Task 5 — Integration Tests

Test:
- Event is published after order creation
- Duplicate messages are handled
- Failed publishing is retried
- Transaction rollback doesn't lose events

---

## Deliverables

1. **Outbox infrastructure** with EF Core configuration
2. **Transactional publishing** in domain services
3. **Outbox processor** background service
4. **Consumer deduplication** with inbox pattern
5. **Integration tests** proving reliability

---

## Resources

### Must-Read
- [Jimmy Bogard: The Outbox Pattern](https://www.jimmybogard.com/life-beyond-distributed-transactions-an-apostates-implementation-sagas/)
- [Microservices.io: Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [MassTransit: Outbox](https://masstransit.io/documentation/patterns/transactional-outbox)

### Videos
- [Derek Comartin: Outbox Pattern Explained](https://www.youtube.com/watch?v=u8fOnxAxKHk)
- [Milan Jovanovic: Outbox Pattern in .NET](https://www.youtube.com/watch?v=XnMrJt2gCPE)
- [Chris Patterson: MassTransit Transactional Outbox](https://www.youtube.com/watch?v=3TjGnmLno_A)

### Libraries
- [MassTransit Outbox](https://masstransit.io/documentation/patterns/transactional-outbox)
- [NServiceBus Outbox](https://docs.particular.net/nservicebus/outbox/)
- [Wolverine](https://wolverine.netlify.app/) — Built-in outbox support

---

## Reflection Questions

1. Why can't you just wrap database + broker in a distributed transaction?
2. What happens if the outbox processor crashes after publishing but before marking as processed?
3. How do you handle messages that should be published in a specific order?
4. What's the trade-off between immediate publishing and polling?

---

**Next:** [Part 4 — Operating Message Systems](./part-4.md)
