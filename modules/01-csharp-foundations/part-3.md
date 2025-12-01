# 01 — Deep C# & .NET Foundations

## Part 3 / 4 — Generics, SOLID, Design Patterns, and Reflection

---

## What You'll Learn

This part covers the abstractions that separate junior code from professional code:
- Generics that enforce type safety and reusability
- SOLID principles (the real understanding, not interview answers)
- Patterns that solve real problems
- Reflection and when to use it

---

## Core Concept #11 — Generics

### Why This Matters

Generics are everywhere in professional .NET:
- `List<T>`, `Dictionary<TKey, TValue>`
- `IRepository<T>`
- `Task<T>`, `IAsyncEnumerable<T>`
- `IRequestHandler<TRequest, TResponse>` (MediatR)

Understanding generics lets you:
- Write reusable code
- Enforce type safety at compile time
- Avoid boxing and improve performance
- Design flexible APIs

### What Goes Wrong Without This

**Real scenario:** A developer creates separate repository classes for each entity — `OrderRepository`, `ProductRepository`, `CustomerRepository`. Each is 90% identical. Changes need to be made in multiple places. Bugs get fixed in one but not others.

With generics: one `Repository<T>` that works for all entities.

### Generic Constraints

Constraints tell the compiler what `T` can do:

```csharp
// T must be a class (reference type)
public class Repository<T> where T : class
{
    public void Add(T entity) { }
}

// T must implement an interface
public interface IRepository<T> where T : IEntity
{
    Task<T?> GetByIdAsync(Guid id);
    Task SaveAsync(T entity);
}

// T must have a parameterless constructor
public class Factory<T> where T : new()
{
    public T Create() => new T();
}

// Multiple constraints
public class Handler<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
    where TResponse : class, new()
{
}
```

### Generic Repository Pattern

```csharp
public interface IRepository<T> where T : Entity
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IReadOnlyList<T>> GetAllAsync();
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task AddAsync(T entity);
    void Update(T entity);
    void Remove(T entity);
}

public class EfRepository<T> : IRepository<T> where T : Entity
{
    private readonly DbContext _context;
    private readonly DbSet<T> _set;

    public EfRepository(DbContext context)
    {
        _context = context;
        _set = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(Guid id) =>
        await _set.FindAsync(id);

    public async Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate) =>
        await _set.Where(predicate).ToListAsync();

    // ... other methods
}
```

### Anti-Patterns

```csharp
// BAD: Returning IQueryable from repository (leaks EF concerns)
public interface IRepository<T>
{
    IQueryable<T> Query(); // Consumers now depend on EF
}

// BAD: Generic for the sake of generic
public interface IThingDoer<T>
{
    void DoThing(T thing); // What does this even mean?
}

// GOOD: Generics with clear purpose
public interface IEventPublisher<TEvent> where TEvent : IDomainEvent
{
    Task PublishAsync(TEvent @event);
}
```

---

## Core Concept #12 — SOLID Principles

### Why This Matters

SOLID isn't academic theory. It's a practical checklist for code that:
- Can be changed without breaking other things
- Can be tested in isolation
- Can be understood by other developers
- Doesn't require rewriting when requirements change

### What Goes Wrong Without This

**Real scenario (SRP violation):** An `OrderService` class handles order creation, validation, email sending, inventory updates, and payment processing. To change how emails are sent, you need to modify a 2000-line class. Tests require mocking 15 dependencies.

**Real scenario (DIP violation):** Business logic directly calls `SqlConnection`. To add caching, you have to modify every data access point. Testing requires a real database.

---

### Single Responsibility Principle (SRP)

**Definition:** A class should have one reason to change.

```csharp
// BAD: Multiple responsibilities
public class OrderService
{
    public void CreateOrder(Order order)
    {
        // Validate order
        if (order.Items.Count == 0)
            throw new ValidationException("Order must have items");

        // Save to database
        using var connection = new SqlConnection(_connectionString);
        connection.Execute("INSERT INTO Orders...", order);

        // Send confirmation email
        var smtp = new SmtpClient();
        smtp.Send(new MailMessage(...));

        // Update inventory
        foreach (var item in order.Items)
        {
            connection.Execute("UPDATE Products SET Stock = Stock - @Qty...", item);
        }
    }
}

// GOOD: Single responsibility each
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IOrderValidator _validator;
    private readonly IInventoryService _inventory;
    private readonly INotificationService _notifications;

    public async Task CreateOrderAsync(Order order)
    {
        _validator.Validate(order);
        await _inventory.ReserveStockAsync(order.Items);
        await _repository.AddAsync(order);
        await _notifications.SendOrderConfirmationAsync(order);
    }
}
```

---

### Open/Closed Principle (OCP)

**Definition:** Open for extension, closed for modification.

```csharp
// BAD: Adding new discount types requires modifying this class
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        switch (discountType)
        {
            case "percentage": return order.Total * 0.1m;
            case "fixed": return 10m;
            case "bulk": return order.Items.Count > 10 ? order.Total * 0.2m : 0;
            // Every new discount type = modify this class
            default: return 0;
        }
    }
}

// GOOD: New discounts = new class, no modification
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public class PercentageDiscount : IDiscountStrategy
{
    private readonly decimal _percentage;
    public PercentageDiscount(decimal percentage) => _percentage = percentage;
    public decimal Calculate(Order order) => order.Total * _percentage;
}

public class BulkDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) =>
        order.Items.Count > 10 ? order.Total * 0.2m : 0;
}

// Adding new discount = add new class implementing IDiscountStrategy
```

---

### Liskov Substitution Principle (LSP)

**Definition:** Subtypes must be usable wherever their base type is used.

```csharp
// BAD: Violates LSP
public class Bird
{
    public virtual void Fly() { /* fly */ }
}

public class Penguin : Bird
{
    public override void Fly() =>
        throw new NotSupportedException("Penguins can't fly");
}

// Code that works with Bird will break with Penguin
void MakeBirdFly(Bird bird)
{
    bird.Fly(); // Throws for Penguin!
}

// GOOD: Design hierarchy correctly
public abstract class Bird { }

public interface IFlyable
{
    void Fly();
}

public class Sparrow : Bird, IFlyable
{
    public void Fly() { /* fly */ }
}

public class Penguin : Bird
{
    public void Swim() { /* swim */ }
}
```

---

### Interface Segregation Principle (ISP)

**Definition:** Clients shouldn't depend on methods they don't use.

```csharp
// BAD: Fat interface
public interface IOrderService
{
    void CreateOrder(Order order);
    void CancelOrder(Guid orderId);
    void ShipOrder(Guid orderId);
    void RefundOrder(Guid orderId);
    void GenerateInvoice(Guid orderId);
    void SendShippingNotification(Guid orderId);
}

// A component that only creates orders must depend on all methods

// GOOD: Segregated interfaces
public interface IOrderCreator
{
    Task<Order> CreateAsync(CreateOrderRequest request);
}

public interface IOrderCancellation
{
    Task CancelAsync(Guid orderId, string reason);
}

public interface IOrderShipping
{
    Task ShipAsync(Guid orderId, ShippingDetails details);
    Task NotifyShippedAsync(Guid orderId);
}

// Components depend only on what they need
public class OrderController
{
    private readonly IOrderCreator _creator; // Only this
    // ...
}
```

---

### Dependency Inversion Principle (DIP)

**Definition:** High-level modules shouldn't depend on low-level modules. Both should depend on abstractions.

```csharp
// BAD: High-level depends on low-level
public class OrderProcessor
{
    private readonly SqlOrderRepository _repository; // Concrete class
    private readonly SmtpEmailSender _emailer;       // Concrete class

    public OrderProcessor()
    {
        _repository = new SqlOrderRepository(); // Creates its own dependencies
        _emailer = new SmtpEmailSender();
    }
}

// GOOD: Depends on abstractions
public class OrderProcessor
{
    private readonly IOrderRepository _repository;
    private readonly IEmailSender _emailer;

    public OrderProcessor(IOrderRepository repository, IEmailSender emailer)
    {
        _repository = repository;
        _emailer = emailer;
    }
}

// Registration
services.AddScoped<IOrderRepository, SqlOrderRepository>();
services.AddScoped<IEmailSender, SmtpEmailSender>();
// In tests: services.AddScoped<IEmailSender, FakeEmailSender>();
```

---

## Core Concept #13 — Design Patterns

### Strategy Pattern

**Use when:** Behavior varies by type, role, or context.

```csharp
public interface IShippingCalculator
{
    Money Calculate(Order order);
}

public class StandardShipping : IShippingCalculator
{
    public Money Calculate(Order order) =>
        Money.USD(order.Items.Sum(i => i.Weight) * 0.5m);
}

public class ExpressShipping : IShippingCalculator
{
    public Money Calculate(Order order) =>
        Money.USD(order.Items.Sum(i => i.Weight) * 1.5m + 10);
}

public class FreeShipping : IShippingCalculator
{
    public Money Calculate(Order order) => Money.USD(0);
}

// Usage
public class OrderService
{
    private readonly IShippingCalculator _shipping;

    public OrderService(IShippingCalculator shipping)
    {
        _shipping = shipping;
    }

    public Money GetShippingCost(Order order) =>
        _shipping.Calculate(order);
}
```

### Decorator Pattern

**Use when:** Adding cross-cutting concerns without modifying the original.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id);
}

public class SqlOrderRepository : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id)
    {
        // Database access
    }
}

// Decorator adds caching without modifying SqlOrderRepository
public class CachedOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly IMemoryCache _cache;

    public CachedOrderRepository(IOrderRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order?> GetByIdAsync(Guid id)
    {
        var key = $"order:{id}";
        if (_cache.TryGetValue(key, out Order? cached))
            return cached;

        var order = await _inner.GetByIdAsync(id);
        if (order is not null)
            _cache.Set(key, order, TimeSpan.FromMinutes(5));

        return order;
    }
}

// Decorator adds logging
public class LoggedOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly ILogger<LoggedOrderRepository> _logger;

    public async Task<Order?> GetByIdAsync(Guid id)
    {
        _logger.LogDebug("Getting order {OrderId}", id);
        var result = await _inner.GetByIdAsync(id);
        _logger.LogDebug("Order {OrderId} found: {Found}", id, result is not null);
        return result;
    }
}

// Compose decorators
services.AddScoped<SqlOrderRepository>();
services.AddScoped<IOrderRepository>(sp =>
    new LoggedOrderRepository(
        new CachedOrderRepository(
            sp.GetRequiredService<SqlOrderRepository>(),
            sp.GetRequiredService<IMemoryCache>()),
        sp.GetRequiredService<ILogger<LoggedOrderRepository>>()));
```

### Factory Pattern

**Use when:** Object creation is complex or varies by context.

```csharp
public interface INotificationFactory
{
    INotification Create(NotificationType type);
}

public class NotificationFactory : INotificationFactory
{
    private readonly IServiceProvider _services;

    public INotification Create(NotificationType type) => type switch
    {
        NotificationType.Email => _services.GetRequiredService<EmailNotification>(),
        NotificationType.Sms => _services.GetRequiredService<SmsNotification>(),
        NotificationType.Push => _services.GetRequiredService<PushNotification>(),
        _ => throw new ArgumentException($"Unknown notification type: {type}")
    };
}
```

---

## Core Concept #14 — Reflection

### Why This Matters

Reflection lets programs inspect themselves at runtime. It powers:
- Dependency injection containers
- Serialization (JSON, XML)
- ORM mapping
- Test discovery
- Attribute processing

### When to Use (and Not Use)

| Use Reflection | Avoid Reflection |
|----------------|------------------|
| Framework/library code | Business logic |
| Plugin systems | Performance-critical paths |
| One-time setup | Every request |
| Generic utilities | When generics suffice |

### Basic Reflection

```csharp
// Get type information
var type = typeof(Order);
var properties = type.GetProperties();

foreach (var prop in properties)
{
    Console.WriteLine($"{prop.Name}: {prop.PropertyType.Name}");
}

// Create instance dynamically
var instance = Activator.CreateInstance(type);

// Call method dynamically
var method = type.GetMethod("Place");
method?.Invoke(instance, null);
```

### Attribute Processing

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class RequiredAttribute : Attribute { }

public class Order
{
    [Required]
    public string CustomerName { get; set; }

    public string? Notes { get; set; }
}

// Validator using reflection
public static class Validator
{
    public static List<string> Validate(object obj)
    {
        var errors = new List<string>();
        var properties = obj.GetType().GetProperties();

        foreach (var prop in properties)
        {
            if (prop.GetCustomAttribute<RequiredAttribute>() is not null)
            {
                var value = prop.GetValue(obj);
                if (value is null or "")
                {
                    errors.Add($"{prop.Name} is required");
                }
            }
        }

        return errors;
    }
}
```

### Performance Warning

```csharp
// BAD: Reflection on every call
public string GetValue(object obj, string propertyName)
{
    return obj.GetType().GetProperty(propertyName)?.GetValue(obj)?.ToString() ?? "";
}

// GOOD: Cache reflection results
private static readonly ConcurrentDictionary<(Type, string), PropertyInfo?> _cache = new();

public string GetValueCached(object obj, string propertyName)
{
    var key = (obj.GetType(), propertyName);
    var prop = _cache.GetOrAdd(key, k => k.Item1.GetProperty(k.Item2));
    return prop?.GetValue(obj)?.ToString() ?? "";
}
```

---

## Hands-On: OrderFlow with SOLID

### Refactor OrderFlow to Use Patterns

```csharp
// Strategy: Different pricing for customer types
public interface IPricingStrategy
{
    Money CalculatePrice(Product product, int quantity);
}

public class StandardPricing : IPricingStrategy
{
    public Money CalculatePrice(Product product, int quantity) =>
        product.Price.Multiply(quantity);
}

public class WholesalePricing : IPricingStrategy
{
    public Money CalculatePrice(Product product, int quantity)
    {
        var discount = quantity >= 100 ? 0.8m : quantity >= 50 ? 0.9m : 1m;
        return product.Price.Multiply(quantity) with
        {
            Amount = product.Price.Amount * quantity * discount
        };
    }
}

// Factory: Create orders with the right pricing
public interface IOrderFactory
{
    Order CreateOrder(Customer customer);
}

public class OrderFactory : IOrderFactory
{
    public Order CreateOrder(Customer customer)
    {
        var pricing = customer.Type switch
        {
            CustomerType.Retail => new StandardPricing(),
            CustomerType.Wholesale => new WholesalePricing(),
            _ => new StandardPricing()
        };

        return new Order(customer.Id, pricing);
    }
}
```

---

## Deliverables

1. **Generic repository** for OrderFlow entities
2. **Refactored Order service** following SRP
3. **Pricing strategies** for different customer types
4. **Decorator** for logging repository calls
5. **Unit tests** proving patterns work correctly

---

## Resources

### Must-Read
- [Clean Architecture](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/) — Robert Martin
- [Dependency Injection Principles, Practices, and Patterns](https://www.manning.com/books/dependency-injection-principles-practices-patterns) — Mark Seemann

### Videos
- [Nick Chapsas: SOLID Principles](https://www.youtube.com/watch?v=_jDNAf3CzeY)
- [Zoran Horvat: Design Patterns Playlist](https://www.youtube.com/playlist?list=PLfGYIpXujUqKIYGNcLoJYz6b-oAWvBBIc)
- [Milan Jovanovic: Repository Pattern](https://www.youtube.com/watch?v=h4KIngWVpfU)

### Documentation
- [Microsoft: Generics](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/generics)
- [Microsoft: Reflection](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/reflection-and-attributes/)

### Code Examples
- [eShopOnContainers](https://github.com/dotnet-architecture/eShopOnContainers) — SOLID in practice
- [Clean Architecture Template](https://github.com/jasontaylordev/CleanArchitecture) — Patterns applied

---

## Reflection Questions

1. When does the Repository pattern help, and when does it add unnecessary abstraction?
2. How do you decide between Strategy and Factory patterns?
3. What's the cost of violating DIP in terms of testing and maintenance?
4. When is reflection worth the performance cost?

---

**Next:** [Part 4 — Async Streams, DI Deep Dive, and Mastery Exercises](./part-4.md)
