# 01 — Deep C# & .NET Foundations

## Part 2 / 4 — Exceptions, Expression Trees, Memory, and Delegates

---

## What You'll Learn

Part 1 covered the fundamentals. Now we go deeper:
- How to design exceptions properly (not just throw them)
- Expression trees and why EF Core uses them
- Memory management and GC awareness
- Delegates, Func, Action, and closures

These concepts separate developers who *use* C# from developers who *understand* it.

---

## Core Concept #6 — Exception Design

### Why This Matters

Exceptions aren't just for errors. They're communication tools. Well-designed exceptions:
- Tell you exactly what went wrong
- Include enough context to debug
- Don't hide the real problem
- Don't crash silently

### What Goes Wrong Without This

**Real scenario:** A developer wraps everything in `try/catch (Exception) { return null; }`. The API returns null for completely different failure modes — database down, invalid input, business rule violation. Debugging is impossible because all failures look the same.

**Real scenario:** Exception messages say "An error occurred" with no context. Support spends hours reproducing issues that should be obvious from the error.

### Exception Design Principles

```csharp
// GOOD: Domain-specific exceptions with context
public class InsufficientStockException : Exception
{
    public string Sku { get; }
    public int Requested { get; }
    public int Available { get; }

    public InsufficientStockException(string sku, int requested, int available)
        : base($"Insufficient stock for {sku}: requested {requested}, available {available}")
    {
        Sku = sku;
        Requested = requested;
        Available = available;
    }
}

// Usage
throw new InsufficientStockException(product.Sku, quantity, product.StockLevel);

// Now the catch site has everything it needs
catch (InsufficientStockException ex)
{
    _logger.LogWarning("Stock issue: {Sku}, need {Requested}, have {Available}",
        ex.Sku, ex.Requested, ex.Available);
    return Problem($"Not enough stock for {ex.Sku}");
}
```

### Anti-Patterns

```csharp
// BAD: Catching everything and hiding it
try
{
    await _service.ProcessOrderAsync(order);
}
catch (Exception)
{
    return false; // What happened? Nobody knows.
}

// BAD: Losing the stack trace
catch (Exception ex)
{
    throw ex; // Stack trace is now from HERE, not the original error
}

// GOOD: Preserve the stack trace
catch (Exception ex)
{
    throw; // Rethrows with original stack trace
}

// GOOD: Wrap with context but preserve inner
catch (SqlException ex)
{
    throw new DataAccessException("Failed to save order", ex);
}

// BAD: Using exceptions for flow control
try
{
    var user = _repo.GetById(id);
    return user;
}
catch (NotFoundException)
{
    return null; // Exceptions are expensive for expected cases
}

// GOOD: Design APIs to avoid exceptions for expected cases
var user = _repo.GetByIdOrDefault(id); // Returns null if not found
if (user is null) return NotFound();
```

### When to Throw vs Return

| Situation | Approach |
|-----------|----------|
| Programming error (null argument) | Throw `ArgumentNullException` |
| Invalid state (can't do this now) | Throw domain exception |
| Business rule violation | Depends — often return a Result type |
| Item not found | Return null or use Result/Option pattern |
| External system failure | Throw and let it bubble up |

---

## Core Concept #7 — Expression Trees

### Why This Matters

Expression trees are how LINQ queries become SQL. They're the reason `Where(p => p.Price > 100)` becomes `WHERE Price > 100` instead of loading everything into memory.

Understanding this explains:
- Why some LINQ methods work with EF and others don't
- Why EF throws "cannot translate" errors
- How specification patterns work
- How dynamic queries are built

### What Goes Wrong Without This

**Real scenario:** A developer adds a custom method to a LINQ query:

```csharp
.Where(p => MyCustomMethod(p.Name)) // EF can't translate this
```

EF either throws a confusing error or silently evaluates in memory, loading millions of rows.

### How Expression Trees Work

```csharp
// This lambda is a DELEGATE (compiled code)
Func<Product, bool> predicate = p => p.Price > 100;

// This lambda is an EXPRESSION TREE (data structure describing the code)
Expression<Func<Product, bool>> expression = p => p.Price > 100;
```

EF Core receives the expression tree and translates it:

```csharp
// This expression tree...
Expression<Func<Product, bool>> expr = p => p.Price > 100;

// ...becomes this SQL
// WHERE [p].[Price] > 100
```

### What Can Be Translated

| Works | Doesn't Work |
|-------|--------------|
| Property access (`p.Name`) | Custom methods (`MyMethod(p.Name)`) |
| Operators (`>`, `==`, `&&`) | Complex C# logic |
| `String.Contains`, `StartsWith` | Most string methods |
| `Any()`, `All()`, `Count()` | External function calls |
| EF.Functions methods | LINQ to Objects methods |

### Building Dynamic Queries

```csharp
public static class ProductSpecifications
{
    public static Expression<Func<Product, bool>> InCategory(Guid categoryId) =>
        p => p.CategoryId == categoryId;

    public static Expression<Func<Product, bool>> PriceRange(decimal min, decimal max) =>
        p => p.Price >= min && p.Price <= max;

    public static Expression<Func<Product, bool>> Active() =>
        p => p.IsActive;
}

// Usage
var query = _context.Products
    .Where(ProductSpecifications.InCategory(categoryId))
    .Where(ProductSpecifications.Active());
```

---

## Core Concept #8 — Memory Management

### Why This Matters

You don't need to be a GC expert. But understanding the cost model helps you:
- Avoid performance pitfalls
- Make informed tradeoffs
- Debug memory issues
- Write efficient hot paths

### What Goes Wrong Without This

**Real scenario:** A developer builds strings in a loop with `+=`:

```csharp
string result = "";
foreach (var item in items) // 10,000 items
{
    result += item.Name + ", "; // Creates new string EVERY iteration
}
```

This creates 10,000 intermediate strings. GC spikes, API slows down.

### Key Concepts

**Stack vs Heap:**
- Stack: Fast, automatic cleanup, limited size, value types
- Heap: Managed by GC, unlimited size, reference types

**Allocations:**
- Every `new` for a class allocates heap memory
- Strings are immutable — concatenation creates new objects
- LINQ methods often allocate iterators
- Closures allocate objects to capture variables

### Avoiding Unnecessary Allocations

```csharp
// BAD: String concatenation in loops
string result = "";
foreach (var item in items)
{
    result += item.Name; // New string every time
}

// GOOD: StringBuilder for building strings
var sb = new StringBuilder();
foreach (var item in items)
{
    sb.Append(item.Name);
}
var result = sb.ToString();

// BAD: Creating arrays in hot paths
public bool IsValid(string input)
{
    var chars = input.ToCharArray(); // Allocates every call
    return chars.All(char.IsLetter);
}

// GOOD: Work with spans to avoid allocation
public bool IsValid(ReadOnlySpan<char> input)
{
    foreach (var c in input)
    {
        if (!char.IsLetter(c)) return false;
    }
    return true;
}

// BAD: Boxing value types
var list = new ArrayList();
list.Add(42); // Boxing: int → object

// GOOD: Generic collections
var list = new List<int>();
list.Add(42); // No boxing
```

### When to Care About Allocations

| Context | Care Level |
|---------|------------|
| Request handler (runs once per request) | Low — allocations are fine |
| Tight loop processing many items | Medium — avoid obvious waste |
| Hot path (thousands of calls/second) | High — profile and optimize |
| Background job (runs occasionally) | Low — clarity over micro-optimization |

**Rule of thumb:** Don't micro-optimize until you've measured. But understand the cost model so you don't create obvious problems.

---

## Core Concept #9 — Delegates, Func, Action

### Why This Matters

Delegates are the foundation of:
- LINQ (`Where`, `Select`, etc.)
- Event handling
- Callbacks and middleware
- Dependency injection factories

### The Basics

```csharp
// Action: No return value
Action<string> log = message => Console.WriteLine(message);
log("Hello"); // Prints "Hello"

// Func: Has return value
Func<int, int, int> add = (a, b) => a + b;
var result = add(2, 3); // 5

// Predicate: Returns bool (common in filtering)
Predicate<Product> isExpensive = p => p.Price > 100;
```

### How LINQ Uses Delegates

```csharp
// Where takes a Func<T, bool>
products.Where(p => p.Price > 100);
//              ↑ This is a Func<Product, bool>

// Select takes a Func<T, TResult>
products.Select(p => p.Name);
//              ↑ This is a Func<Product, string>
```

### Events

```csharp
public class Order
{
    public event EventHandler<OrderPlacedEventArgs>? Placed;

    public void Place()
    {
        // ... place order logic
        Placed?.Invoke(this, new OrderPlacedEventArgs(Id, Total));
    }
}

// Subscribing
order.Placed += (sender, args) =>
{
    _logger.LogInformation("Order {Id} placed for {Total}", args.Id, args.Total);
};
```

---

## Core Concept #10 — Closures and Captured Variables

### Why This Matters

Closures are powerful but dangerous. They:
- Can extend variable lifetime unexpectedly
- Can cause memory leaks
- Can behave differently than expected in loops
- Are created automatically (you might not notice)

### What Goes Wrong Without This

**Real scenario:** A developer creates handlers in a loop:

```csharp
for (int i = 0; i < buttons.Length; i++)
{
    buttons[i].Click += () => Console.WriteLine($"Button {i}");
}
// Every button prints "Button 5" because they all capture the SAME 'i'
```

### How Closures Work

```csharp
int multiplier = 10;
Func<int, int> multiply = x => x * multiplier; // Captures 'multiplier'

Console.WriteLine(multiply(5)); // 50

multiplier = 20; // Change the captured variable
Console.WriteLine(multiply(5)); // 100! The closure sees the change
```

The compiler creates a hidden class:

```csharp
// What the compiler generates (simplified)
class ClosureClass
{
    public int multiplier;
    public int Invoke(int x) => x * multiplier;
}
```

### The Classic Loop Bug

```csharp
// BAD: All closures capture same variable
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var action in actions)
{
    action(); // Prints 5, 5, 5, 5, 5
}

// GOOD: Capture a copy
for (int i = 0; i < 5; i++)
{
    int copy = i; // New variable each iteration
    actions.Add(() => Console.WriteLine(copy));
}
// Prints 0, 1, 2, 3, 4
```

### Closure Memory Leaks

```csharp
// Potential leak: closure keeps expensive object alive
public IEnumerable<string> GetNames(LargeDataSet data)
{
    return _items.Select(item => ProcessWith(item, data));
    // The closure captures 'data', keeping it alive until the IEnumerable is disposed
}
```

---

## Hands-On: OrderFlow Exceptions and Events

### Custom Exceptions

```csharp
public abstract class OrderFlowException : Exception
{
    protected OrderFlowException(string message) : base(message) { }
    protected OrderFlowException(string message, Exception inner) : base(message, inner) { }
}

public class InsufficientStockException : OrderFlowException
{
    public string Sku { get; }
    public int Requested { get; }
    public int Available { get; }

    public InsufficientStockException(string sku, int requested, int available)
        : base($"Insufficient stock for {sku}: requested {requested}, available {available}")
    {
        Sku = sku;
        Requested = requested;
        Available = available;
    }
}

public class CurrencyMismatchException : OrderFlowException
{
    public string Expected { get; }
    public string Actual { get; }

    public CurrencyMismatchException(string expected, string actual)
        : base($"Currency mismatch: expected {expected}, got {actual}")
    {
        Expected = expected;
        Actual = actual;
    }
}
```

### Domain Events

```csharp
public interface IDomainEvent
{
    DateTime OccurredAt { get; }
}

public record OrderPlacedEvent(
    Guid OrderId,
    Guid CustomerId,
    Money Total,
    DateTime OccurredAt) : IDomainEvent;

public record StockReservedEvent(
    Guid ProductId,
    string Sku,
    int Quantity,
    DateTime OccurredAt) : IDomainEvent;

// In Order aggregate
public class Order
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void Place()
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Can only place draft orders");

        Status = OrderStatus.Placed;
        _domainEvents.Add(new OrderPlacedEvent(Id, CustomerId, Total, DateTime.UtcNow));
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

---

## Deliverables

1. **Exception hierarchy** for OrderFlow:
   - Base `OrderFlowException`
   - `InsufficientStockException`
   - `CurrencyMismatchException`
   - `OrderStateException`

2. **Domain events**:
   - `OrderPlacedEvent`
   - `OrderCancelledEvent`
   - `StockReservedEvent`
   - `StockReleasedEvent`

3. **Specification classes** for Product queries:
   - `ProductSpecifications.InCategory()`
   - `ProductSpecifications.PriceRange()`
   - `ProductSpecifications.InStock()`

4. **Unit tests** proving:
   - Exceptions contain correct context
   - Domain events are raised correctly
   - Specifications compose correctly

---

## Resources

### Must-Read
- [Stephen Toub: Understanding ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/) — Deep dive on async memory
- [Pro .NET Memory Management, Chapter 3-5](https://prodotnetmemory.com/) — GC internals

### Videos
- [Nick Chapsas: Expression Trees](https://www.youtube.com/watch?v=OEHV_kPGPww)
- [Zoran Horvat: Designing Exceptions](https://www.youtube.com/watch?v=PK3B4RkhOm4)

### Documentation
- [Microsoft: Expression Trees](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/expression-trees/)
- [Microsoft: Garbage Collection](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/)
- [Microsoft: Delegates](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/)

### Tools
- [BenchmarkDotNet](https://benchmarkdotnet.org/) — Measure allocations
- [dotnet-trace](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) — Profile GC

---

## Reflection Questions

1. When should you catch exceptions vs let them propagate?
2. Why does `throw ex;` lose the stack trace while `throw;` preserves it?
3. How would you debug a memory leak caused by a closure?
4. Why do expression trees exist instead of just using delegates everywhere?

---

**Next:** [Part 3 — Generics, SOLID, Design Patterns, and Reflection](./part-3.md)
