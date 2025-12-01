# 13 — Working with Legacy Code & Debugging

## Part 2 / 3 — Debugging Systematically

---

## Why This Skill Matters

"It works on my machine" isn't debugging. Neither is randomly adding `Console.WriteLine` until something appears.

Systematic debugging means:
- Reproducing issues reliably
- Narrowing down causes methodically
- Finding root causes, not symptoms
- Fixing without introducing new bugs

Senior developers aren't smarter — they have better debugging process.

---

## What Goes Wrong Without This

**Real scenario:** A developer spends 3 days on a bug. They try random fixes, each breaking something else. Finally, a senior developer asks "can you reproduce it?" The developer can't. The senior finds the reproduction in 30 minutes and the fix in 10.

**Real scenario:** Production is down. A developer reverts the last deployment. It doesn't help. They keep reverting. Still broken. Turns out the issue was a database migration from 3 deployments ago. 4 hours of downtime because they didn't check logs systematically.

---

## Core Skill #1 — Reading Stack Traces

### Anatomy of a Stack Trace

```
System.NullReferenceException: Object reference not set to an instance of an object.
   at OrderFlow.Services.OrderService.CalculateTotal(Order order) in /src/Services/OrderService.cs:line 45
   at OrderFlow.Services.OrderService.PlaceOrderAsync(Order order) in /src/Services/OrderService.cs:line 32
   at OrderFlow.Api.Endpoints.OrderEndpoints.CreateOrder(CreateOrderRequest request) in /src/Api/Endpoints/OrderEndpoints.cs:line 28
   at lambda_method123(Closure, Object, Object[])
   at Microsoft.AspNetCore.Routing.EndpointMiddleware.<Invoke>g__AwaitRequestTask|6_0(...)
```

**Read bottom to top** for flow, **top to bottom** for cause:
- Line 45 in `CalculateTotal` is where it crashed
- It was called from line 32 in `PlaceOrderAsync`
- Which was called from line 28 in `CreateOrder`

### Common Stack Trace Patterns

**NullReferenceException:**
```csharp
// Stack trace points here
var total = order.Items.Sum(i => i.Price);  // Line 45

// Possible nulls:
// - order is null
// - order.Items is null
// - An item in Items is null
// - i.Price is null (if nullable)
```

**InvalidOperationException in LINQ:**
```
System.InvalidOperationException: Sequence contains no elements
   at System.Linq.ThrowHelper.ThrowNoElementsException()
   at System.Linq.Enumerable.First[TSource](IEnumerable`1 source)
   at OrderFlow.Services.OrderService.GetPrimaryItem(Order order)
```

This means `.First()` was called on an empty collection. Use `.FirstOrDefault()` or check `.Any()` first.

**Async Stack Traces:**
```
System.Exception: Something went wrong
   at OrderFlow.Services.OrderService.ProcessAsync() in OrderService.cs:line 50
--- End of stack trace from previous location ---
   at OrderFlow.Api.Endpoints.OrderEndpoints.CreateOrder()
```

The `--- End of stack trace from previous location ---` indicates an await boundary.

---

## Core Skill #2 — Using the Debugger Effectively

### Beyond F5 and F10

**Conditional Breakpoints:**
```
Right-click breakpoint → Conditions → Expression:
order.CustomerId == Guid.Parse("abc...")
```
Stops only when the condition is true. Essential for bugs that only happen with specific data.

**Tracepoints (Log without stopping):**
```
Right-click breakpoint → Actions → Log message:
"Processing order {order.Id} with {order.Items.Count} items"
```
Logs without stopping execution. Useful for understanding flow.

**Data Breakpoints (.NET 5+):**
Break when a specific field changes value. Find unexpected mutations.

**Exception Settings:**
```
Debug → Windows → Exception Settings
Check "Common Language Runtime Exceptions"
```
Break when ANY exception is thrown, not just unhandled ones.

### The Scientific Method

1. **Observe:** What exactly is the symptom?
2. **Hypothesize:** What could cause this?
3. **Predict:** If my hypothesis is right, what should I see when...?
4. **Test:** Check the prediction
5. **Iterate:** Refine hypothesis based on results

```
Observation: Orders sometimes have wrong totals
Hypothesis: Tax calculation uses wrong rate
Prediction: If I breakpoint the tax calculation, I'll see the wrong rate
Test: Set breakpoint, step through
Result: Rate is correct. Back to step 2.

New Hypothesis: Items are being counted twice
Prediction: I'll see the same item processed multiple times
Test: Set breakpoint on item loop
Result: Confirmed! Back button resubmits form. Found it.
```

---

## Core Skill #3 — Log Analysis

### Structured Logging

```csharp
// BAD: Unstructured
_logger.LogInformation($"Processing order {order.Id} for customer {order.CustomerId}");

// GOOD: Structured (searchable, parseable)
_logger.LogInformation("Processing order {OrderId} for customer {CustomerId}",
    order.Id, order.CustomerId);
```

Structured logs let you query:
```sql
-- Find all logs for a specific order
SELECT * FROM logs WHERE OrderId = 'abc-123'

-- Find slow operations
SELECT * FROM logs WHERE Duration > 1000
```

### Correlation IDs

```csharp
// Every request gets a correlation ID
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
        ?? Guid.NewGuid().ToString();

    using (_logger.BeginScope(new Dictionary<string, object>
    {
        ["CorrelationId"] = correlationId
    }))
    {
        context.Response.Headers["X-Correlation-ID"] = correlationId;
        await next();
    }
});
```

Now you can trace a request across services:
```
[CorrelationId=abc-123] API: Received order request
[CorrelationId=abc-123] OrderService: Validating order
[CorrelationId=abc-123] InventoryService: Checking stock
[CorrelationId=abc-123] PaymentService: Processing payment
[CorrelationId=abc-123] API: Returning success
```

### Log Levels

| Level | When to Use |
|-------|-------------|
| **Trace** | Detailed debugging (method entry/exit) |
| **Debug** | Diagnostic information for developers |
| **Information** | Normal operations worth recording |
| **Warning** | Something unexpected but handled |
| **Error** | Failures that affect functionality |
| **Critical** | System-wide failures |

Production typically shows Information and above. Enable Debug/Trace for specific investigations.

---

## Core Skill #4 — Production Debugging

### When You Can't Attach a Debugger

**Application Insights / Seq queries:**
```
// Find errors in the last hour
traces
| where timestamp > ago(1h)
| where severityLevel >= 3
| order by timestamp desc

// Find slow requests
requests
| where duration > 1000
| summarize count() by name
```

**dotnet-trace for CPU profiling:**
```bash
# Find the process ID
dotnet-trace ps

# Collect a trace
dotnet-trace collect -p <PID> --duration 00:00:30

# Analyze with PerfView or speedscope.app
```

**dotnet-dump for memory analysis:**
```bash
# Create a dump
dotnet-dump collect -p <PID>

# Analyze
dotnet-dump analyze <dump-file>

# Find large objects
> dumpheap -stat
> dumpheap -type Order
```

### Common Production Issues

**Memory Leak Pattern:**
```
1. Monitor memory over time
2. If it grows without bound → leak
3. Take dump when high
4. Compare object counts to earlier dump
5. Find what's accumulating
```

**Thread Pool Starvation Pattern:**
```
1. Requests start timing out
2. CPU is low (threads are waiting, not working)
3. Check for sync-over-async (.Result, .Wait())
4. Check for blocking calls in async methods
```

**Database Connection Exhaustion:**
```
1. Intermittent connection failures
2. "Timeout expired" errors
3. Check for undisposed DbContexts
4. Check for connections not returned to pool
```

---

## Core Skill #5 — Binary Search Debugging

When you have no idea where the bug is:

### Git Bisect

```bash
# Start bisect
git bisect start

# Mark current commit as bad
git bisect bad

# Mark a known good commit
git bisect good <commit-hash>

# Git checks out middle commit
# Test it, then mark:
git bisect good  # or
git bisect bad

# Repeat until Git finds the commit that introduced the bug
```

### Code Bisect

When bisecting code instead of commits:

```csharp
// Original code with bug somewhere
public async Task ProcessOrderAsync(Order order)
{
    ValidateOrder(order);           // 1
    await ReserveStock(order);      // 2
    await ProcessPayment(order);    // 3
    await SendConfirmation(order);  // 4
    await UpdateAnalytics(order);   // 5
}

// Bisect: Comment out half
public async Task ProcessOrderAsync(Order order)
{
    ValidateOrder(order);           // 1
    await ReserveStock(order);      // 2
    // await ProcessPayment(order); // 3
    // await SendConfirmation(order); // 4
    // await UpdateAnalytics(order);  // 5
}
// If bug disappears → it's in 3, 4, or 5
// If bug persists → it's in 1 or 2
```

---

## Hands-On: Debugging Challenges

### Challenge 1: The Intermittent Failure

Add this bug to OrderFlow:
```csharp
public async Task<Order> GetOrderAsync(Guid id)
{
    // Bug: Race condition
    if (_cache.TryGet(id, out var order))
        return order;

    order = await _repository.GetAsync(id);
    _cache.Set(id, order);
    return order;  // Can be null if not found
}
```

Debug it:
1. Write a test that reproduces it
2. Find the race condition
3. Fix it properly

### Challenge 2: The Memory Leak

Add this bug:
```csharp
public class OrderEventHandler
{
    private static readonly List<Order> _processedOrders = new();

    public async Task HandleAsync(OrderPlacedEvent e)
    {
        var order = await _repository.GetAsync(e.OrderId);
        _processedOrders.Add(order);  // Never cleared
        await ProcessAsync(order);
    }
}
```

Debug it:
1. Use dotnet-dump to find the leak
2. Identify the accumulating objects
3. Fix it

### Challenge 3: The Slow Endpoint

Add this bug:
```csharp
public async Task<List<OrderSummary>> GetOrdersAsync(Guid customerId)
{
    var orders = await _context.Orders
        .Where(o => o.CustomerId == customerId)
        .ToListAsync();

    // N+1 query bug
    foreach (var order in orders)
    {
        order.CustomerName = (await _context.Customers.FindAsync(order.CustomerId))?.Name;
    }

    return orders.Select(o => new OrderSummary(o)).ToList();
}
```

Debug it:
1. Profile the endpoint
2. Find the N+1 pattern
3. Fix with proper Include or projection

---

## Deliverables

1. **Debugging playbook** — Your personal process for different bug types
2. **Stack trace analysis** — 5 real stack traces you've analyzed with explanations
3. **Debugging session recording** — Video of you debugging a real issue, narrating your process
4. **Production incident simulation** — Debug the challenges above with logs only (no debugger)

---

## Resources

### Must-Read
- [Debugging Teams](https://www.oreilly.com/library/view/debugging-teams/9781491932049/) — Process, not just tools
- [Why Programs Fail](https://www.elsevier.com/books/why-programs-fail/zeller/978-0-12-374515-6) — Scientific debugging

### Videos
- [Julia Evans: Debugging Zines](https://wizardzines.com/)
- [Bryan Cantrill: Debugging Production Systems](https://www.youtube.com/watch?v=tDacjrSCeq4)

### Tools
- [Seq](https://datalust.co/seq) — Structured log analysis
- [Jaeger](https://www.jaegertracing.io/) — Distributed tracing
- [speedscope](https://www.speedscope.app/) — Trace visualization

---

## Reflection Questions

1. What's your first step when you hear "it's broken"?
2. How do you know when to stop debugging and ask for help?
3. What's the most time you've wasted on a bug that had a simple fix?
4. How do you debug issues that only happen in production?

---

**Next:** [Part 3 — Making Safe Changes](./part-3.md)
