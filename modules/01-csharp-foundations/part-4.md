# 01 — Deep C# & .NET Foundations

## Part 4 / 4 — Async Streams, DI Deep Dive, and Mastery Exercises

---

## What You'll Learn

This final part covers advanced topics that round out your C# mastery:
- Span/Memory for high-performance scenarios
- Async streams for processing large datasets
- Dependency injection internals
- Practical mastery exercises

---

## Core Concept #15 — Span<T> and Memory<T>

### Why This Matters

Span and Memory let you work with slices of data without allocating new arrays. They're essential for:
- Parsing strings efficiently
- Processing large files
- Building high-performance APIs
- Reducing GC pressure in hot paths

### What Goes Wrong Without This

**Real scenario:** A CSV parser creates a new string for every field in every row. Processing a 1GB file allocates 10GB+ of strings. GC pauses cause request timeouts.

With Spans: parse the entire file without allocating any intermediate strings.

### The Basics

```csharp
// Span: A view into contiguous memory (stack-only)
Span<int> numbers = stackalloc int[10];
numbers[0] = 42;

// ReadOnlySpan: Immutable view
ReadOnlySpan<char> slice = "Hello, World!".AsSpan()[0..5]; // "Hello"
// No allocation! Just a pointer + length

// Slicing without allocation
string text = "name=value";
ReadOnlySpan<char> name = text.AsSpan()[..4];   // "name"
ReadOnlySpan<char> value = text.AsSpan()[5..];  // "value"
```

### Practical Example: Parsing

```csharp
// BAD: Allocates strings for each part
public (string Key, string Value) ParseBad(string line)
{
    var parts = line.Split('='); // Allocates array + strings
    return (parts[0], parts[1]);
}

// GOOD: Zero allocations
public (ReadOnlySpan<char> Key, ReadOnlySpan<char> Value) ParseGood(ReadOnlySpan<char> line)
{
    var index = line.IndexOf('=');
    return (line[..index], line[(index + 1)..]);
}

// Even better for repeated use
public static bool TryParse(ReadOnlySpan<char> line, out ReadOnlySpan<char> key, out ReadOnlySpan<char> value)
{
    var index = line.IndexOf('=');
    if (index < 0)
    {
        key = default;
        value = default;
        return false;
    }
    key = line[..index];
    value = line[(index + 1)..];
    return true;
}
```

### Memory<T> for Async

Span is stack-only — it can't cross await boundaries. Use Memory for async:

```csharp
// This won't compile — Span can't be used across await
async Task ProcessBad(Span<byte> data)
{
    await Task.Delay(100);
    Process(data); // Error: can't use Span here
}

// Memory works across await
async Task ProcessGood(Memory<byte> data)
{
    await Task.Delay(100);
    Process(data.Span); // Convert to Span when needed
}
```

---

## Core Concept #16 — Async Streams (IAsyncEnumerable)

### Why This Matters

Traditional async returns one result. Async streams return multiple results over time. Perfect for:
- Streaming database results
- Processing files line by line
- Real-time data feeds
- Paginated API responses

### What Goes Wrong Without This

**Real scenario:** An API loads 100,000 records into a List, then returns them. Memory spikes, response times balloon, and users wait for everything before seeing anything.

With async streams: start returning results immediately while continuing to fetch.

### Basic Usage

```csharp
// Producer: yields items asynchronously
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var batch in _context.Orders.AsAsyncEnumerable().WithCancellation(ct))
    {
        yield return batch;
    }
}

// Consumer: processes items as they arrive
await foreach (var order in StreamOrdersAsync(cancellationToken))
{
    Console.WriteLine(order.Id);
}
```

### Real-World Example: File Processing

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);

    while (!reader.EndOfStream)
    {
        ct.ThrowIfCancellationRequested();
        var line = await reader.ReadLineAsync();
        if (line is not null)
            yield return line;
    }
}

// Usage
await foreach (var line in ReadLinesAsync("huge-file.csv", ct))
{
    Process(line);
    // Memory stays flat even for multi-GB files
}
```

### Combining with LINQ

```csharp
// System.Linq.Async package adds LINQ for IAsyncEnumerable
await foreach (var order in StreamOrdersAsync()
    .Where(o => o.Status == OrderStatus.Pending)
    .Take(100))
{
    await ProcessAsync(order);
}
```

---

## Core Concept #17 — Dependency Injection Deep Dive

### Why This Matters

DI isn't just "put interfaces everywhere." Understanding lifetimes, scopes, and registration patterns prevents:
- Memory leaks from wrong lifetimes
- DbContext sharing across requests
- Captive dependencies
- Hard-to-debug runtime errors

### What Goes Wrong Without This

**Real scenario (captive dependency):** A Singleton service takes a Scoped service (DbContext) as a constructor parameter. The DbContext is created once and reused for all requests. After 100 requests, the DbContext tracking explodes and performance tanks.

### Lifetimes Explained

| Lifetime | When to Use | Danger |
|----------|-------------|--------|
| **Singleton** | Truly stateless services, caches, configuration | Using scoped dependencies |
| **Scoped** | Per-request services, DbContext, unit of work | Long-lived consumers |
| **Transient** | Lightweight, stateless operations | Using as if it were scoped |

```csharp
// GOOD: Appropriate lifetimes
services.AddDbContext<AppDbContext>(); // Scoped by default
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IOrderService, OrderService>();
services.AddSingleton<IConfigurationService, ConfigurationService>();
services.AddTransient<IDateTimeProvider, DateTimeProvider>();

// BAD: Captive dependency
services.AddSingleton<IBadService, BadService>(); // Singleton...
public class BadService
{
    private readonly AppDbContext _context; // ...holds a Scoped service!
    public BadService(AppDbContext context) => _context = context;
}
```

### Advanced Registration Patterns

```csharp
// Factory registration for complex construction
services.AddScoped<IOrderService>(sp =>
{
    var context = sp.GetRequiredService<AppDbContext>();
    var logger = sp.GetRequiredService<ILogger<OrderService>>();
    var config = sp.GetRequiredService<IOptions<OrderSettings>>();
    return new OrderService(context, logger, config.Value);
});

// Open generics
services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
// IRepository<Order> resolves to Repository<Order>
// IRepository<Product> resolves to Repository<Product>

// Multiple implementations
services.AddScoped<INotificationSender, EmailSender>();
services.AddScoped<INotificationSender, SmsSender>();
services.AddScoped<INotificationSender, PushSender>();

// Resolve all implementations
public class NotificationService
{
    private readonly IEnumerable<INotificationSender> _senders;

    public NotificationService(IEnumerable<INotificationSender> senders)
    {
        _senders = senders;
    }

    public async Task NotifyAllAsync(Notification notification)
    {
        foreach (var sender in _senders)
        {
            await sender.SendAsync(notification);
        }
    }
}

// Keyed services (.NET 8+)
services.AddKeyedScoped<IPaymentProcessor, StripeProcessor>("stripe");
services.AddKeyedScoped<IPaymentProcessor, PayPalProcessor>("paypal");

public class CheckoutService
{
    public CheckoutService([FromKeyedServices("stripe")] IPaymentProcessor processor)
    {
    }
}
```

### Decorator Pattern with DI

```csharp
// Using Scrutor library
services.AddScoped<IOrderRepository, OrderRepository>();
services.Decorate<IOrderRepository, CachingOrderRepository>();
services.Decorate<IOrderRepository, LoggingOrderRepository>();

// Execution order: LoggingOrderRepository → CachingOrderRepository → OrderRepository
```

---

## Core Concept #18 — Coding Standards and Style

### Why This Matters

Consistent code is:
- Faster to read
- Easier to review
- Less error-prone
- More maintainable

### The Non-Negotiables

```csharp
// Naming conventions
public class OrderService { }                    // PascalCase for types
public void ProcessOrder() { }                   // PascalCase for methods
private readonly ILogger _logger;                // _camelCase for private fields
public string CustomerName { get; set; }         // PascalCase for properties
var orderCount = 5;                              // camelCase for locals
public const int MaxRetries = 3;                 // PascalCase for constants

// Async suffix
public async Task ProcessAsync() { }             // Always use Async suffix
public async Task<Order> GetOrderAsync() { }

// Interface prefix
public interface IOrderService { }               // Always prefix with I

// Avoid abbreviations
public class CustomerManager { }                 // NOT: CustMgr
public string EmailAddress { get; set; }         // NOT: EmailAddr
```

### EditorConfig Setup

```ini
# .editorconfig
root = true

[*.cs]
indent_style = space
indent_size = 4
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

# Naming rules
dotnet_naming_rule.private_fields_should_be_camel_case.severity = warning
dotnet_naming_rule.private_fields_should_be_camel_case.symbols = private_fields
dotnet_naming_rule.private_fields_should_be_camel_case.style = camel_case_underscore

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.camel_case_underscore.capitalization = camel_case
dotnet_naming_style.camel_case_underscore.required_prefix = _

# Code style
csharp_style_var_for_built_in_types = true:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_prefer_simple_using_statement = true:suggestion
csharp_style_expression_bodied_methods = when_on_single_line:suggestion
```

---

## Mastery Exercises

These exercises prove you understand the concepts, not just the syntax.

### Exercise 1: High-Performance Parser

Build a CSV parser that:
- Uses `Span<T>` for zero-allocation parsing
- Streams results via `IAsyncEnumerable`
- Handles quoted fields and escapes
- Processes a 1GB file with <50MB memory

```csharp
public interface ICsvParser
{
    IAsyncEnumerable<T> ParseAsync<T>(string path, CancellationToken ct = default)
        where T : new();
}

// Test: Process orders.csv (1M rows) and verify memory stays flat
```

### Exercise 2: Plugin System

Build a plugin loader that:
- Discovers plugins via reflection
- Loads them with correct DI lifetimes
- Executes them in a pipeline

```csharp
public interface IPlugin
{
    int Order { get; }
    Task ExecuteAsync(PluginContext context);
}

[Plugin(Order = 1)]
public class ValidationPlugin : IPlugin { }

[Plugin(Order = 2)]
public class EnrichmentPlugin : IPlugin { }

// Test: Plugins execute in order, failures are handled gracefully
```

### Exercise 3: Generic Pipeline

Build a generic request pipeline (simplified MediatR):

```csharp
public interface IRequest<TResponse> { }

public interface IRequestHandler<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, CancellationToken ct);
}

public interface IPipelineBehavior<TRequest, TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, RequestHandlerDelegate<TResponse> next);
}

// Test: Behaviors execute in order, wrapping the handler
```

### Exercise 4: OrderFlow Integration

Integrate all concepts into OrderFlow:
- Async streaming for order export
- Span-based SKU parsing
- Proper DI lifetimes for all services
- Decorator for repository caching/logging

---

## Deliverables

1. **CSV Parser** with benchmarks showing zero-allocation parsing
2. **Plugin system** with reflection-based discovery
3. **Request pipeline** with generic handlers and behaviors
4. **OrderFlow updates** with all concepts integrated
5. **Code review checklist** documenting your team's standards

---

## Resources

### Must-Read
- [Stephen Toub: How Async/Await Really Works](https://devblogs.microsoft.com/dotnet/how-async-await-really-works-in-csharp/) — Deep dive
- [Adam Sitnik: Span<T>](https://adamsitnik.com/Span/) — Performance implications

### Videos
- [Nick Chapsas: IAsyncEnumerable Deep Dive](https://www.youtube.com/watch?v=3dCWgzNnM6s)
- [Nick Chapsas: Dependency Injection Deep Dive](https://www.youtube.com/watch?v=YR6HkvNBpX4)
- [Raw Coding: DI Container Internals](https://www.youtube.com/watch?v=8JIp8S6G2Es)

### Documentation
- [Microsoft: Span<T>](https://learn.microsoft.com/en-us/dotnet/api/system.span-1)
- [Microsoft: IAsyncEnumerable](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/generate-consume-asynchronous-stream)
- [Microsoft: DI Guidelines](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines)

### Tools
- [BenchmarkDotNet](https://benchmarkdotnet.org/) — Measure allocations
- [Scrutor](https://github.com/khellang/Scrutor) — DI decorators and scanning

---

## Module Summary

After completing Module 01, you can:

| Skill | Evidence |
|-------|----------|
| Write correct C# | Domain models with proper types, nullability, immutability |
| Design exceptions | Custom exception hierarchy with context |
| Use LINQ properly | Expression-based queries that translate to SQL |
| Understand async | No blocking, proper cancellation, async streams |
| Apply SOLID | Services with single responsibilities, proper DI |
| Use patterns | Strategy, Decorator, Factory where appropriate |
| Optimize when needed | Span-based parsing, understanding of allocations |

This foundation makes everything else possible. Move to Module 02 when your OrderFlow domain models are complete and tested.

---

## Reflection Questions

1. When would you choose `Memory<T>` over `Span<T>`?
2. What happens if a Singleton service depends on a Scoped service?
3. How would you test a service that uses `IAsyncEnumerable`?
4. When is reflection appropriate in application code vs library code?

---

**Next:** [Module 02 — REST Fundamentals](../02-rest-apis/part-1.md)
