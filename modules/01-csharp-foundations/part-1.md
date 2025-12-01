# 01 — Deep C# & .NET Foundations

## Part 1 / 4 — Types, Memory, and LINQ Mastery

---

## Why This Module Exists

C# is your primary tool. Everything else — APIs, EF Core, messaging, Clean Architecture — sits on top of your ability to express ideas in C# accurately and efficiently.

Most junior plateaus come from weak C# fundamentals, not framework issues. You can follow tutorials forever and still write code that:
- Allocates memory unnecessarily
- Deadlocks under load
- Confuses value and reference semantics
- Breaks in production but works locally

This module fixes that.

---

## What You'll Build (OrderFlow)

In this part, you'll create the **domain models** for OrderFlow:
- `Money` (value object)
- `Address` (value object)
- `Product` (entity)
- `LineItem` (entity)
- `Order` (aggregate root)
- `Customer` (entity)

These models will exercise every concept in this part: value vs reference types, records, nullable reference types, and LINQ.

---

## Core Concept #1 — Value Types vs Reference Types

### Why This Matters

Understanding memory layout determines:
- Whether your code allocates on stack or heap
- Whether mutations affect the original or a copy
- Whether your hot paths cause GC pressure
- Whether concurrent code is safe

### What Goes Wrong Without This Knowledge

**Real scenario:** A junior developer creates a `Point` struct with X and Y coordinates. They pass it to a method that "moves" the point. The original doesn't change. They spend hours debugging, confused why the mutation didn't work.

**Real scenario:** A developer puts custom structs in a `List<object>`. Every access boxes the struct, creating heap allocations. Under load, GC pauses spike and the API becomes unresponsive.

### Value Types (Structs)

Stored on the **stack** (usually), copied by value.

```csharp
public readonly record struct Money(decimal Amount, string Currency);

var original = new Money(100, "USD");
var copy = original; // Full copy made
// Modifying 'copy' does not affect 'original'
```

**When to use structs:**
- Small, immutable data (< 16 bytes ideally)
- Value semantics required (two `Money(100, "USD")` should be equal)
- High-frequency allocation paths where GC pressure matters

### Reference Types (Classes)

Stored on the **heap**, variables hold a reference.

```csharp
public class Order
{
    public List<LineItem> Items { get; } = new();
}

var order1 = new Order();
var order2 = order1; // Same object, two references
order2.Items.Add(new LineItem()); // Affects order1 too!
```

**When to use classes:**
- Identity matters (two orders with same data are still different orders)
- Mutable state required
- Complex objects with many fields
- Inheritance needed

### Anti-Patterns

```csharp
// BAD: Large mutable struct
public struct LargeData
{
    public byte[] Buffer; // 8 bytes (reference)
    public int Count;     // 4 bytes
    public DateTime Created; // 8 bytes
    public string Name;   // 8 bytes (reference)
    // Every pass-by-value copies 28+ bytes
}

// BAD: Boxing in hot paths
foreach (object item in structList) // Boxing happens here
{
    Process(item);
}

// GOOD: Use generics to avoid boxing
foreach (var item in structList) // No boxing
{
    Process(item);
}
```

---

## Core Concept #2 — Nullable Reference Types (NRT)

### Why This Matters

`NullReferenceException` is the most common runtime error. NRT catches these at compile time.

### What Goes Wrong Without This

**Real scenario:** An API returns a `Customer` object. Somewhere in the chain, it can be null. Three layers deep, code assumes it's not null. In production, a specific edge case returns null. The API crashes. Users see 500 errors. The bug takes hours to trace.

With NRT enabled, the compiler would have warned about this.

### The Two States

```csharp
public class OrderService
{
    // This MUST never be null
    public string OrderNumber { get; }

    // This CAN be null
    public string? Notes { get; set; }

    public OrderService(string orderNumber)
    {
        // Compiler enforces this assignment
        OrderNumber = orderNumber ?? throw new ArgumentNullException(nameof(orderNumber));
    }
}
```

### Professional Patterns

```csharp
// Guard clauses at boundaries
public void ProcessOrder(Order? order)
{
    ArgumentNullException.ThrowIfNull(order);
    // From here, 'order' is known non-null
}

// Null-coalescing for defaults
var displayName = customer.Nickname ?? customer.FullName ?? "Unknown";

// Null-conditional for safe navigation
var city = customer.Address?.City ?? "No city";
```

### Anti-Patterns

```csharp
// BAD: Disabling NRT warnings
#nullable disable // Don't do this

// BAD: Overusing null-forgiving operator
var name = GetName()!; // "Trust me, it's not null" - famous last words

// BAD: Not handling nullable returns
var customer = await _repo.GetByIdAsync(id);
return customer.Name; // NullReferenceException waiting to happen

// GOOD: Handle the null case
var customer = await _repo.GetByIdAsync(id);
if (customer is null)
    return NotFound();
return customer.Name;
```

---

## Core Concept #3 — Records and Immutability

### Why This Matters

Immutable objects:
- Can't be corrupted by concurrent access
- Are easier to reason about (no hidden state changes)
- Work naturally with value semantics
- Are safer to cache

### What Goes Wrong Without This

**Real scenario:** A `Price` object is shared between multiple threads. One thread updates the price while another reads it. The reader sees a partially-updated state — the amount from the new price but the currency from the old. The order is charged in the wrong currency.

With immutable records, this is impossible.

### Records for Domain Values

```csharp
// Immutable by default, value equality built-in
public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }
}

// Usage
var price = new Money(100, "USD");
var tax = new Money(10, "USD");
var total = price.Add(tax); // Returns new Money(110, "USD")
// 'price' is still Money(100, "USD")
```

### Record Structs for Small Values

```csharp
// Stack-allocated, zero GC pressure
public readonly record struct Coordinates(double Latitude, double Longitude);

// Value equality works automatically
var a = new Coordinates(51.5, -0.1);
var b = new Coordinates(51.5, -0.1);
Console.WriteLine(a == b); // True
```

### When to Use What

| Type | Use Case |
|------|----------|
| `record class` | DTOs, events, value objects > 16 bytes |
| `record struct` | Small value objects, coordinates, IDs |
| `class` | Entities with identity, mutable state |
| `struct` | Performance-critical small data |

---

## Core Concept #4 — Deep LINQ Understanding

### Why This Matters

LINQ is how you work with data in C#. Misunderstanding it causes:
- Performance disasters (loading entire tables into memory)
- Subtle bugs (multiple enumeration)
- Confusing errors (client-side evaluation)

### What Goes Wrong Without This

**Real scenario:** A developer writes `_context.Products.ToList().Where(p => p.Price > 100)`. This loads ALL products into memory, then filters. With 1 million products, the server runs out of memory. The correct version filters in SQL.

### IEnumerable vs IQueryable

```csharp
// IEnumerable: In-memory, uses delegates
IEnumerable<Product> inMemory = products.Where(p => p.Price > 100);

// IQueryable: Translates to SQL, uses expression trees
IQueryable<Product> queryable = _context.Products.Where(p => p.Price > 100);
```

The critical difference:

```csharp
// BAD: Filters in memory (loads entire table first)
var expensive = _context.Products
    .ToList()                        // <-- Loads ALL products
    .Where(p => p.Price > 100);      // <-- Filters in C#

// GOOD: Filters in SQL (only loads matching rows)
var expensive = await _context.Products
    .Where(p => p.Price > 100)       // <-- Becomes WHERE clause
    .ToListAsync();                  // <-- Only loads filtered results
```

### Deferred Execution

LINQ queries don't execute until you enumerate them:

```csharp
var query = products.Where(p => p.Price > 100); // Nothing happens yet

foreach (var p in query) // NOW it executes
{
    Console.WriteLine(p.Name);
}

// Danger: Multiple enumeration
var query = GetProducts().Where(p => p.Active);
var count = query.Count();      // Executes query
var first = query.First();      // Executes query AGAIN

// Fix: Materialize once
var list = query.ToList();
var count = list.Count;         // Uses cached list
var first = list.First();       // Uses cached list
```

### Essential LINQ Operations

```csharp
// Projection (SELECT)
var names = products.Select(p => p.Name);
var dtos = products.Select(p => new ProductDto(p.Id, p.Name));

// Filtering (WHERE)
var active = products.Where(p => p.IsActive);

// Sorting (ORDER BY)
var sorted = products.OrderBy(p => p.Name).ThenByDescending(p => p.Price);

// Grouping (GROUP BY)
var byCategory = products.GroupBy(p => p.CategoryId);

// Aggregation (COUNT, SUM, AVG)
var total = lineItems.Sum(li => li.Quantity * li.UnitPrice);
var hasAny = products.Any(p => p.Stock == 0);

// Pagination (OFFSET/FETCH)
var page = products.Skip(20).Take(10);

// Joining
var orderDetails = orders
    .Join(customers, o => o.CustomerId, c => c.Id,
          (order, customer) => new { order, customer });
```

---

## Core Concept #5 — Async/Await Fundamentals

### Why This Matters

Async isn't about speed. It's about **scalability**. Without async:
- Each request ties up a thread
- Thread pool exhaustion kills throughput
- Your API stops responding under load

### What Goes Wrong Without This

**Real scenario:** A developer calls `.Result` on an async method in a controller. Under light load, it works. Under heavy load, all thread pool threads are blocked waiting for I/O. New requests queue up. The API becomes unresponsive even though the database is fine.

### The Fundamentals

```csharp
// GOOD: Async all the way
public async Task<Order> GetOrderAsync(Guid id)
{
    var order = await _context.Orders.FindAsync(id);
    return order;
}

// BAD: Blocking on async (can deadlock, wastes threads)
public Order GetOrder(Guid id)
{
    var order = _context.Orders.FindAsync(id).Result; // DANGER
    return order;
}
```

### When to Use Async

Use async for **I/O-bound operations**:
- Database queries
- HTTP calls
- File operations
- Message queue operations

Don't use async for **CPU-bound operations** (use `Task.Run` instead):

```csharp
// I/O-bound: Use async naturally
var data = await _httpClient.GetStringAsync(url);

// CPU-bound: Offload to thread pool
var result = await Task.Run(() => HeavyComputation(data));
```

### Common Mistakes

```csharp
// BAD: async void (fire-and-forget, exceptions lost)
public async void ProcessOrder(Order order) // Never do this
{
    await _service.ProcessAsync(order);
}

// GOOD: async Task
public async Task ProcessOrderAsync(Order order)
{
    await _service.ProcessAsync(order);
}

// BAD: Not awaiting (task might not complete)
public Task ProcessOrderAsync(Order order)
{
    _service.ProcessAsync(order); // Missing await!
    return Task.CompletedTask;
}

// GOOD: Await properly
public async Task ProcessOrderAsync(Order order)
{
    await _service.ProcessAsync(order);
}
```

---

## Hands-On: OrderFlow Domain Models

Create these models for OrderFlow. They exercise every concept above.

### Money Value Object

```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money USD(decimal amount) => new(amount, "USD");
    public static Money GBP(decimal amount) => new(amount, "GBP");

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new CurrencyMismatchException(Currency, other.Currency);
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(int quantity) => this with { Amount = Amount * quantity };
}
```

### Address Value Object

```csharp
public record Address(
    string Line1,
    string? Line2,
    string City,
    string PostalCode,
    string Country)
{
    public string FullAddress => string.Join(", ",
        new[] { Line1, Line2, City, PostalCode, Country }
        .Where(s => !string.IsNullOrWhiteSpace(s)));
}
```

### Product Entity

```csharp
public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public string Sku { get; private set; }
    public Money Price { get; private set; }
    public int StockLevel { get; private set; }

    private Product() { } // EF Core

    public Product(string name, string sku, Money price)
    {
        Id = Guid.NewGuid();
        Name = name ?? throw new ArgumentNullException(nameof(name));
        Sku = sku ?? throw new ArgumentNullException(nameof(sku));
        Price = price;
        StockLevel = 0;
    }

    public void AddStock(int quantity)
    {
        if (quantity <= 0) throw new ArgumentException("Quantity must be positive");
        StockLevel += quantity;
    }

    public void ReserveStock(int quantity)
    {
        if (quantity > StockLevel)
            throw new InsufficientStockException(Sku, quantity, StockLevel);
        StockLevel -= quantity;
    }
}
```

---

## Deliverables

1. **OrderFlow.Domain project** with:
   - `Money` record struct
   - `Address` record
   - `Product` class with stock management
   - `LineItem` class
   - `Order` aggregate with line items
   - `Customer` class

2. **Unit tests** proving:
   - Value equality for Money and Address
   - Product stock management (happy path and exceptions)
   - Order total calculation using LINQ

3. **Benchmark** comparing:
   - `record struct Money` vs `class Money` for 1M operations
   - Show allocation difference

---

## Resources

### Must-Read
- [C# in Depth, Chapter 2-4](https://csharpindepth.com/) — Jon Skeet on types and memory
- [Stephen Cleary: Async and Await](https://blog.stephencleary.com/2012/02/async-and-await.html) — The definitive async explainer

### Videos
- [Nick Chapsas: Records Deep Dive](https://www.youtube.com/watch?v=9Byvwa9yF-I)
- [Nick Chapsas: Stop Using Async Void](https://www.youtube.com/watch?v=zhCRX3B7qwY)

### Documentation
- [Microsoft: Value Types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-types)
- [Microsoft: Nullable Reference Types](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)
- [Microsoft: LINQ Overview](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)

### Practice
- [LINQPad](https://www.linqpad.net/) — Essential tool for experimenting with LINQ
- [BenchmarkDotNet](https://benchmarkdotnet.org/) — For the allocation benchmarks

---

## Reflection Questions

1. Why would you choose `record struct` over `record class` for `Money`?
2. What happens if you call `.ToList()` before `.Where()` on an EF query?
3. Why does blocking on async (`.Result`) cause problems under load but work fine in simple tests?
4. How does nullable reference types change how you design public APIs?

---

**Next:** [Part 2 — Records, Exceptions, Expression Trees, and Memory](./part-2.md)
