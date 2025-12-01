# Part 2 — Domain-Driven Design (DDD)

## Why This Matters

DDD isn't about patterns like Repository or Value Object. It's about **speaking the same language as the business**.

When developers use different words than the business:
- Meetings become translation sessions
- Requirements get lost in translation
- Code doesn't reflect how the business actually works
- Changes take longer because developers must interpret intent

When everyone uses the same language:
- Requirements are clearer
- Code reads like business documentation
- Domain experts can validate the model
- New features map naturally to the code

---

## What Goes Wrong Without This

### The Translation Tax

```
Product Owner: "We need to allow customers to place orders on hold"
Developer: "So... set a flag in the database?"
Product Owner: "Well, when an order is on hold, billing should pause"
Developer: "I'll add an if-check in the billing service"
Product Owner: "And shipping should be blocked"
Developer: "Another if-check..."
Product Owner: "And the customer service team should see why it's on hold"
Developer: "Where does that information come from?"
Product Owner: "The reason the customer gave when they put it on hold"
Developer: "So there's a reason field now?"

Three weeks of back-and-forth that should have been one conversation.
```

### The Entity Soup

```csharp
// Everything is an "entity" with a bunch of properties
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public string Status { get; set; }  // "pending", "paid", "shipped"...
    public DateTime? PaidAt { get; set; }
    public DateTime? ShippedAt { get; set; }
    public DateTime? CancelledAt { get; set; }
    public string CancelReason { get; set; }
    public decimal Subtotal { get; set; }
    public decimal Tax { get; set; }
    public decimal ShippingCost { get; set; }
    public decimal Total { get; set; }  // Is this calculated or stored?
    public decimal Discount { get; set; }
    public string DiscountCode { get; set; }
    // ... 47 more properties

    // No behavior, just data
    // Rules scattered across 12 service classes
}
```

### The Aggregate Anemia

```csharp
// "We're doing DDD!" but really it's just smart data containers
public class OrderService
{
    public void CancelOrder(int orderId, string reason)
    {
        var order = _repository.GetById(orderId);

        // All "domain logic" in the service
        if (order.Status == "shipped")
            throw new Exception("Can't cancel shipped order");

        if (order.Status == "cancelled")
            throw new Exception("Already cancelled");

        order.Status = "cancelled";
        order.CancelledAt = DateTime.UtcNow;
        order.CancelReason = reason;

        // Manually emit "events"
        _eventBus.Publish(new OrderCancelledEvent { OrderId = orderId });

        _repository.Save(order);
    }
}

// The Order "aggregate" knows nothing about its own rules
```

---

## DDD Building Blocks

### The Strategic vs Tactical Split

```
┌─────────────────────────────────────────────────────────────┐
│                    STRATEGIC DDD                            │
│           (What problems do we solve?)                      │
├─────────────────────────────────────────────────────────────┤
│  • Bounded Contexts: Where do responsibilities lie?         │
│  • Context Maps: How do contexts relate?                    │
│  • Ubiquitous Language: What words mean in each context     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    TACTICAL DDD                             │
│           (How do we model the solution?)                   │
├─────────────────────────────────────────────────────────────┤
│  • Aggregates: Consistency boundaries                       │
│  • Entities: Things with identity                           │
│  • Value Objects: Things without identity                   │
│  • Domain Events: Facts that happened                       │
│  • Domain Services: Verbs that don't belong to one entity   │
└─────────────────────────────────────────────────────────────┘
```

---

## Event Storming

### What Is Event Storming?

A collaborative workshop where developers and domain experts discover the domain together using sticky notes.

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT STORMING BOARD                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Actor]         [Command]        [Event]         [Policy]  │
│  (Yellow)        (Blue)           (Orange)        (Purple)  │
│                                                             │
│  ┌─────────┐     ┌───────────┐    ┌────────────┐           │
│  │Customer │ --> │Place Order│ -->│Order Placed│           │
│  └─────────┘     └───────────┘    └─────┬──────┘           │
│                                         │                   │
│                                         ▼                   │
│                               ┌─────────────────┐           │
│                               │ When order      │           │
│                               │ placed, reserve │           │
│                               │ inventory       │           │
│                               └────────┬────────┘           │
│                                        │                    │
│  ┌─────────┐                           ▼                    │
│  │Inventory│     ┌────────────┐  ┌─────────────────┐        │
│  │ System  │ <-- │ Reserve    │  │Stock Reserved   │        │
│  └─────────┘     │ Stock      │  └─────────────────┘        │
│                  └────────────┘                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Running an Event Storming Session

**Materials needed**:
- Large wall or whiteboard
- Sticky notes (orange, blue, yellow, purple, pink, green)
- Markers
- Domain experts and developers

**Process**:

1. **Chaotic Exploration (20 min)**
   - Everyone writes domain events (orange) on stickies
   - "Order Placed", "Payment Received", "Stock Reserved"
   - No discussion yet, just capture events

2. **Timeline Organization (15 min)**
   - Place events on the wall left-to-right chronologically
   - Group related events together
   - Mark confused areas with pink "hotspot" stickies

3. **Add Commands and Actors (20 min)**
   - Blue stickies for commands that cause events
   - Yellow stickies for who/what triggers commands

4. **Add Policies (15 min)**
   - Purple stickies for business rules
   - "When X happens, then Y should happen"

5. **Identify Aggregates (20 min)**
   - Draw boundaries around related events
   - Name the aggregates
   - These become your bounded contexts or modules

### OrderFlow Event Storm Example

```
┌──────────────────────────────────────────────────────────────────┐
│                         ORDER CONTEXT                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Customer    Place Order     Order Placed                         │
│ [yellow]    [blue]          [orange]                             │
│    │            │               │                                │
│    └────────────┴───────────────┴────────────────┐               │
│                                                  │               │
│                                                  ▼               │
│                                    ┌─────────────────────────┐   │
│                                    │ Policy: Reserve Stock   │   │
│                                    │ Policy: Create Invoice  │   │
│                                    └─────────────────────────┘   │
│                                                                  │
│ Customer    Cancel Order    Order Cancelled                      │
│ [yellow]    [blue]          [orange]                             │
│    │            │               │                                │
│    └────────────┴───────────────┴────────────────┐               │
│                                                  │               │
│                                                  ▼               │
│                                    ┌─────────────────────────┐   │
│                                    │ Policy: Release Stock   │   │
│                                    │ Policy: Void Invoice    │   │
│                                    │ Policy: Refund Payment  │   │
│                                    └─────────────────────────┘   │
│                                                                  │
│  AGGREGATE: Order                                                │
│  • Enforces: Cannot cancel shipped orders                        │
│  • Enforces: Order total = sum of lines                         │
│  • Enforces: Minimum order value $10                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Bounded Contexts

### What Is a Bounded Context?

A boundary within which a model has a specific meaning.

**Same word, different meanings:**

| Word | Sales Context | Shipping Context | Finance Context |
|------|---------------|------------------|-----------------|
| Order | Products customer wants | Packages to ship | Revenue to record |
| Customer | Someone to sell to | Delivery address | Account to bill |
| Product | SKU with price | Item with weight | Line item |
| Address | Billing preference | Delivery location | Legal entity location |

### Context Mapping

```
┌─────────────────────────────────────────────────────────────────┐
│                       ORDERFLOW CONTEXT MAP                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────────┐          ┌─────────────┐                     │
│    │   Orders    │          │  Inventory  │                     │
│    │   Context   │ ◄─────── │  Context    │                     │
│    │             │ Customer │             │                     │
│    │ (Core)      │ Supplier │ (Core)      │                     │
│    └──────┬──────┘          └─────────────┘                     │
│           │                                                     │
│           │ Published                                           │
│           │ Language                                            │
│           ▼                                                     │
│    ┌─────────────┐          ┌─────────────┐                     │
│    │  Payments   │          │  Shipping   │                     │
│    │  Context    │          │  Context    │                     │
│    │             │          │             │                     │
│    │ (External)  │          │ (External)  │                     │
│    └─────────────┘          └─────────────┘                     │
│                                                                 │
│  Relationship Types:                                            │
│  ◄── Customer/Supplier: Orders tells Inventory what it needs   │
│  → Published Language: Orders publishes events others consume  │
│  External: Third-party systems we integrate with               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Context Relationship Patterns

| Pattern | Description | When to Use |
|---------|-------------|-------------|
| **Partnership** | Two contexts evolve together | Same team owns both |
| **Customer-Supplier** | Supplier provides what customer needs | Different teams, one depends on other |
| **Conformist** | Downstream adopts upstream's model | When you can't influence upstream |
| **Anti-Corruption Layer** | Translate external model to yours | Integrating with legacy/external systems |
| **Shared Kernel** | Small shared model both contexts use | Carefully! Creates coupling |
| **Published Language** | Well-defined API/events for consumers | Multiple downstream consumers |

---

## Implementing Aggregates

### What Is an Aggregate?

A cluster of objects treated as a single unit for data changes.

```
┌─────────────────────────────────────────────────────────────┐
│                        ORDER AGGREGATE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    ┌─────────────────────────────────┐                      │
│    │         Order (Root)            │                      │
│    │  • Id                           │                      │
│    │  • CustomerId                   │                      │
│    │  • Status                       │                      │
│    │  • PlacedAt                     │                      │
│    └─────────────┬───────────────────┘                      │
│                  │                                          │
│          ┌───────┴───────┐                                  │
│          │               │                                  │
│    ┌─────▼─────┐   ┌─────▼─────────────┐                    │
│    │OrderLine  │   │ ShippingAddress   │                    │
│    │• ProductId│   │ (Value Object)    │                    │
│    │• Quantity │   │ • Street          │                    │
│    │• Price    │   │ • City            │                    │
│    └───────────┘   │ • PostalCode      │                    │
│                    └───────────────────┘                    │
│                                                             │
│  RULES:                                                     │
│  • All access goes through the Order (root)                 │
│  • OrderLines cannot exist without an Order                 │
│  • Order enforces all invariants                            │
│  • Single transaction boundary                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Implementing the Order Aggregate

```csharp
// Domain/Aggregates/Order.cs
namespace OrderFlow.Domain.Aggregates;

public class Order : AggregateRoot
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public ShippingAddress ShippingAddress { get; private set; }
    public DateTime PlacedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public DateTime? CancelledAt { get; private set; }
    public string? CancellationReason { get; private set; }

    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order() { } // For EF

    public static Order Create(
        CustomerId customerId,
        ShippingAddress shippingAddress,
        IEnumerable<OrderLineData> lines)
    {
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            ShippingAddress = shippingAddress,
            Status = OrderStatus.Pending,
            PlacedAt = DateTime.UtcNow
        };

        foreach (var lineData in lines)
        {
            order.AddLine(lineData.ProductId, lineData.Quantity, lineData.UnitPrice);
        }

        // Invariant: Orders must have at least one line
        if (!order._lines.Any())
            throw new DomainException("Order must have at least one line");

        // Invariant: Minimum order value
        if (order.Total < Money.FromDecimal(10m, "USD"))
            throw new DomainException("Minimum order value is $10");

        order.AddDomainEvent(new OrderPlacedEvent(
            order.Id,
            order.CustomerId,
            order.Total,
            order.Lines.Select(l => new OrderLineSnapshot(l.ProductId, l.Quantity)).ToList()));

        return order;
    }

    public void AddLine(ProductId productId, int quantity, Money unitPrice)
    {
        // Invariant: Cannot modify non-pending orders
        if (Status != OrderStatus.Pending)
            throw new DomainException("Cannot modify non-pending order");

        // Invariant: Quantity must be positive
        if (quantity <= 0)
            throw new DomainException("Quantity must be positive");

        var existingLine = _lines.FirstOrDefault(l => l.ProductId == productId);
        if (existingLine != null)
        {
            existingLine.IncreaseQuantity(quantity);
        }
        else
        {
            _lines.Add(new OrderLine(productId, quantity, unitPrice));
        }

        RecalculateTotal();
    }

    public void Ship(string trackingNumber)
    {
        // Invariant: Can only ship confirmed orders
        if (Status != OrderStatus.Confirmed)
            throw new DomainException("Can only ship confirmed orders");

        Status = OrderStatus.Shipped;
        ShippedAt = DateTime.UtcNow;

        AddDomainEvent(new OrderShippedEvent(Id, trackingNumber));
    }

    public void Cancel(string reason)
    {
        // Invariant: Cannot cancel shipped orders
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Cannot cancel shipped order");

        if (Status == OrderStatus.Cancelled)
            throw new DomainException("Order already cancelled");

        Status = OrderStatus.Cancelled;
        CancelledAt = DateTime.UtcNow;
        CancellationReason = reason;

        AddDomainEvent(new OrderCancelledEvent(Id, reason));
    }

    private void RecalculateTotal()
    {
        Total = _lines.Aggregate(
            Money.Zero("USD"),
            (sum, line) => sum + line.LineTotal);
    }
}
```

### Value Objects

```csharp
// Domain/ValueObjects/Money.cs
namespace OrderFlow.Domain.ValueObjects;

public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new DomainException("Amount cannot be negative");

        Amount = amount;
        Currency = currency ?? throw new ArgumentNullException(nameof(currency));
    }

    public static Money FromDecimal(decimal amount, string currency)
        => new(amount, currency);

    public static Money Zero(string currency)
        => new(0, currency);

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new DomainException("Cannot add different currencies");

        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public static Money operator *(Money money, int quantity)
        => new(money.Amount * quantity, money.Currency);

    public static bool operator <(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new DomainException("Cannot compare different currencies");

        return a.Amount < b.Amount;
    }

    public static bool operator >(Money a, Money b) => b < a;

    public override string ToString() => $"{Amount:F2} {Currency}";
}

// Domain/ValueObjects/ShippingAddress.cs
public record ShippingAddress
{
    public string Street { get; }
    public string City { get; }
    public string State { get; }
    public string PostalCode { get; }
    public string Country { get; }

    public ShippingAddress(
        string street, string city, string state,
        string postalCode, string country)
    {
        if (string.IsNullOrWhiteSpace(street))
            throw new DomainException("Street is required");
        if (string.IsNullOrWhiteSpace(city))
            throw new DomainException("City is required");
        if (string.IsNullOrWhiteSpace(postalCode))
            throw new DomainException("Postal code is required");

        Street = street;
        City = city;
        State = state ?? "";
        PostalCode = postalCode;
        Country = country ?? "USA";
    }

    public string FormatSingleLine()
        => $"{Street}, {City}, {State} {PostalCode}, {Country}";
}
```

### Strongly-Typed IDs

```csharp
// Domain/ValueObjects/OrderId.cs
namespace OrderFlow.Domain.ValueObjects;

public readonly record struct OrderId
{
    public Guid Value { get; }

    public OrderId(Guid value)
    {
        if (value == Guid.Empty)
            throw new ArgumentException("OrderId cannot be empty", nameof(value));

        Value = value;
    }

    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value) => new(value);
    public override string ToString() => Value.ToString();
}

// Prevents this kind of bug:
// void ProcessOrder(int orderId, int customerId) { }
// ProcessOrder(customerId, orderId);  // Compiler allows this!

// With strongly-typed IDs:
// void ProcessOrder(OrderId orderId, CustomerId customerId) { }
// ProcessOrder(customerId, orderId);  // Compiler error!
```

---

## Domain Events

### What Are Domain Events?

Facts about things that happened in the domain. They're named in past tense because they've already occurred.

```csharp
// Domain/Events/OrderPlacedEvent.cs
namespace OrderFlow.Domain.Events;

public record OrderPlacedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    Money Total,
    List<OrderLineSnapshot> Lines) : IDomainEvent
{
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}

public record OrderLineSnapshot(ProductId ProductId, int Quantity);

// Other events
public record OrderConfirmedEvent(OrderId OrderId) : IDomainEvent;
public record OrderShippedEvent(OrderId OrderId, string TrackingNumber) : IDomainEvent;
public record OrderCancelledEvent(OrderId OrderId, string Reason) : IDomainEvent;
```

### Publishing Domain Events

```csharp
// Domain/Common/AggregateRoot.cs
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent @event)
    {
        _domainEvents.Add(@event);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Infrastructure/Persistence/OrderDbContext.cs
public class OrderDbContext : DbContext
{
    private readonly IMediator _mediator;

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Publish domain events before saving
        var aggregates = ChangeTracker
            .Entries<AggregateRoot>()
            .Select(e => e.Entity)
            .Where(e => e.DomainEvents.Any())
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();

        foreach (var aggregate in aggregates)
        {
            aggregate.ClearDomainEvents();
        }

        var result = await base.SaveChangesAsync(ct);

        // Dispatch events after successful save
        foreach (var @event in events)
        {
            await _mediator.Publish(@event, ct);
        }

        return result;
    }
}
```

### Handling Domain Events

```csharp
// Application/Orders/EventHandlers/OrderPlacedHandler.cs
public class OrderPlacedHandler : INotificationHandler<OrderPlacedEvent>
{
    private readonly IInventoryService _inventory;
    private readonly IEmailService _email;

    public async Task Handle(OrderPlacedEvent notification, CancellationToken ct)
    {
        // Reserve inventory
        await _inventory.ReserveStockAsync(
            notification.Lines.Select(l => new StockReservation(l.ProductId, l.Quantity)),
            notification.OrderId,
            ct);

        // Send confirmation email
        await _email.SendOrderConfirmationAsync(
            notification.CustomerId,
            notification.OrderId,
            notification.Total,
            ct);
    }
}
```

---

## Anti-Corruption Layer

When integrating with external systems, don't let their model pollute yours.

```csharp
// Infrastructure/ExternalServices/LegacyErpAdapter.cs
namespace OrderFlow.Infrastructure.ExternalServices;

// The legacy ERP has a terrible API
public class LegacyErpClient
{
    public ErpOrderResponse CreateErpOrder(ErpOrderRequest request)
    {
        // Returns things like:
        // { "ord_num": "00012345", "cust_code": "C123", "stat": 1, "amt": "99.99" }
    }
}

// Anti-corruption layer translates to our model
public class ErpOrderAdapter : IExternalOrderService
{
    private readonly LegacyErpClient _client;

    public async Task<ExternalOrderResult> SubmitOrderAsync(Order order)
    {
        // Translate our model to ERP's model
        var erpRequest = new ErpOrderRequest
        {
            CustCode = TranslateCustomerId(order.CustomerId),
            OrdLines = order.Lines.Select(l => new ErpOrderLine
            {
                ItemCode = TranslateProductId(l.ProductId),
                Qty = l.Quantity,
                Amt = l.UnitPrice.Amount.ToString("F2")
            }).ToList(),
            ShipAddr = FormatAddress(order.ShippingAddress)
        };

        var response = await _client.CreateErpOrder(erpRequest);

        // Translate ERP's response back to our model
        return new ExternalOrderResult
        {
            ExternalOrderId = response.OrdNum,
            Status = TranslateStatus(response.Stat),
            SubmittedAt = DateTime.UtcNow
        };
    }

    private string TranslateCustomerId(CustomerId id) =>
        $"C{id.Value:D6}";

    private string TranslateProductId(ProductId id) =>
        $"P{id.Value:D8}";

    private OrderStatus TranslateStatus(int erpStatus) => erpStatus switch
    {
        1 => OrderStatus.Confirmed,
        2 => OrderStatus.Processing,
        9 => OrderStatus.Failed,
        _ => OrderStatus.Unknown
    };
}
```

---

## Hands-On Exercise: Model OrderFlow's Domain

### Step 1: Run an Event Storming Session (Even Solo)

Create sticky notes (or use a tool like Miro) for:

1. **Events** (orange): What happens in the order lifecycle?
   - Order Placed, Order Confirmed, Payment Received, Stock Reserved...

2. **Commands** (blue): What triggers these events?
   - Place Order, Confirm Order, Process Payment...

3. **Actors** (yellow): Who/what issues commands?
   - Customer, Admin, Payment Processor, Inventory System...

4. **Policies** (purple): What rules trigger reactions?
   - When Order Placed → Reserve Stock
   - When Payment Failed → Cancel Order

### Step 2: Identify Aggregates

Draw boundaries around related events:
- Order Aggregate: Order lifecycle
- Inventory Aggregate: Stock management
- Payment Aggregate: Payment processing

### Step 3: Implement One Aggregate

Build the Order aggregate with:
- Proper invariant enforcement
- Value objects (Money, Address)
- Domain events
- No infrastructure dependencies

### Step 4: Create a Ubiquitous Language Glossary

```markdown
# OrderFlow Ubiquitous Language

## Order Context

| Term | Definition |
|------|------------|
| Order | A customer's request to purchase products |
| Order Line | A single product with quantity in an order |
| Place Order | Customer submits order for processing |
| Confirm Order | System validates and accepts the order |
| Ship Order | Warehouse sends the order |
| Cancel Order | Order terminated before shipping |
```

---

## Deliverables

1. **Event Storming Artifacts**: Screenshots/exports of your event storm
2. **Context Map**: Diagram showing bounded contexts and relationships
3. **Aggregate Implementation**: Order aggregate with tests
4. **Ubiquitous Language Glossary**: Terms used in your domain

---

## Reflection Questions

1. How does DDD help when requirements are ambiguous?
2. When would you NOT use DDD?
3. How do you know if you have the right aggregate boundaries?
4. What's the difference between a Domain Event and an Integration Event?

---

## Resources

### Books
- [Domain-Driven Design Distilled](https://www.oreilly.com/library/view/domain-driven-design-distilled/9780134434964/) by Vaughn Vernon
- [Implementing Domain-Driven Design](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/) by Vaughn Vernon
- [Domain-Driven Design](https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/) by Eric Evans (The Blue Book)

### Videos
- [Alberto Brandolini - Event Storming](https://www.youtube.com/watch?v=mLXQIYEwK24)
- [Vaughn Vernon - Effective Aggregate Design](https://www.youtube.com/watch?v=8M_lUULm-dU)
- [Eric Evans - DDD Today](https://www.youtube.com/watch?v=o_jhdz-8fQ4)

### Tools
- [Miro](https://miro.com/) - For Event Storming
- [EventStorming.com](https://www.eventstorming.com/) - Official resources
- [Context Mapper](https://contextmapper.org/) - DDD modeling tool

---

**Next:** [Part 3 — Service Boundaries](./part-3.md)
