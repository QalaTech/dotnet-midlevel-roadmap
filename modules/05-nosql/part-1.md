# 05 — NoSQL & Distributed Data

## Part 1 / 3 — NoSQL Foundations & CAP Trade-offs

---

## Why This Matters

Choosing the wrong database is one of the most expensive mistakes you can make:
- It's hard to change once you have data
- Wrong choice leads to performance problems that can't be fixed
- Teams spend months migrating when they should be building features
- Some problems become impossible without architectural changes

Understanding when NoSQL fits — and when it doesn't — is a senior engineering skill.

---

## Core Concept #1 — SQL vs NoSQL: When to Use What

### Why This Matters

This isn't a religious debate. It's engineering trade-offs:

| Criteria | SQL (PostgreSQL, SQL Server) | NoSQL (MongoDB, DynamoDB) |
|----------|------------------------------|---------------------------|
| Schema flexibility | Low — migrations required | High — schema-on-read |
| Relationships | Excellent — JOINs | Poor — denormalize or multiple queries |
| ACID transactions | Built-in | Limited or none |
| Horizontal scaling | Difficult | Built for it |
| Query flexibility | Very high (ad-hoc queries) | Limited (plan queries upfront) |
| Operational complexity | Lower | Higher |

### What Goes Wrong Without This

**War story #1 — The MongoDB for Everything Disaster:**
Startup uses MongoDB for their entire application. Works great at first — fast development, no migrations. Then they need reporting: "Show me all orders from customers who spent more than $1000 last month." Requires JOINs across three collections. Query takes 30 seconds. They build a SQL replica for reporting. Now they have two databases, sync problems, and data inconsistency.

Should have used SQL from the start for their transactional, relational data.

**War story #2 — The PostgreSQL Scale Wall:**
E-commerce site stores product catalog in PostgreSQL. Works fine until they hit 10 million products with 50 variations each. Every product query requires JOINs to load variations, images, attributes. Database CPU pegged at 100%. Add indexes, tune queries, still slow.

Eventually migrate product catalog to MongoDB where products are self-contained documents. Query goes from 50ms to 5ms.

### Decision Framework

```
Choose SQL when:
├── Data is highly relational (orders → customers → products)
├── ACID transactions are critical (financial data)
├── Ad-hoc querying is needed (business intelligence)
├── Data integrity is paramount (constraints, foreign keys)
├── Schema is stable and well-understood
└── Scale is predictable (millions, not billions of rows)

Choose NoSQL when:
├── Data is self-contained (documents, events, logs)
├── Schema varies widely (different product types)
├── Read patterns are predictable (key lookups)
├── Horizontal scaling is required (global distribution)
├── High write throughput is needed (IoT, logging)
└── Eventual consistency is acceptable
```

### Real Examples

```
OrderFlow Data Store Decisions:

✅ SQL (PostgreSQL/SQL Server):
├── Orders — Relational (order → items → products → customers)
├── Customers — Need foreign keys to orders
├── Inventory — ACID required (can't oversell)
└── Payments — Financial data, audit requirements

✅ MongoDB:
├── Product Catalog — Flexible attributes per product type
├── Audit Logs — Append-only, flexible schema
├── Search Indexes — Denormalized for fast search
└── User Activity — High write volume, time-series data

✅ Redis:
├── Session Data — Fast, ephemeral
├── Shopping Cart — Fast reads, TTL for abandoned carts
├── Rate Limiting — Atomic counters
└── Distributed Locks — Checkout inventory holds
```

---

## Core Concept #2 — CAP Theorem (What It Actually Means)

### Why This Matters

CAP theorem is misunderstood and over-simplified. Understanding it properly helps you make informed decisions about data consistency.

### The Theorem

In a distributed system, when a network partition occurs, you must choose between:
- **Consistency** — Every read returns the most recent write
- **Availability** — Every request receives a response

You cannot have both during a partition. But here's the key insight: **partitions are rare, and you make this choice per-operation, not globally**.

### What CAP Actually Means in Practice

```
Normal Operation (no partition):
└── You can have both Consistency AND Availability

During Partition:
├── Choose Consistency (CP): Reject writes until partition heals
│   └── "Sorry, we can't process your order right now"
│   └── Example: Bank transfers, inventory reservations
│
└── Choose Availability (AP): Accept writes, reconcile later
    └── "Order accepted, we'll confirm availability"
    └── Example: Shopping cart, user preferences, activity feed
```

### Real-World Database Positions

| Database | Default Behavior | Trade-off |
|----------|------------------|-----------|
| PostgreSQL | CP | Single node, strong consistency |
| MongoDB | CP (single doc) | Tunable consistency per operation |
| DynamoDB | AP | Eventually consistent by default |
| Cosmos DB | Tunable | 5 consistency levels |
| Redis | AP | Master-replica, eventual consistency |
| Cassandra | AP | Tunable consistency |

### Cosmos DB Consistency Levels (Example of Tunable Consistency)

```
Strong ──────────────────────────────────> Eventual
   │                                           │
   ├── Strong: Always read latest write        │
   │   └── High latency, low throughput        │
   │                                           │
   ├── Bounded Staleness: Read lags by K       │
   │   └── Predictable lag                     │
   │                                           │
   ├── Session: Read-your-writes guaranteed    │
   │   └── Best for most web apps              │
   │                                           │
   ├── Consistent Prefix: Reads in order       │
   │   └── Never see out-of-order updates      │
   │                                           │
   └── Eventual: Fastest, cheapest             │
       └── May read stale data                 │
```

---

## Core Concept #3 — Eventual Consistency Patterns

### Why This Matters

If you use NoSQL (or any distributed system), you'll deal with eventual consistency. Handling it wrong leads to:
- Users seeing stale data
- Lost updates
- Inconsistent state across services

### What Goes Wrong Without This

**Scenario:** User adds item to cart. Cache shows item. User refreshes page. Replica hasn't caught up. Item appears gone. User adds again. Now has duplicate item.

### Pattern 1: Read-Your-Writes

```csharp
public class CartService
{
    private readonly IDistributedCache _cache;
    private readonly ICartRepository _repository;

    public async Task<Cart> GetCartAsync(string userId)
    {
        // Try cache first
        var cached = await _cache.GetStringAsync($"cart:{userId}");
        if (cached != null)
        {
            return JsonSerializer.Deserialize<Cart>(cached);
        }

        // Fall back to database
        var cart = await _repository.GetByUserIdAsync(userId);

        // Cache for next read
        await _cache.SetStringAsync(
            $"cart:{userId}",
            JsonSerializer.Serialize(cart),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
            });

        return cart;
    }

    public async Task AddItemAsync(string userId, CartItem item)
    {
        // Write to primary
        await _repository.AddItemAsync(userId, item);

        // Immediately update cache (read-your-writes)
        var cart = await _repository.GetByUserIdAsync(userId);
        await _cache.SetStringAsync(
            $"cart:{userId}",
            JsonSerializer.Serialize(cart));

        // Now user will see their own write immediately
    }
}
```

### Pattern 2: Version Vectors / ETags

```csharp
public class ProductService
{
    public async Task<ProductUpdateResult> UpdateProductAsync(
        Guid productId,
        UpdateProductRequest request,
        string expectedVersion)
    {
        var product = await _collection.Find(p => p.Id == productId)
            .FirstOrDefaultAsync();

        if (product == null)
            return ProductUpdateResult.NotFound();

        // Check version matches
        if (product.Version != expectedVersion)
        {
            return ProductUpdateResult.Conflict(
                currentVersion: product.Version,
                message: "Product was modified by another request");
        }

        // Update with new version
        var newVersion = Guid.NewGuid().ToString();
        var update = Builders<Product>.Update
            .Set(p => p.Name, request.Name)
            .Set(p => p.Price, request.Price)
            .Set(p => p.Version, newVersion);

        var result = await _collection.UpdateOneAsync(
            p => p.Id == productId && p.Version == expectedVersion,
            update);

        if (result.ModifiedCount == 0)
        {
            // Concurrent modification
            return ProductUpdateResult.Conflict(
                currentVersion: product.Version,
                message: "Product was modified during update");
        }

        return ProductUpdateResult.Success(newVersion);
    }
}
```

### Pattern 3: Idempotent Operations

```csharp
public class PaymentService
{
    public async Task<PaymentResult> ProcessPaymentAsync(
        ProcessPaymentRequest request)
    {
        // Check if this idempotency key was already processed
        var existing = await _paymentRepository.GetByIdempotencyKeyAsync(
            request.IdempotencyKey);

        if (existing != null)
        {
            // Return same result as before (idempotent)
            return new PaymentResult
            {
                PaymentId = existing.Id,
                Status = existing.Status,
                WasAlreadyProcessed = true
            };
        }

        // Process new payment
        var payment = new Payment
        {
            Id = Guid.NewGuid(),
            IdempotencyKey = request.IdempotencyKey,
            Amount = request.Amount,
            Status = PaymentStatus.Processing
        };

        await _paymentRepository.CreateAsync(payment);

        // External payment processor call
        var externalResult = await _paymentGateway.ChargeAsync(request);

        payment.Status = externalResult.Success
            ? PaymentStatus.Completed
            : PaymentStatus.Failed;

        await _paymentRepository.UpdateAsync(payment);

        return new PaymentResult
        {
            PaymentId = payment.Id,
            Status = payment.Status,
            WasAlreadyProcessed = false
        };
    }
}
```

### Pattern 4: Compensating Transactions (Saga)

```csharp
// When you need consistency across multiple services/databases
public class OrderSaga
{
    public async Task<OrderResult> PlaceOrderAsync(PlaceOrderRequest request)
    {
        var sagaId = Guid.NewGuid();
        var compensations = new Stack<Func<Task>>();

        try
        {
            // Step 1: Reserve inventory
            var inventoryHold = await _inventoryService.ReserveAsync(
                request.Items, sagaId);
            compensations.Push(() => _inventoryService.ReleaseAsync(sagaId));

            // Step 2: Charge payment
            var payment = await _paymentService.ChargeAsync(
                request.PaymentInfo, sagaId);
            compensations.Push(() => _paymentService.RefundAsync(sagaId));

            // Step 3: Create order
            var order = await _orderService.CreateAsync(request, sagaId);
            // No compensation needed — order is the goal

            // Success!
            return OrderResult.Success(order);
        }
        catch (Exception ex)
        {
            // Compensate in reverse order
            while (compensations.Count > 0)
            {
                var compensate = compensations.Pop();
                try
                {
                    await compensate();
                }
                catch (Exception compensationEx)
                {
                    // Log but continue compensating
                    _logger.LogError(compensationEx,
                        "Compensation failed for saga {SagaId}", sagaId);
                }
            }

            return OrderResult.Failure(ex.Message);
        }
    }
}
```

---

## Core Concept #4 — Polyglot Persistence

### Why This Matters

Modern applications often need multiple databases. The key is choosing the right database for each use case, not picking one for everything.

### OrderFlow's Polyglot Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        OrderFlow API                             │
└─────────────────────────────────────────────────────────────────┘
          │                    │                    │
          ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   PostgreSQL    │  │    MongoDB      │  │     Redis       │
│                 │  │                 │  │                 │
│  Orders         │  │  Products       │  │  Sessions       │
│  Customers      │  │  Audit Logs     │  │  Carts          │
│  Inventory      │  │  Search Index   │  │  Rate Limits    │
│  Payments       │  │  Activity Feed  │  │  Locks          │
└─────────────────┘  └─────────────────┘  └─────────────────┘

Why each choice:

PostgreSQL (Transactional, Relational):
├── Orders need ACID transactions
├── Customers have relationships to orders
├── Inventory requires strong consistency (can't oversell)
└── Payments require audit trails and referential integrity

MongoDB (Flexible, Document-Oriented):
├── Products have varying attributes (electronics vs clothing)
├── Audit logs are append-only with flexible schema
├── Search index is denormalized for fast queries
└── Activity feed is time-series with nested data

Redis (Fast, Ephemeral):
├── Sessions need sub-millisecond access
├── Carts expire after 24 hours (TTL)
├── Rate limiting needs atomic increment
└── Checkout locks need distributed coordination
```

### Keeping Data Synchronized

```csharp
// When order is created, update multiple stores
public class OrderCreatedHandler : INotificationHandler<OrderCreatedEvent>
{
    private readonly IProductSearchIndex _searchIndex;
    private readonly IActivityFeed _activityFeed;
    private readonly IAnalyticsStore _analytics;

    public async Task Handle(OrderCreatedEvent notification, CancellationToken ct)
    {
        // Update MongoDB search index (async, eventual consistency OK)
        await _searchIndex.UpdateProductSalesCountAsync(notification.ProductIds);

        // Append to MongoDB activity feed (async, eventual consistency OK)
        await _activityFeed.AppendAsync(new ActivityEntry
        {
            Type = "OrderCreated",
            CustomerId = notification.CustomerId,
            Data = notification
        });

        // Update analytics (async, eventual consistency OK)
        await _analytics.RecordOrderAsync(notification);

        // Note: These updates are eventually consistent
        // The source of truth (PostgreSQL order) is strongly consistent
    }
}
```

---

## Hands-On: OrderFlow Data Architecture

### Task 1 — Create Architecture Decision Record

Document the data store choices for OrderFlow:
- Which data goes where and why
- Consistency requirements per data type
- Trade-offs accepted

### Task 2 — Docker Compose Setup

Create a local development environment with all databases:

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: orderflow
      POSTGRES_PASSWORD: orderflow
      POSTGRES_DB: orderflow
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  mongo_data:
  redis_data:
```

### Task 3 — Eventual Consistency Drill

Implement a scenario where eventual consistency is visible:
1. Write to primary database
2. Add artificial delay before replica sync
3. Read from replica (shows stale data)
4. Implement read-your-writes pattern to fix

### Task 4 — Failure Simulation

1. Stop MongoDB replica and observe behavior
2. Kill Redis and see cache fallback
3. Document recovery procedures

---

## Deliverables

1. **Architecture Decision Record** for OrderFlow's data stores
2. **Docker Compose** setup for local development
3. **Consistency patterns** implemented in code
4. **Failure runbook** documenting how to handle database outages

---

## Resources

### Must-Read
- [MongoDB: SQL to MongoDB Mapping](https://www.mongodb.com/docs/manual/reference/sql-comparison/)
- [AWS: When to Use NoSQL](https://aws.amazon.com/nosql/)
- [Martin Kleppmann: Designing Data-Intensive Applications](https://dataintensive.net/) — Chapters 5-9

### Videos
- [Martin Kleppmann: Transactions in Distributed Systems](https://www.youtube.com/watch?v=5ZjhNTM8XU8)
- [Hussein Nasser: CAP Theorem Simplified](https://www.youtube.com/watch?v=BHqjEjzAicg)
- [Nick Chapsas: MongoDB in .NET](https://www.youtube.com/watch?v=5ER4_0b5n-A)

### Documentation
- [Azure Cosmos DB Consistency Levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [MongoDB Replication](https://www.mongodb.com/docs/manual/replication/)

---

## Reflection Questions

1. When is eventual consistency acceptable for user-facing data?
2. How would you handle a scenario where Redis fails mid-checkout?
3. What data would you never put in NoSQL?
4. How do you ensure data consistency across multiple databases?

---

**Next:** [Part 2 — Document Databases (MongoDB)](./part-2.md)
