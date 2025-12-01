# Part 1 — Performance Foundations

## Why This Matters

Every performance conversation starts the same way:

> Developer: "This endpoint is slow"
> Lead: "How slow?"
> Developer: "Um... slow?"

Without measurement, you're guessing. And guessing leads to:
- Optimizing code that doesn't matter
- Missing the actual bottleneck
- Wasting time on micro-optimizations
- Never knowing if your "fix" helped

**The #1 performance skill isn't coding — it's measuring.**

---

## What Goes Wrong Without This

### The Optimization That Made Things Worse
```
Complaint: "The order search is slow"
Guess: "Must be the database query"
Action: Add caching layer
Result: 10x more memory usage, same latency
Reality: Slow JSON serialization of 10,000 items
```

### The "Fixed It" That Wasn't
```
Before optimization: 450ms average
Developer: "I optimized the algorithm"
After optimization: 420ms average
Developer: "30ms improvement!"
Statistics: That's within normal variance
Reality: Nothing changed
```

### The Profiler That Revealed the Truth
```
Team assumption: Database is the bottleneck
Profiler showed:
  - Database: 15ms
  - Business logic: 25ms
  - JSON serialization: 180ms
  - HTTP client to external API: 280ms

The "slow database" was 3% of the problem.
```

---

## The Optimization Loop

Never optimize without this cycle:

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  1. MEASURE                                     │
│     └── Establish baseline with profiler        │
│              │                                  │
│              ▼                                  │
│  2. IDENTIFY                                    │
│     └── Find the actual bottleneck              │
│              │                                  │
│              ▼                                  │
│  3. HYPOTHESIZE                                 │
│     └── "If I change X, metric Y improves"      │
│              │                                  │
│              ▼                                  │
│  4. CHANGE                                      │
│     └── Make ONE change                         │
│              │                                  │
│              ▼                                  │
│  5. MEASURE AGAIN                               │
│     └── Verify improvement with same profiler   │
│              │                                  │
│              ▼                                  │
│  6. REPEAT or STOP                              │
│     └── Until target is met                     │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Profiling Tools for .NET

### BenchmarkDotNet: Micro-Benchmarks

For comparing specific code paths:

```csharp
// Install: dotnet add package BenchmarkDotNet

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class SerializationBenchmarks
{
    private readonly Order _order;
    private readonly JsonSerializerOptions _options;

    public SerializationBenchmarks()
    {
        _order = CreateTestOrder();
        _options = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }

    [Benchmark(Baseline = true)]
    public string SystemTextJson()
    {
        return JsonSerializer.Serialize(_order, _options);
    }

    [Benchmark]
    public string SystemTextJsonSourceGen()
    {
        return JsonSerializer.Serialize(_order, OrderJsonContext.Default.Order);
    }

    [Benchmark]
    public string NewtonsoftJson()
    {
        return Newtonsoft.Json.JsonConvert.SerializeObject(_order);
    }
}

// Source generator for System.Text.Json
[JsonSerializable(typeof(Order))]
public partial class OrderJsonContext : JsonSerializerContext { }
```

Run and get results:

```bash
dotnet run -c Release
```

```
|                   Method |      Mean |    Error |   StdDev | Ratio |   Gen0 | Allocated |
|------------------------- |----------:|---------:|---------:|------:|-------:|----------:|
|           SystemTextJson |  2.341 μs | 0.012 μs | 0.011 μs |  1.00 | 0.2365 |   1.94 KB |
| SystemTextJsonSourceGen  |  1.156 μs | 0.008 μs | 0.007 μs |  0.49 | 0.1984 |   1.63 KB |
|          NewtonsoftJson  |  4.892 μs | 0.031 μs | 0.029 μs |  2.09 | 0.5493 |   4.52 KB |
```

**Key insight**: Source-generated JSON is 2x faster with less allocation.

### dotnet-trace: Production-Safe Profiling

Capture traces without significant overhead:

```bash
# Install the tool
dotnet tool install -g dotnet-trace

# List running .NET processes
dotnet-trace ps

# Capture a 30-second trace
dotnet-trace collect -p <PID> --duration 00:00:30

# Or collect specific providers
dotnet-trace collect -p <PID> \
  --providers Microsoft-DotNETCore-SampleProfiler \
  --duration 00:00:30
```

Convert to speedscope for visualization:

```bash
dotnet-trace convert trace.nettrace --format speedscope
# Open in https://speedscope.app
```

### PerfView: Deep GC and Allocation Analysis

PerfView is the gold standard for .NET memory analysis:

```bash
# Download from https://github.com/microsoft/perfview/releases

# Collect CPU and GC data
PerfView.exe collect -GCCollectOnly

# Analyze allocations
PerfView.exe /GCCollectOnly collect
```

Key views in PerfView:
- **GC Stats**: See GC frequency, duration, generations
- **CPU Stacks**: Find what's consuming CPU
- **GC Heap Snapshots**: See what's allocated

---

## Memory Analysis: Finding Allocations

### The Hidden Cost of LINQ

```csharp
// Anti-pattern: Excessive allocations
public class OrderService
{
    public decimal CalculateTotal(IEnumerable<OrderLine> lines)
    {
        // Each LINQ operation allocates
        return lines
            .Where(l => l.IsActive)      // Iterator allocation
            .Select(l => l.Quantity * l.UnitPrice)  // Iterator allocation
            .Sum();                       // Enumeration
    }
}

// Better: Single pass with no allocations
public decimal CalculateTotalOptimized(IEnumerable<OrderLine> lines)
{
    decimal total = 0;
    foreach (var line in lines)
    {
        if (line.IsActive)
        {
            total += line.Quantity * line.UnitPrice;
        }
    }
    return total;
}
```

### Span<T> for Zero-Allocation Parsing

```csharp
// Anti-pattern: String allocations
public class OrderIdParser
{
    public (string Region, int Number) Parse(string orderId)
    {
        // "US-12345" -> ("US", 12345)
        var parts = orderId.Split('-');  // Allocates array
        return (parts[0], int.Parse(parts[1]));
    }
}

// Better: Span-based parsing (zero allocation)
public (string Region, int Number) ParseOptimized(ReadOnlySpan<char> orderId)
{
    var separatorIndex = orderId.IndexOf('-');
    var region = orderId[..separatorIndex];
    var number = int.Parse(orderId[(separatorIndex + 1)..]);
    return (region.ToString(), number);  // Only allocate at the end
}
```

### Object Pooling for Hot Paths

```csharp
// For frequently allocated objects
public class OrderProcessingService
{
    private readonly ObjectPool<StringBuilder> _stringBuilderPool;

    public OrderProcessingService(ObjectPoolProvider poolProvider)
    {
        _stringBuilderPool = poolProvider.CreateStringBuilderPool();
    }

    public string FormatOrderSummary(Order order)
    {
        var sb = _stringBuilderPool.Get();
        try
        {
            sb.Append("Order #").Append(order.Id);
            sb.Append(" - ").Append(order.CustomerName);
            sb.Append(" - $").Append(order.Total.ToString("F2"));

            foreach (var line in order.Lines)
            {
                sb.AppendLine();
                sb.Append("  - ").Append(line.ProductName);
                sb.Append(" x").Append(line.Quantity);
            }

            return sb.ToString();
        }
        finally
        {
            _stringBuilderPool.Return(sb);
        }
    }
}

// Register in DI
services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
```

---

## GC Tuning

### Understanding GC Modes

```csharp
// In your csproj or runtimeconfig.json
// Server GC: Higher throughput, more memory
// Workstation GC: Lower latency, less memory

// For API servers (high throughput)
{
    "runtimeOptions": {
        "configProperties": {
            "System.GC.Server": true,
            "System.GC.Concurrent": true
        }
    }
}

// For containers with memory limits
{
    "runtimeOptions": {
        "configProperties": {
            "System.GC.Server": true,
            "System.GC.HeapHardLimit": 1073741824  // 1GB
        }
    }
}
```

### GC Metrics to Watch

```csharp
public class GcMetricsCollector : BackgroundService
{
    private readonly ILogger<GcMetricsCollector> _logger;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var gen0 = GC.CollectionCount(0);
            var gen1 = GC.CollectionCount(1);
            var gen2 = GC.CollectionCount(2);
            var totalMemory = GC.GetTotalMemory(false);
            var gcInfo = GC.GetGCMemoryInfo();

            _logger.LogInformation(
                "GC: Gen0={Gen0}, Gen1={Gen1}, Gen2={Gen2}, " +
                "Memory={Memory}MB, HeapSize={HeapSize}MB, " +
                "FragmentedBytes={Fragmented}MB",
                gen0, gen1, gen2,
                totalMemory / 1024 / 1024,
                gcInfo.HeapSizeBytes / 1024 / 1024,
                gcInfo.FragmentedBytes / 1024 / 1024);

            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

### Warning Signs in GC

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Gen2 collections/min | < 1 | 1-5 | > 5 |
| % Time in GC | < 5% | 5-15% | > 15% |
| LOH allocations | Rare | Occasional | Frequent |
| Memory growth | Stable | Slow growth | Unbounded |

---

## Defining SLOs That Matter

### What Are SLOs?

- **SLI (Service Level Indicator)**: A metric you measure (e.g., p99 latency)
- **SLO (Service Level Objective)**: Your target (e.g., p99 < 500ms)
- **SLA (Service Level Agreement)**: Contractual commitment (external)

### Setting Realistic SLOs for OrderFlow

```yaml
# orderflow-slos.yaml
service: orderflow-api

slos:
  - name: order-creation-latency
    description: Time to create a new order
    sli:
      type: latency
      percentile: 99
    target: 500ms
    window: 30d
    error_budget: 0.1%  # 99.9% of requests under 500ms

  - name: order-creation-availability
    description: Order creation success rate
    sli:
      type: availability
    target: 99.9%
    window: 30d

  - name: order-search-latency
    description: Time to search orders
    sli:
      type: latency
      percentile: 95
    target: 200ms
    window: 30d
    error_budget: 1%  # 99% of requests under 200ms
```

### Implementing SLO Tracking

```csharp
public class SloMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<SloMiddleware> _logger;
    private static readonly Histogram RequestDuration = Metrics.CreateHistogram(
        "http_request_duration_seconds",
        "Request duration in seconds",
        new HistogramConfiguration
        {
            LabelNames = new[] { "method", "endpoint", "status_code" },
            Buckets = new[] { .01, .025, .05, .1, .25, .5, 1, 2.5, 5 }
        });

    public SloMiddleware(RequestDelegate next, ILogger<SloMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();

            var endpoint = context.GetEndpoint()?.DisplayName ?? "unknown";
            var method = context.Request.Method;
            var statusCode = context.Response.StatusCode.ToString();

            RequestDuration
                .WithLabels(method, endpoint, statusCode)
                .Observe(stopwatch.Elapsed.TotalSeconds);

            // Log slow requests
            if (stopwatch.ElapsedMilliseconds > 500)
            {
                _logger.LogWarning(
                    "Slow request: {Method} {Path} took {Duration}ms",
                    method,
                    context.Request.Path,
                    stopwatch.ElapsedMilliseconds);
            }
        }
    }
}
```

---

## Async Performance Patterns

### Avoiding Sync-over-Async

```csharp
// Anti-pattern: Blocking on async (thread pool starvation)
public class OrderService
{
    public Order GetOrder(int id)
    {
        // NEVER DO THIS
        return GetOrderAsync(id).Result;
    }
}

// Anti-pattern: Async-over-sync (false async)
public async Task<Order> GetOrderAsync(int id)
{
    // This just wraps sync in Task, no benefit
    return await Task.Run(() => _repository.GetOrderSync(id));
}

// Correct: True async all the way
public async Task<Order> GetOrderAsync(int id, CancellationToken ct)
{
    return await _repository.GetOrderAsync(id, ct);
}
```

### ConfigureAwait Considerations

```csharp
// In libraries (no context needed)
public async Task<Order> GetOrderAsync(int id)
{
    var data = await _httpClient.GetAsync($"/orders/{id}")
        .ConfigureAwait(false);
    return await data.Content.ReadFromJsonAsync<Order>()
        .ConfigureAwait(false);
}

// In ASP.NET Core controllers (context flows automatically)
// ConfigureAwait(false) is unnecessary and can hide bugs
[HttpGet("{id}")]
public async Task<ActionResult<Order>> GetOrder(int id)
{
    var order = await _orderService.GetOrderAsync(id);
    return Ok(order);
}
```

### ValueTask for Hot Paths

```csharp
// When result is often cached/synchronous
public class CachedOrderService
{
    private readonly IMemoryCache _cache;
    private readonly IOrderRepository _repository;

    public ValueTask<Order> GetOrderAsync(int id)
    {
        if (_cache.TryGetValue($"order:{id}", out Order? cached))
        {
            // No allocation for cached case
            return ValueTask.FromResult(cached!);
        }

        // Async path when cache miss
        return new ValueTask<Order>(GetOrderSlowPath(id));
    }

    private async Task<Order> GetOrderSlowPath(int id)
    {
        var order = await _repository.GetOrderAsync(id);
        _cache.Set($"order:{id}", order, TimeSpan.FromMinutes(5));
        return order;
    }
}
```

---

## Hands-On Exercise: Profile OrderFlow's Slow Endpoint

### Step 1: Create a Deliberately Slow Endpoint

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrderReportsController : ControllerBase
{
    private readonly OrderDbContext _context;

    [HttpGet("slow")]
    public async Task<ActionResult<OrderReport>> GetSlowReport()
    {
        // Multiple performance issues for you to find:
        var orders = await _context.Orders
            .Include(o => o.Lines)
            .ToListAsync();

        var report = new OrderReport();

        foreach (var order in orders)
        {
            // N+1: Loading customer for each order
            var customer = await _context.Customers
                .FirstAsync(c => c.Id == order.CustomerId);

            // String concatenation in loop
            report.Summary += $"Order {order.Id}: {customer.Name}\n";

            // Unnecessary serialization
            var json = JsonConvert.SerializeObject(order);
            report.TotalBytes += json.Length;
        }

        return report;
    }
}
```

### Step 2: Profile and Find Issues

```bash
# Start your API
dotnet run

# In another terminal, capture a trace while hitting the endpoint
dotnet-trace collect -p $(pgrep -f OrderFlow.Api) --duration 00:00:30 &
curl http://localhost:5000/api/orderreports/slow

# Convert and analyze
dotnet-trace convert trace.nettrace --format speedscope
```

### Step 3: Document Your Findings

Create `profiling-results.md`:

```markdown
# OrderFlow Profiling Results

## Endpoint: GET /api/orderreports/slow

### Baseline
- Response time: 2,340ms
- Memory allocated: 45MB
- Database queries: 1,247

### Issues Found
1. **N+1 Query Problem** (60% of time)
   - Loading customers individually for each order
   - 1,245 extra database queries

2. **String Concatenation** (25% of time)
   - Building summary with += in loop
   - 1,244 intermediate string allocations

3. **Unnecessary Serialization** (15% of time)
   - Serializing each order to JSON just to count bytes
   - 1,244 JSON serialization operations

### Optimizations Applied
[Document each fix and its impact]
```

---

## Common Performance Anti-Patterns

### Anti-Pattern: Chatty Database Access

```csharp
// Anti-pattern: Multiple round trips
public async Task<OrderSummary> GetOrderSummary(int customerId)
{
    var customer = await _context.Customers.FindAsync(customerId);
    var orders = await _context.Orders
        .Where(o => o.CustomerId == customerId)
        .ToListAsync();
    var totalSpent = await _context.Orders
        .Where(o => o.CustomerId == customerId)
        .SumAsync(o => o.Total);

    return new OrderSummary(customer, orders, totalSpent);
}

// Better: Single query with projection
public async Task<OrderSummary> GetOrderSummaryOptimized(int customerId)
{
    return await _context.Customers
        .Where(c => c.Id == customerId)
        .Select(c => new OrderSummary
        {
            CustomerName = c.Name,
            OrderCount = c.Orders.Count,
            TotalSpent = c.Orders.Sum(o => o.Total),
            RecentOrders = c.Orders
                .OrderByDescending(o => o.CreatedAt)
                .Take(5)
                .Select(o => new OrderDto { Id = o.Id, Total = o.Total })
                .ToList()
        })
        .FirstOrDefaultAsync();
}
```

### Anti-Pattern: Unbounded Collections

```csharp
// Anti-pattern: Load everything
public async Task<List<Order>> GetAllOrders()
{
    return await _context.Orders.ToListAsync();  // Could be millions
}

// Better: Always paginate
public async Task<PagedResult<Order>> GetOrders(int page, int pageSize)
{
    var query = _context.Orders.OrderByDescending(o => o.CreatedAt);

    var total = await query.CountAsync();
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return new PagedResult<Order>(items, total, page, pageSize);
}
```

---

## Deliverables

1. **Profiling Report**: Document baseline measurements for 3 OrderFlow endpoints
2. **Benchmark Suite**: BenchmarkDotNet tests for critical code paths
3. **SLO Document**: Define SLOs for OrderFlow's key operations
4. **Optimization Log**: Before/after metrics for at least one optimization

---

## Reflection Questions

1. What's the difference between optimizing latency vs throughput?
2. When would you choose Server GC vs Workstation GC?
3. How do you decide if a performance improvement is statistically significant?
4. What's the cost of premature optimization?

---

## Resources

### Tools
- [BenchmarkDotNet](https://benchmarkdotnet.org/)
- [dotnet-trace](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)
- [PerfView](https://github.com/microsoft/perfview)
- [Speedscope](https://speedscope.app/)

### Learning
- [Adam Sitnik - Profiling .NET Code](https://www.youtube.com/watch?v=4yALYEINbyI)
- [Writing High-Performance .NET Code](https://www.writinghighperf.net/)
- [.NET Performance Tips](https://docs.microsoft.com/en-us/dotnet/framework/performance/performance-tips)

### Deep Dives
- [Span<T> and Memory<T>](https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans/)
- [GC Fundamentals](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)
- [ArrayPool and Object Pooling](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)

---

**Next:** [Part 2 — Scaling APIs Horizontally](./part-2.md)
