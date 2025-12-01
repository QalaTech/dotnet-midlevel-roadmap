# 07 — Testing Mastery: Unit, Integration, Contract

## Part 1 / 4 — Testing Foundations

---

## Why This Matters

Tests are not about proving your code works. Tests are about:
- **Documenting behavior** — Tests show what the code should do
- **Enabling change** — Safe refactoring requires tests
- **Catching regressions** — Bugs shouldn't come back
- **Designing APIs** — Writing tests first reveals awkward interfaces

Without a testing strategy, you get:
- Tests that test the wrong things
- Tests that break constantly
- Tests that take forever to run
- Tests that give false confidence

---

## Core Concept #1 — The Testing Pyramid

### Why This Matters

The testing pyramid guides where to invest testing effort:

```
        /\
       /  \        E2E Tests
      /    \       (Few, Slow, Expensive)
     /------\
    /        \     Integration Tests
   /          \    (Some, Medium Speed)
  /------------\
 /              \  Unit Tests
/________________\ (Many, Fast, Cheap)
```

But the pyramid is often misapplied:
- "Unit test" doesn't mean "test every class in isolation"
- "Integration test" doesn't mean "test with mocks for everything external"
- "E2E test" doesn't mean "Selenium test for every scenario"

### What Goes Wrong Without This

**War story — The Inverted Pyramid:**
Team has 2,000 Selenium tests, 50 "unit" tests that mock everything. CI takes 4 hours. Tests constantly fail due to timing issues. Developers stop trusting tests, stop running them locally. Production bugs ship regularly.

**War story — The Ice Cream Cone:**
```
     _________
    |  E2E    |   ← Lots of slow, flaky browser tests
    |_________|
      |  Int  |   ← Few integration tests
      |_______|
        |Unit|    ← Almost no unit tests
        |____|
```
Every change requires updating 15 Selenium tests. Test runs take 2 hours. Developers avoid changes that might break tests.

### The Right Balance

```csharp
// UNIT TESTS: Test business logic in isolation
// Fast, focused, no I/O
public class OrderTests
{
    [Fact]
    public void Cannot_ship_unpaid_order()
    {
        var order = new Order(customerId: Guid.NewGuid());
        order.AddItem(productId: Guid.NewGuid(), quantity: 2, price: 100m);

        var result = order.Ship();

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be("Cannot ship unpaid order");
    }
}

// INTEGRATION TESTS: Test components working together
// Slower, real I/O, verify wiring
public class OrderApiTests : IClassFixture<OrderFlowWebAppFactory>
{
    [Fact]
    public async Task Create_order_returns_201_and_persists()
    {
        var client = _factory.CreateClient();

        var response = await client.PostAsJsonAsync("/orders", new
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 2 } }
        });

        response.StatusCode.Should().Be(HttpStatusCode.Created);

        // Verify it was actually persisted
        var location = response.Headers.Location;
        var getResponse = await client.GetAsync(location);
        getResponse.Should().BeSuccessful();
    }
}

// E2E TESTS: Test critical user journeys
// Slowest, most expensive, highest confidence
// Only for most important flows
[Fact]
public async Task Complete_order_flow()
{
    // Place order → Pay → Ship → Complete
    // This tests the entire system working together
}
```

---

## Core Concept #2 — What to Test

### Why This Matters

Not everything needs a test. Testing the wrong things wastes time and creates maintenance burden.

### What Goes Wrong Without This

**Anti-pattern — Testing Implementation Details:**
```csharp
// ❌ WRONG: Testing how, not what
[Fact]
public void ProcessOrder_calls_repository_save()
{
    var mockRepo = new Mock<IOrderRepository>();
    var service = new OrderService(mockRepo.Object);

    service.ProcessOrder(new Order());

    // This breaks if you refactor, even if behavior is correct
    mockRepo.Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
}
```

**Anti-pattern — Testing Trivial Code:**
```csharp
// ❌ WRONG: Testing auto-generated code
[Fact]
public void Order_has_Id_property()
{
    var order = new Order();
    order.Id = Guid.NewGuid();
    order.Id.Should().NotBeEmpty();
}
```

**Anti-pattern — Testing Framework Code:**
```csharp
// ❌ WRONG: Testing that ASP.NET Core works
[Fact]
public void Controller_returns_Ok_result()
{
    var controller = new OrdersController();
    var result = controller.Get();
    result.Should().BeOfType<OkResult>();  // You're testing the framework
}
```

### What TO Test

```csharp
// ✅ Test business rules (domain logic)
[Fact]
public void Order_total_applies_volume_discount_above_10_items()
{
    var order = new Order(customerId: Guid.NewGuid());
    order.AddItem(productId: Guid.NewGuid(), quantity: 15, unitPrice: 10m);

    order.Total.Should().Be(135m);  // 15 * 10 * 0.9 = 135
}

// ✅ Test edge cases and boundaries
[Theory]
[InlineData(9, 90)]    // Below threshold: no discount
[InlineData(10, 90)]   // At threshold: discount applies
[InlineData(11, 99)]   // Above threshold: discount applies
public void Volume_discount_threshold_is_10_items(int quantity, decimal expectedTotal)
{
    var order = new Order(customerId: Guid.NewGuid());
    order.AddItem(productId: Guid.NewGuid(), quantity: quantity, unitPrice: 10m);

    order.Total.Should().Be(expectedTotal);
}

// ✅ Test failure modes
[Fact]
public void Cannot_add_negative_quantity()
{
    var order = new Order(customerId: Guid.NewGuid());

    var act = () => order.AddItem(Guid.NewGuid(), quantity: -1, unitPrice: 10m);

    act.Should().Throw<DomainException>()
        .WithMessage("*quantity*positive*");
}

// ✅ Test integration points (actual behavior with real dependencies)
[Fact]
public async Task Order_persisted_to_database()
{
    var order = CreateValidOrder();
    await _repository.AddAsync(order);
    await _dbContext.SaveChangesAsync();

    var retrieved = await _repository.GetByIdAsync(order.Id);

    retrieved.Should().BeEquivalentTo(order);
}
```

### The Test Decision Framework

```
Should I write a test for this?
│
├─ Is it business logic / domain rule?
│   └─ YES → Unit test
│
├─ Is it integration with external system?
│   └─ YES → Integration test
│
├─ Is it critical user journey?
│   └─ YES → E2E test (one per journey)
│
├─ Is it trivial getter/setter?
│   └─ NO → Skip it
│
├─ Is it testing the framework?
│   └─ NO → Skip it
│
└─ Does this code have bugs or could it?
    └─ YES → Write a test
    └─ NO → Consider skipping
```

---

## Core Concept #3 — Test Doubles

### Why This Matters

Test doubles replace real dependencies in tests. Using the wrong type causes problems:
- Over-mocking hides bugs
- Under-mocking makes tests slow and flaky
- Wrong double type makes tests brittle

### Types of Test Doubles

```csharp
// STUB: Returns canned data, no verification
public class StubOrderRepository : IOrderRepository
{
    private readonly Order _orderToReturn;

    public StubOrderRepository(Order orderToReturn)
    {
        _orderToReturn = orderToReturn;
    }

    public Task<Order?> GetByIdAsync(Guid id) => Task.FromResult(_orderToReturn);
    public Task AddAsync(Order order) => Task.CompletedTask;
}

// FAKE: Working implementation with shortcuts
public class FakeOrderRepository : IOrderRepository
{
    private readonly Dictionary<Guid, Order> _orders = new();

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

    // Test helper
    public int Count => _orders.Count;
}

// MOCK: Verifies interactions (use sparingly)
public class MockEmailSender
{
    private readonly List<(string To, string Subject)> _sentEmails = new();

    public Task SendAsync(string to, string subject, string body)
    {
        _sentEmails.Add((to, subject));
        return Task.CompletedTask;
    }

    // Verification method
    public void VerifySent(string to, string subject)
    {
        _sentEmails.Should().Contain(e => e.To == to && e.Subject.Contains(subject));
    }
}

// SPY: Records calls for later inspection
public class SpyLogger : ILogger
{
    public List<string> LoggedMessages { get; } = new();

    public void Log(LogLevel level, string message)
    {
        LoggedMessages.Add($"[{level}] {message}");
    }
}
```

### When to Use Each

```csharp
// USE STUBS: When you need to provide data but don't care about interactions
[Fact]
public void Calculate_total_uses_product_prices()
{
    var productRepo = new StubProductRepository(new Product { Price = 50m });
    var calculator = new OrderTotalCalculator(productRepo);

    var total = calculator.Calculate(productId: Guid.NewGuid(), quantity: 3);

    total.Should().Be(150m);
}

// USE FAKES: When you need realistic behavior without real infrastructure
[Fact]
public async Task Order_can_be_retrieved_after_saving()
{
    var repo = new FakeOrderRepository();
    var order = new Order(Guid.NewGuid());

    await repo.AddAsync(order);
    var retrieved = await repo.GetByIdAsync(order.Id);

    retrieved.Should().Be(order);
}

// USE MOCKS: When interaction IS the behavior you're testing
[Fact]
public async Task Order_placed_sends_confirmation_email()
{
    var emailSender = new MockEmailSender();
    var handler = new OrderPlacedHandler(emailSender);

    await handler.Handle(new OrderPlacedEvent
    {
        OrderId = Guid.NewGuid(),
        CustomerEmail = "test@example.com"
    });

    emailSender.VerifySent("test@example.com", "Order Confirmation");
}
```

### What Goes Wrong With Mocking

**War story — The Mock That Lied:**
```csharp
// ❌ Test passes, production fails
[Fact]
public async Task Get_order_returns_order()
{
    var mockRepo = new Mock<IOrderRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
        .ReturnsAsync(new Order { Status = "Active" });

    var service = new OrderService(mockRepo.Object);
    var order = await service.GetOrderAsync(Guid.NewGuid());

    order.Status.Should().Be("Active");
}

// But in production, GetByIdAsync returns null for non-existent orders
// and the service doesn't handle it. NullReferenceException!
```

**The fix — Test with real behavior:**
```csharp
// ✅ Use a fake that behaves realistically
[Fact]
public async Task Get_nonexistent_order_returns_null()
{
    var repo = new FakeOrderRepository();  // Empty repo
    var service = new OrderService(repo);

    var order = await service.GetOrderAsync(Guid.NewGuid());

    order.Should().BeNull();
}
```

---

## Core Concept #4 — Test Project Setup

### Why This Matters

Good test infrastructure reduces friction. Bad infrastructure makes tests painful to write.

### Project Structure

```
OrderFlow/
├── src/
│   ├── OrderFlow.Domain/
│   ├── OrderFlow.Application/
│   ├── OrderFlow.Infrastructure/
│   └── OrderFlow.Api/
├── tests/
│   ├── OrderFlow.Domain.Tests/           # Fast unit tests
│   ├── OrderFlow.Application.Tests/      # Handler/service tests
│   ├── OrderFlow.Integration.Tests/      # API + DB tests
│   ├── OrderFlow.Contract.Tests/         # Pact tests
│   └── OrderFlow.TestHelpers/            # Shared utilities
```

### Essential Packages

```xml
<!-- OrderFlow.TestHelpers.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net8.0</TargetFramework>
        <IsPackable>false</IsPackable>
    </PropertyGroup>

    <ItemGroup>
        <!-- Test framework -->
        <PackageReference Include="xunit" Version="2.9.0" />
        <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />

        <!-- Assertions -->
        <PackageReference Include="FluentAssertions" Version="6.12.0" />

        <!-- Mocking -->
        <PackageReference Include="NSubstitute" Version="5.1.0" />

        <!-- Test data -->
        <PackageReference Include="Bogus" Version="35.6.0" />
        <PackageReference Include="AutoFixture" Version="4.18.1" />
        <PackageReference Include="AutoFixture.Xunit2" Version="4.18.1" />

        <!-- Integration testing -->
        <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
        <PackageReference Include="Testcontainers.PostgreSql" Version="3.9.0" />
        <PackageReference Include="Testcontainers.Redis" Version="3.9.0" />
        <PackageReference Include="Testcontainers.RabbitMq" Version="3.9.0" />
    </ItemGroup>
</Project>
```

### Test Data Builders

```csharp
// Fluent builders for test data
public class OrderBuilder
{
    private Guid _customerId = Guid.NewGuid();
    private readonly List<OrderItem> _items = new();
    private OrderStatus _status = OrderStatus.Pending;
    private DateTime _createdAt = DateTime.UtcNow;

    public OrderBuilder WithCustomer(Guid customerId)
    {
        _customerId = customerId;
        return this;
    }

    public OrderBuilder WithItem(Guid productId, int quantity, decimal price)
    {
        _items.Add(new OrderItem(productId, quantity, price));
        return this;
    }

    public OrderBuilder WithStatus(OrderStatus status)
    {
        _status = status;
        return this;
    }

    public OrderBuilder Paid()
    {
        _status = OrderStatus.Paid;
        return this;
    }

    public OrderBuilder Shipped()
    {
        _status = OrderStatus.Shipped;
        return this;
    }

    public Order Build()
    {
        var order = new Order(_customerId);
        foreach (var item in _items)
        {
            order.AddItem(item.ProductId, item.Quantity, item.Price);
        }
        // Use reflection to set status for testing
        typeof(Order).GetProperty(nameof(Order.Status))!
            .SetValue(order, _status);
        return order;
    }

    // Static factory for common scenarios
    public static Order PaidOrder() => new OrderBuilder()
        .WithItem(Guid.NewGuid(), 2, 100m)
        .Paid()
        .Build();

    public static Order ShippedOrder() => new OrderBuilder()
        .WithItem(Guid.NewGuid(), 1, 50m)
        .Shipped()
        .Build();
}

// Usage in tests
[Fact]
public void Paid_order_can_be_shipped()
{
    var order = new OrderBuilder()
        .WithItem(Guid.NewGuid(), quantity: 2, price: 100m)
        .Paid()
        .Build();

    var result = order.Ship();

    result.IsSuccess.Should().BeTrue();
}
```

### AutoFixture for Random Data

```csharp
public class OrderCustomization : ICustomization
{
    public void Customize(IFixture fixture)
    {
        fixture.Customize<Order>(composer => composer
            .FromFactory(() => new Order(fixture.Create<Guid>()))
            .Without(o => o.Items));  // Don't auto-generate items

        fixture.Customize<OrderItem>(composer => composer
            .With(i => i.Quantity, () => fixture.Create<int>() % 10 + 1)  // 1-10
            .With(i => i.Price, () => Math.Round(fixture.Create<decimal>() % 1000, 2)));
    }
}

// Usage
public class OrderTests
{
    private readonly IFixture _fixture;

    public OrderTests()
    {
        _fixture = new Fixture().Customize(new OrderCustomization());
    }

    [Theory, AutoData]
    public void Order_with_items_has_positive_total(Order order)
    {
        order.AddItem(_fixture.Create<Guid>(), 2, 50m);

        order.Total.Should().BePositive();
    }
}
```

### Bogus for Realistic Data

```csharp
public static class TestData
{
    private static readonly Faker<Customer> CustomerFaker = new Faker<Customer>()
        .RuleFor(c => c.Id, f => Guid.NewGuid())
        .RuleFor(c => c.Email, f => f.Internet.Email())
        .RuleFor(c => c.Name, f => f.Name.FullName())
        .RuleFor(c => c.Phone, f => f.Phone.PhoneNumber())
        .RuleFor(c => c.Address, f => new Address
        {
            Line1 = f.Address.StreetAddress(),
            City = f.Address.City(),
            PostalCode = f.Address.ZipCode(),
            Country = f.Address.Country()
        });

    public static Customer RandomCustomer() => CustomerFaker.Generate();

    public static List<Customer> RandomCustomers(int count) =>
        CustomerFaker.Generate(count);

    private static readonly Faker<Product> ProductFaker = new Faker<Product>()
        .RuleFor(p => p.Id, f => Guid.NewGuid())
        .RuleFor(p => p.Name, f => f.Commerce.ProductName())
        .RuleFor(p => p.Description, f => f.Commerce.ProductDescription())
        .RuleFor(p => p.Price, f => decimal.Parse(f.Commerce.Price()))
        .RuleFor(p => p.Sku, f => f.Commerce.Ean13());

    public static Product RandomProduct() => ProductFaker.Generate();
}
```

---

## Core Concept #5 — Test Naming and Organization

### Why This Matters

Tests are documentation. Good names explain what the system should do. Poor names require reading the test code to understand purpose.

### Naming Conventions

```csharp
// Pattern: Method_Scenario_ExpectedBehavior
// Or: Given_When_Then

// ✅ GOOD: Descriptive names that document behavior
public class Order_Ship
{
    [Fact]
    public void Paid_order_transitions_to_shipped()
    {
        var order = OrderBuilder.PaidOrder();

        order.Ship();

        order.Status.Should().Be(OrderStatus.Shipped);
    }

    [Fact]
    public void Unpaid_order_returns_error()
    {
        var order = new OrderBuilder().Build();  // Pending status

        var result = order.Ship();

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("unpaid");
    }

    [Fact]
    public void Already_shipped_order_returns_error()
    {
        var order = OrderBuilder.ShippedOrder();

        var result = order.Ship();

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("already shipped");
    }
}

// ❌ BAD: Vague names
public class OrderTests
{
    [Fact]
    public void Test1() { }

    [Fact]
    public void ShipTest() { }

    [Fact]
    public void TestShipping() { }
}
```

### Arrange-Act-Assert

```csharp
[Fact]
public async Task Placing_order_publishes_event()
{
    // Arrange — Set up preconditions
    var eventBus = new FakeEventBus();
    var repository = new FakeOrderRepository();
    var handler = new PlaceOrderHandler(repository, eventBus);
    var command = new PlaceOrderCommand
    {
        CustomerId = Guid.NewGuid(),
        Items = new[] { new OrderItemDto(Guid.NewGuid(), 2, 100m) }
    };

    // Act — Execute the behavior being tested
    await handler.Handle(command, CancellationToken.None);

    // Assert — Verify the expected outcome
    eventBus.PublishedEvents.Should().ContainSingle()
        .Which.Should().BeOfType<OrderPlacedEvent>();
}
```

### Test Class Organization

```csharp
// Organize by behavior/feature, not by class
namespace OrderFlow.Domain.Tests.Orders;

public class Creating_an_order
{
    [Fact]
    public void Requires_customer_id() { }

    [Fact]
    public void Starts_in_pending_status() { }

    [Fact]
    public void Has_zero_total_with_no_items() { }
}

public class Adding_items_to_order
{
    [Fact]
    public void Increases_total() { }

    [Fact]
    public void Allows_same_product_multiple_times() { }

    [Fact]
    public void Rejects_negative_quantity() { }

    [Fact]
    public void Rejects_negative_price() { }
}

public class Shipping_an_order
{
    [Fact]
    public void Paid_order_can_be_shipped() { }

    [Fact]
    public void Unpaid_order_cannot_be_shipped() { }

    [Fact]
    public void Records_shipped_date() { }

    [Fact]
    public void Raises_order_shipped_event() { }
}
```

---

## Hands-On: OrderFlow Test Infrastructure

### Task 1 — Set Up Test Projects

Create the test project structure:
- Domain.Tests
- Application.Tests
- Integration.Tests
- TestHelpers

### Task 2 — Build Test Data Helpers

Create builders for:
- Order (with status, items, customer)
- Customer (with address)
- Product (with price, stock)

### Task 3 — Create Fake Implementations

Build fakes for:
- IOrderRepository (in-memory dictionary)
- IEventBus (captures published events)
- IEmailSender (records sent emails)

### Task 4 — Write First Unit Tests

Test Order domain entity:
- Cannot add negative quantity
- Total calculation with items
- Cannot ship unpaid order
- Cannot modify shipped order

---

## Deliverables

1. **Test project structure** with proper separation
2. **Test helpers library** with builders and fakes
3. **Naming conventions document** for your team
4. **10 unit tests** following AAA pattern

---

## Resources

### Must-Read
- [xUnit Documentation](https://xunit.net/)
- [FluentAssertions Documentation](https://fluentassertions.com/)
- [Martin Fowler: Test Double](https://martinfowler.com/bliki/TestDouble.html)

### Videos
- [Nick Chapsas: How to Write Better Unit Tests](https://www.youtube.com/watch?v=EZ05e7EMOLM)
- [Nick Chapsas: Stop Using Mocks](https://www.youtube.com/watch?v=A8-yvAjpJqI)
- [Ian Cooper: TDD, Where Did It All Go Wrong](https://www.youtube.com/watch?v=EZ05e7EMOLM)
- [Kent Beck on Test Desiderata](https://www.youtube.com/watch?v=5LOdKDqdWYU)

### Books
- [Unit Testing Principles, Practices, and Patterns](https://www.manning.com/books/unit-testing) — Vladimir Khorikov
- [The Art of Unit Testing](https://www.manning.com/books/the-art-of-unit-testing-third-edition) — Roy Osherove
- [Growing Object-Oriented Software, Guided by Tests](http://www.growing-object-oriented-software.com/) — Freeman & Pryce

---

## Reflection Questions

1. What's the difference between testing behavior and testing implementation?
2. When is mocking appropriate vs when should you use fakes?
3. How do you decide if something needs a test?
4. What makes a test "flaky" and how do you prevent it?

---

**Next:** [Part 2 — Unit Testing Domain & Application](./part-2.md)
