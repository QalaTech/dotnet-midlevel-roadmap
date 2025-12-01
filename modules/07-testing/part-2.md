# 07 — Testing Mastery: Unit, Integration, Contract

## Part 2 / 4 — Unit Testing Domain & Application

---

## Why This Matters

Unit tests are the foundation of your test suite:
- **Fastest feedback** — Run in milliseconds
- **Pinpoint failures** — Know exactly what broke
- **Design pressure** — Hard-to-test code is usually poorly designed
- **Living documentation** — Tests show how code should be used

But unit tests done wrong create:
- Brittle suites that break on every refactor
- False confidence (everything mocked, nothing tested)
- Maintenance nightmares (change one line, fix 50 tests)

---

## Core Concept #6 — Testing Domain Entities

### Why This Matters

Domain entities contain your most important business logic. A bug here means wrong data, wrong decisions, wrong outcomes.

### What Goes Wrong Without This

**War story — The Shipping Bug:**
Order entity allowed shipping before payment verification was complete. No unit test for this rule. Bug discovered when 500 orders shipped to customers who never paid. Chargebacks cost $50,000.

### Testing Business Rules

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();

    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal Total => _items.Sum(i => i.Subtotal);
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public Order(Guid customerId)
    {
        if (customerId == Guid.Empty)
            throw new DomainException("Customer ID is required");

        Id = Guid.NewGuid();
        CustomerId = customerId;
        Status = OrderStatus.Pending;
    }

    public Result AddItem(Guid productId, int quantity, decimal price)
    {
        if (Status != OrderStatus.Pending)
            return Result.Failure("Cannot modify non-pending order");

        if (quantity <= 0)
            return Result.Failure("Quantity must be positive");

        if (price < 0)
            return Result.Failure("Price cannot be negative");

        _items.Add(new OrderItem(productId, quantity, price));
        return Result.Success();
    }

    public Result MarkAsPaid()
    {
        if (Status != OrderStatus.Pending)
            return Result.Failure($"Cannot pay order in {Status} status");

        if (!_items.Any())
            return Result.Failure("Cannot pay empty order");

        Status = OrderStatus.Paid;
        return Result.Success();
    }

    public Result Ship()
    {
        if (Status != OrderStatus.Paid)
            return Result.Failure("Cannot ship unpaid order");

        Status = OrderStatus.Shipped;
        return Result.Success();
    }
}
```

```csharp
// Tests covering all business rules
public class Order_Construction
{
    [Fact]
    public void Requires_customer_id()
    {
        var act = () => new Order(Guid.Empty);

        act.Should().Throw<DomainException>()
            .WithMessage("*Customer ID*required*");
    }

    [Fact]
    public void Generates_unique_id()
    {
        var order1 = new Order(Guid.NewGuid());
        var order2 = new Order(Guid.NewGuid());

        order1.Id.Should().NotBe(order2.Id);
    }

    [Fact]
    public void Starts_in_pending_status()
    {
        var order = new Order(Guid.NewGuid());

        order.Status.Should().Be(OrderStatus.Pending);
    }

    [Fact]
    public void Has_empty_items_initially()
    {
        var order = new Order(Guid.NewGuid());

        order.Items.Should().BeEmpty();
        order.Total.Should().Be(0);
    }
}

public class Order_AddItem
{
    private readonly Order _order;

    public Order_AddItem()
    {
        _order = new Order(Guid.NewGuid());
    }

    [Fact]
    public void Adds_item_to_order()
    {
        var productId = Guid.NewGuid();

        var result = _order.AddItem(productId, quantity: 2, price: 50m);

        result.IsSuccess.Should().BeTrue();
        _order.Items.Should().ContainSingle()
            .Which.ProductId.Should().Be(productId);
    }

    [Fact]
    public void Calculates_total_correctly()
    {
        _order.AddItem(Guid.NewGuid(), quantity: 2, price: 50m);
        _order.AddItem(Guid.NewGuid(), quantity: 3, price: 30m);

        _order.Total.Should().Be(190m);  // (2 * 50) + (3 * 30)
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public void Rejects_non_positive_quantity(int quantity)
    {
        var result = _order.AddItem(Guid.NewGuid(), quantity, price: 50m);

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Quantity");
    }

    [Fact]
    public void Rejects_negative_price()
    {
        var result = _order.AddItem(Guid.NewGuid(), quantity: 1, price: -10m);

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Price");
    }

    [Fact]
    public void Cannot_add_to_paid_order()
    {
        _order.AddItem(Guid.NewGuid(), 1, 50m);
        _order.MarkAsPaid();

        var result = _order.AddItem(Guid.NewGuid(), 1, 50m);

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("non-pending");
    }
}

public class Order_Ship
{
    [Fact]
    public void Paid_order_can_ship()
    {
        var order = CreatePaidOrder();

        var result = order.Ship();

        result.IsSuccess.Should().BeTrue();
        order.Status.Should().Be(OrderStatus.Shipped);
    }

    [Fact]
    public void Pending_order_cannot_ship()
    {
        var order = new Order(Guid.NewGuid());
        order.AddItem(Guid.NewGuid(), 1, 50m);

        var result = order.Ship();

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("unpaid");
    }

    [Fact]
    public void Already_shipped_order_cannot_ship_again()
    {
        var order = CreatePaidOrder();
        order.Ship();

        var result = order.Ship();

        result.IsFailure.Should().BeTrue();
    }

    private static Order CreatePaidOrder()
    {
        var order = new Order(Guid.NewGuid());
        order.AddItem(Guid.NewGuid(), 1, 50m);
        order.MarkAsPaid();
        return order;
    }
}
```

### Testing Value Objects

```csharp
public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new DomainException("Amount cannot be negative");

        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new DomainException("Currency must be 3-letter code");

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new DomainException("Cannot add different currencies");

        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public static Money Zero(string currency) => new(0, currency);
}

public class Money_Tests
{
    [Theory]
    [InlineData(0, "USD")]
    [InlineData(100, "EUR")]
    [InlineData(0.01, "GBP")]
    public void Creates_valid_money(decimal amount, string currency)
    {
        var money = new Money(amount, currency);

        money.Amount.Should().Be(amount);
        money.Currency.Should().Be(currency);
    }

    [Fact]
    public void Rejects_negative_amount()
    {
        var act = () => new Money(-1, "USD");

        act.Should().Throw<DomainException>();
    }

    [Theory]
    [InlineData("")]
    [InlineData("US")]
    [InlineData("USDD")]
    public void Rejects_invalid_currency(string currency)
    {
        var act = () => new Money(100, currency);

        act.Should().Throw<DomainException>();
    }

    [Fact]
    public void Normalizes_currency_to_uppercase()
    {
        var money = new Money(100, "usd");

        money.Currency.Should().Be("USD");
    }

    [Fact]
    public void Adds_same_currency()
    {
        var a = new Money(100, "USD");
        var b = new Money(50, "USD");

        var result = a + b;

        result.Amount.Should().Be(150);
        result.Currency.Should().Be("USD");
    }

    [Fact]
    public void Cannot_add_different_currencies()
    {
        var usd = new Money(100, "USD");
        var eur = new Money(50, "EUR");

        var act = () => usd + eur;

        act.Should().Throw<DomainException>();
    }

    [Fact]
    public void Value_equality()
    {
        var money1 = new Money(100, "USD");
        var money2 = new Money(100, "USD");

        money1.Should().Be(money2);
        (money1 == money2).Should().BeTrue();
    }
}
```

---

## Core Concept #7 — Testing Application Services

### Why This Matters

Application services orchestrate domain logic and infrastructure. They coordinate the "how" of use cases.

### What Goes Wrong Without This

**War story — The Silent Failure:**
Order placement handler silently swallowed exceptions from inventory service. Orders accepted, inventory not decremented. Customers received "Order Confirmed" but products were out of stock. 200 angry customer support tickets.

### Testing Command Handlers

```csharp
public class PlaceOrderHandler
{
    private readonly IOrderRepository _orderRepository;
    private readonly IInventoryService _inventoryService;
    private readonly IEventBus _eventBus;

    public PlaceOrderHandler(
        IOrderRepository orderRepository,
        IInventoryService inventoryService,
        IEventBus eventBus)
    {
        _orderRepository = orderRepository;
        _inventoryService = inventoryService;
        _eventBus = eventBus;
    }

    public async Task<Result<Guid>> Handle(PlaceOrderCommand command)
    {
        // Validate inventory
        foreach (var item in command.Items)
        {
            var available = await _inventoryService.CheckAvailabilityAsync(
                item.ProductId, item.Quantity);

            if (!available)
                return Result<Guid>.Failure(
                    $"Product {item.ProductId} not available in quantity {item.Quantity}");
        }

        // Create order
        var order = new Order(command.CustomerId);
        foreach (var item in command.Items)
        {
            order.AddItem(item.ProductId, item.Quantity, item.Price);
        }

        // Reserve inventory
        foreach (var item in command.Items)
        {
            await _inventoryService.ReserveAsync(item.ProductId, item.Quantity);
        }

        // Persist
        await _orderRepository.AddAsync(order);

        // Publish event
        await _eventBus.PublishAsync(new OrderPlacedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Total = order.Total,
            Items = order.Items.Select(i => new OrderItemDto(
                i.ProductId, i.Quantity, i.Price)).ToList()
        });

        return Result<Guid>.Success(order.Id);
    }
}
```

```csharp
public class PlaceOrderHandler_Tests
{
    private readonly FakeOrderRepository _orderRepository;
    private readonly FakeInventoryService _inventoryService;
    private readonly FakeEventBus _eventBus;
    private readonly PlaceOrderHandler _handler;

    public PlaceOrderHandler_Tests()
    {
        _orderRepository = new FakeOrderRepository();
        _inventoryService = new FakeInventoryService();
        _eventBus = new FakeEventBus();
        _handler = new PlaceOrderHandler(
            _orderRepository, _inventoryService, _eventBus);
    }

    [Fact]
    public async Task Places_order_successfully()
    {
        // Arrange
        var productId = Guid.NewGuid();
        _inventoryService.SetAvailable(productId, quantity: 10);

        var command = new PlaceOrderCommand
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new OrderItemDto(productId, 2, 50m) }
        };

        // Act
        var result = await _handler.Handle(command);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();
    }

    [Fact]
    public async Task Persists_order()
    {
        var productId = Guid.NewGuid();
        _inventoryService.SetAvailable(productId, 10);

        var command = new PlaceOrderCommand
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new OrderItemDto(productId, 2, 50m) }
        };

        var result = await _handler.Handle(command);

        var order = await _orderRepository.GetByIdAsync(result.Value);
        order.Should().NotBeNull();
        order!.Items.Should().HaveCount(1);
        order.Total.Should().Be(100m);
    }

    [Fact]
    public async Task Publishes_order_placed_event()
    {
        var productId = Guid.NewGuid();
        var customerId = Guid.NewGuid();
        _inventoryService.SetAvailable(productId, 10);

        var command = new PlaceOrderCommand
        {
            CustomerId = customerId,
            Items = new[] { new OrderItemDto(productId, 2, 50m) }
        };

        var result = await _handler.Handle(command);

        _eventBus.PublishedEvents.Should().ContainSingle()
            .Which.Should().BeOfType<OrderPlacedEvent>()
            .Which.Should().Match<OrderPlacedEvent>(e =>
                e.OrderId == result.Value &&
                e.CustomerId == customerId &&
                e.Total == 100m);
    }

    [Fact]
    public async Task Reserves_inventory()
    {
        var productId = Guid.NewGuid();
        _inventoryService.SetAvailable(productId, 10);

        var command = new PlaceOrderCommand
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new OrderItemDto(productId, 3, 50m) }
        };

        await _handler.Handle(command);

        _inventoryService.GetReserved(productId).Should().Be(3);
    }

    [Fact]
    public async Task Fails_when_insufficient_inventory()
    {
        var productId = Guid.NewGuid();
        _inventoryService.SetAvailable(productId, quantity: 5);

        var command = new PlaceOrderCommand
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new OrderItemDto(productId, quantity: 10, 50m) }
        };

        var result = await _handler.Handle(command);

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("not available");
    }

    [Fact]
    public async Task Does_not_persist_when_validation_fails()
    {
        var productId = Guid.NewGuid();
        _inventoryService.SetAvailable(productId, 0);  // Out of stock

        var command = new PlaceOrderCommand
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new OrderItemDto(productId, 1, 50m) }
        };

        await _handler.Handle(command);

        _orderRepository.Count.Should().Be(0);
    }

    [Fact]
    public async Task Does_not_publish_event_when_validation_fails()
    {
        var productId = Guid.NewGuid();
        _inventoryService.SetAvailable(productId, 0);

        var command = new PlaceOrderCommand
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new OrderItemDto(productId, 1, 50m) }
        };

        await _handler.Handle(command);

        _eventBus.PublishedEvents.Should().BeEmpty();
    }
}
```

### Fake Implementations

```csharp
public class FakeOrderRepository : IOrderRepository
{
    private readonly Dictionary<Guid, Order> _orders = new();

    public int Count => _orders.Count;

    public Task<Order?> GetByIdAsync(Guid id)
    {
        _orders.TryGetValue(id, out var order);
        return Task.FromResult(order);
    }

    public Task AddAsync(Order order)
    {
        _orders[order.Id] = order;
        return Task.CompletedTask;
    }

    public Task UpdateAsync(Order order)
    {
        _orders[order.Id] = order;
        return Task.CompletedTask;
    }

    // Test helpers
    public void Seed(Order order) => _orders[order.Id] = order;
    public List<Order> GetAll() => _orders.Values.ToList();
}

public class FakeInventoryService : IInventoryService
{
    private readonly Dictionary<Guid, int> _available = new();
    private readonly Dictionary<Guid, int> _reserved = new();

    public void SetAvailable(Guid productId, int quantity)
    {
        _available[productId] = quantity;
    }

    public Task<bool> CheckAvailabilityAsync(Guid productId, int quantity)
    {
        var available = _available.GetValueOrDefault(productId, 0);
        var reserved = _reserved.GetValueOrDefault(productId, 0);
        return Task.FromResult(available - reserved >= quantity);
    }

    public Task ReserveAsync(Guid productId, int quantity)
    {
        var current = _reserved.GetValueOrDefault(productId, 0);
        _reserved[productId] = current + quantity;
        return Task.CompletedTask;
    }

    public int GetReserved(Guid productId) =>
        _reserved.GetValueOrDefault(productId, 0);
}

public class FakeEventBus : IEventBus
{
    public List<object> PublishedEvents { get; } = new();

    public Task PublishAsync<TEvent>(TEvent @event) where TEvent : class
    {
        PublishedEvents.Add(@event!);
        return Task.CompletedTask;
    }
}
```

---

## Core Concept #8 — Testing Event Handlers

### Why This Matters

Event handlers react to things that happened. They must be idempotent and handle duplicates gracefully.

### Testing Message Consumers

```csharp
public class OrderPlacedHandler : IConsumer<OrderPlacedEvent>
{
    private readonly IProcessedMessageRepository _processedMessages;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrderPlacedHandler> _logger;

    public async Task Consume(ConsumeContext<OrderPlacedEvent> context)
    {
        var message = context.Message;

        // Idempotency check
        if (await _processedMessages.ExistsAsync(message.EventId))
        {
            _logger.LogInformation(
                "Message {EventId} already processed, skipping",
                message.EventId);
            return;
        }

        // Send confirmation email
        await _emailService.SendOrderConfirmationAsync(
            message.CustomerEmail,
            message.OrderId,
            message.Total);

        // Mark as processed
        await _processedMessages.MarkAsProcessedAsync(message.EventId);
    }
}
```

```csharp
public class OrderPlacedHandler_Tests
{
    private readonly FakeProcessedMessageRepository _processedMessages;
    private readonly FakeEmailService _emailService;
    private readonly OrderPlacedHandler _handler;

    public OrderPlacedHandler_Tests()
    {
        _processedMessages = new FakeProcessedMessageRepository();
        _emailService = new FakeEmailService();
        _handler = new OrderPlacedHandler(
            _processedMessages,
            _emailService,
            NullLogger<OrderPlacedHandler>.Instance);
    }

    [Fact]
    public async Task Sends_confirmation_email()
    {
        var @event = new OrderPlacedEvent
        {
            EventId = Guid.NewGuid(),
            OrderId = Guid.NewGuid(),
            CustomerEmail = "customer@example.com",
            Total = 150m
        };

        await _handler.Consume(CreateContext(@event));

        _emailService.SentEmails.Should().ContainSingle()
            .Which.Should().Match<SentEmail>(e =>
                e.To == "customer@example.com" &&
                e.Subject.Contains("Order Confirmation"));
    }

    [Fact]
    public async Task Marks_message_as_processed()
    {
        var eventId = Guid.NewGuid();
        var @event = new OrderPlacedEvent { EventId = eventId };

        await _handler.Consume(CreateContext(@event));

        (await _processedMessages.ExistsAsync(eventId)).Should().BeTrue();
    }

    [Fact]
    public async Task Skips_duplicate_messages()
    {
        var eventId = Guid.NewGuid();
        var @event = new OrderPlacedEvent
        {
            EventId = eventId,
            CustomerEmail = "test@example.com"
        };

        // Process first time
        await _handler.Consume(CreateContext(@event));

        // Process second time (duplicate)
        await _handler.Consume(CreateContext(@event));

        // Should only send one email
        _emailService.SentEmails.Should().HaveCount(1);
    }

    [Fact]
    public async Task Does_not_mark_processed_if_email_fails()
    {
        var eventId = Guid.NewGuid();
        _emailService.ShouldFail = true;

        var @event = new OrderPlacedEvent { EventId = eventId };

        var act = () => _handler.Consume(CreateContext(@event));

        await act.Should().ThrowAsync<EmailException>();
        (await _processedMessages.ExistsAsync(eventId)).Should().BeFalse();
    }

    private static ConsumeContext<OrderPlacedEvent> CreateContext(OrderPlacedEvent @event)
    {
        var mock = Substitute.For<ConsumeContext<OrderPlacedEvent>>();
        mock.Message.Returns(@event);
        return mock;
    }
}
```

---

## Core Concept #9 — Testing Async Code

### Why This Matters

Async code has subtle failure modes:
- Exceptions lost in fire-and-forget
- Race conditions
- Deadlocks with .Result or .Wait()

### Testing Async Patterns

```csharp
public class Async_Testing_Patterns
{
    [Fact]
    public async Task Async_method_returns_result()
    {
        var service = new OrderService();

        var result = await service.GetOrderAsync(Guid.NewGuid());

        result.Should().BeNull();  // Doesn't exist
    }

    [Fact]
    public async Task Async_method_throws_exception()
    {
        var service = new OrderService();

        var act = () => service.ProcessInvalidOrderAsync();

        await act.Should().ThrowAsync<InvalidOperationException>()
            .WithMessage("*invalid*");
    }

    [Fact]
    public async Task Async_method_completes_within_timeout()
    {
        var service = new SlowService();

        var task = service.SlowOperationAsync();

        // Fails if operation takes too long
        await task.Should().CompleteWithinAsync(TimeSpan.FromSeconds(1));
    }

    [Fact]
    public async Task Parallel_operations_all_complete()
    {
        var tasks = Enumerable.Range(1, 10)
            .Select(i => ProcessItemAsync(i))
            .ToList();

        await Task.WhenAll(tasks);

        tasks.Should().AllSatisfy(t => t.IsCompletedSuccessfully.Should().BeTrue());
    }

    [Fact]
    public async Task Cancellation_is_respected()
    {
        var cts = new CancellationTokenSource();
        var service = new CancellableService();

        var task = service.LongRunningAsync(cts.Token);
        cts.Cancel();

        var act = () => task;

        await act.Should().ThrowAsync<OperationCanceledException>();
    }
}
```

### Testing Time-Dependent Code

```csharp
// Inject time abstraction
public interface ISystemClock
{
    DateTime UtcNow { get; }
}

public class SystemClock : ISystemClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

public class FakeClock : ISystemClock
{
    public DateTime UtcNow { get; set; } = DateTime.UtcNow;

    public void Advance(TimeSpan duration) => UtcNow = UtcNow.Add(duration);
}

// Service using clock
public class OrderExpirationService
{
    private readonly ISystemClock _clock;
    private readonly IOrderRepository _orders;

    public OrderExpirationService(ISystemClock clock, IOrderRepository orders)
    {
        _clock = clock;
        _orders = orders;
    }

    public async Task<List<Order>> GetExpiredOrdersAsync()
    {
        var cutoff = _clock.UtcNow.AddHours(-24);
        return await _orders.GetPendingOrdersBeforeAsync(cutoff);
    }
}

// Tests
public class OrderExpirationService_Tests
{
    [Fact]
    public async Task Returns_orders_older_than_24_hours()
    {
        var clock = new FakeClock { UtcNow = new DateTime(2024, 1, 15, 12, 0, 0) };
        var repo = new FakeOrderRepository();

        // Order from 25 hours ago (should be expired)
        var oldOrder = CreateOrder(createdAt: clock.UtcNow.AddHours(-25));
        repo.Seed(oldOrder);

        // Order from 23 hours ago (should not be expired)
        var newOrder = CreateOrder(createdAt: clock.UtcNow.AddHours(-23));
        repo.Seed(newOrder);

        var service = new OrderExpirationService(clock, repo);

        var expired = await service.GetExpiredOrdersAsync();

        expired.Should().ContainSingle()
            .Which.Id.Should().Be(oldOrder.Id);
    }

    [Fact]
    public async Task Boundary_exactly_24_hours()
    {
        var clock = new FakeClock { UtcNow = new DateTime(2024, 1, 15, 12, 0, 0) };
        var repo = new FakeOrderRepository();

        // Exactly 24 hours old — should NOT be expired (boundary condition)
        var exactlyAtCutoff = CreateOrder(createdAt: clock.UtcNow.AddHours(-24));
        repo.Seed(exactlyAtCutoff);

        var service = new OrderExpirationService(clock, repo);

        var expired = await service.GetExpiredOrdersAsync();

        expired.Should().BeEmpty();
    }
}
```

---

## Core Concept #10 — When NOT to Mock

### Why This Matters

Over-mocking creates tests that pass but don't prove anything.

### Anti-Patterns

```csharp
// ❌ WRONG: Mocking the thing you're testing
[Fact]
public void Order_total_is_calculated()
{
    var mockOrder = new Mock<Order>();
    mockOrder.Setup(o => o.Total).Returns(100m);

    mockOrder.Object.Total.Should().Be(100m);  // You tested your mock, not your code!
}

// ❌ WRONG: Mocking implementation details
[Fact]
public async Task Handler_calls_repository()
{
    var mockRepo = new Mock<IOrderRepository>();
    var handler = new GetOrderHandler(mockRepo.Object);

    await handler.Handle(new GetOrderQuery { Id = Guid.NewGuid() });

    // This tests HOW, not WHAT
    mockRepo.Verify(r => r.GetByIdAsync(It.IsAny<Guid>()), Times.Once);
}

// ❌ WRONG: Mocking value objects
[Fact]
public void Price_calculation()
{
    var mockMoney = new Mock<Money>();  // Don't mock value objects!
    // ...
}
```

### When to Use Real Implementations

```csharp
// ✅ Use real implementations for:
// 1. Value objects (Money, Address, etc.)
// 2. Domain entities (Order, Customer, etc.)
// 3. Pure functions
// 4. Internal helper classes

// ✅ Use fakes for:
// 1. Repositories (use in-memory dictionary)
// 2. External service clients
// 3. Message buses
// 4. Email services

// ✅ Use mocks ONLY for:
// 1. Verifying outgoing interactions (emails sent, events published)
// 2. Simulating failure modes (network errors, timeouts)
```

---

## Hands-On: OrderFlow Unit Tests

### Task 1 — Test Order Entity

Write tests for:
- Order creation (validation, initial state)
- Adding items (success, validation, state transitions)
- Payment (preconditions, state changes)
- Shipping (preconditions, state changes)

### Task 2 — Test Value Objects

Create and test:
- Money (arithmetic, currency validation)
- Address (validation, equality)
- OrderId (strongly-typed ID)

### Task 3 — Test PlaceOrderHandler

Test:
- Success path with all side effects
- Inventory validation failure
- Customer not found
- Event publishing

### Task 4 — Test OrderPlacedHandler

Test:
- Email sending
- Idempotency (duplicate handling)
- Failure scenarios

---

## Deliverables

1. **20 domain entity tests** covering all business rules
2. **Value object tests** for Money, Address, OrderId
3. **Handler tests** for PlaceOrderHandler with fakes
4. **Event handler tests** with idempotency verification

---

## Resources

### Must-Read
- [Vladimir Khorikov: Unit Testing](https://enterprisecraftsmanship.com/posts/unit-testing-best-practices/)
- [Mark Seemann: Code That Fits in Your Head](https://blog.ploeh.dk/)
- [Jimmy Bogard: Vertical Slice Testing](https://jimmybogard.com/vertical-slice-test-fixtures-for-mediatr-and-asp-net-core/)

### Videos
- [Nick Chapsas: Integration Testing Done Right](https://www.youtube.com/watch?v=7roqteWLw4s)
- [Ian Cooper: TDD, Where Did It All Go Wrong](https://www.youtube.com/watch?v=EZ05e7EMOLM)
- [Vladimir Khorikov: Unit Testing Course](https://www.youtube.com/watch?v=f4qqvDQPRd8)

### Books
- [Unit Testing Principles, Practices, and Patterns](https://www.manning.com/books/unit-testing) — Vladimir Khorikov
- [Domain-Driven Design Distilled](https://www.amazon.com/Domain-Driven-Design-Distilled-Vaughn-Vernon/dp/0134434420) — Testing DDD entities

---

## Reflection Questions

1. When should you use a fake vs a mock?
2. How do you test code that uses DateTime.Now?
3. What's wrong with testing that a method was called?
4. How do you make tests less brittle?

---

**Next:** [Part 3 — Integration Testing](./part-3.md)
