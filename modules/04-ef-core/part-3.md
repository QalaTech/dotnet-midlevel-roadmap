# 04 — Entity Framework Core & Data Access

## Part 3 / 5 — Querying and Performance

---

## Why This Matters

Queries are where EF Core either shines or destroys your application:
- A bad query can take 30 seconds instead of 30 milliseconds
- Client-side evaluation can crash your server with out-of-memory errors
- N+1 queries can generate thousands of database round trips
- Poor tracking decisions waste memory and cause stale data bugs

Understanding how LINQ becomes SQL is essential for building performant applications.

---

## Core Concept #10 — LINQ to SQL Translation

### Why This Matters

EF Core translates your LINQ queries into SQL. But not everything translates. When it can't translate, it either:
1. Throws an exception (good — you know immediately)
2. Evaluates in memory (dangerous — pulls all data to application)

### What Goes Wrong Without This

**War story #1 — The Client-Side Evaluation Disaster:**
Developer writes innocent-looking code:
```csharp
var orders = await context.Orders
    .Where(o => CalculateAge(o.CreatedAt) > 30)
    .ToListAsync();

private int CalculateAge(DateTime date) => (DateTime.Now - date).Days;
```
EF Core can't translate `CalculateAge` to SQL. In EF Core 2.x, it silently pulled ALL orders to memory and filtered there. With 2 million orders, the server ran out of memory. (EF Core 3.0+ throws an exception instead.)

**War story #2 — The ToString Trap:**
```csharp
var orders = await context.Orders
    .Where(o => o.Status.ToString() == "Pending")
    .ToListAsync();
```
Enum's `ToString()` doesn't translate to SQL. Query evaluates client-side or throws.

### Translatable vs Non-Translatable

```csharp
// ✅ TRANSLATES — These become SQL
context.Orders.Where(o => o.Status == OrderStatus.Pending)
context.Orders.Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-30))
context.Orders.Where(o => o.Customer.Name.Contains("Acme"))
context.Orders.Where(o => EF.Functions.Like(o.Customer.Name, "%Corp%"))
context.Orders.OrderByDescending(o => o.CreatedAt).Take(10)
context.Orders.Select(o => new { o.Id, o.Status, o.Total.Amount })

// ❌ DOES NOT TRANSLATE — Will throw or evaluate client-side
context.Orders.Where(o => MyCustomMethod(o))
context.Orders.Where(o => o.GetType().Name == "Order")
context.Orders.Where(o => Regex.IsMatch(o.OrderNumber, @"\d+"))
context.Orders.Where(o => o.Items.Any(i => i.GetHashCode() > 0))
```

### See the SQL

```csharp
// Program.cs — Enable SQL logging
builder.Services.AddDbContext<OrderFlowDbContext>(options =>
    options.UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging());  // Shows parameter values

// In production, use structured logging
builder.Services.AddDbContext<OrderFlowDbContext>(options =>
    options.UseSqlServer(connectionString)
        .LogTo(
            message => Log.Information("{EFQuery}", message),
            new[] { DbLoggerCategory.Database.Command.Name },
            LogLevel.Information));
```

### Query Tags for Tracing

```csharp
// Add comments to generated SQL for easier debugging
var orders = await context.Orders
    .TagWith("GetPendingOrdersForDashboard")
    .TagWith($"RequestId: {httpContext.TraceIdentifier}")
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();

// Generated SQL:
// -- GetPendingOrdersForDashboard
// -- RequestId: 0HMVD5H3Q1234:00000001
// SELECT * FROM Orders WHERE Status = 'Pending'
```

---

## Core Concept #11 — Loading Related Data

### Why This Matters

How you load related data determines whether your page loads in 100ms or 10 seconds. There are three strategies, each with trade-offs.

### What Goes Wrong Without This

**The N+1 Query Problem:**
```csharp
// This innocent code generates N+1 queries
var orders = await context.Orders.ToListAsync();  // Query 1

foreach (var order in orders)
{
    Console.WriteLine($"Order {order.Id}");
    foreach (var item in order.Items)  // Query 2, 3, 4... N+1
    {
        Console.WriteLine($"  - {item.ProductName}");
    }
}

// With 1000 orders, this executes 1001 queries
// Each query has network latency, execution overhead
// Total time: 30+ seconds instead of <100ms
```

### Strategy 1: Eager Loading (Include)

```csharp
// Load related data in single query
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();

// Generated SQL (simplified):
// SELECT o.*, i.*, c.*
// FROM Orders o
// LEFT JOIN OrderItems i ON o.Id = i.OrderId
// LEFT JOIN Customers c ON o.CustomerId = c.Id
// WHERE o.Status = 'Pending'

// Nested includes
var orders = await context.Orders
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
            .ThenInclude(p => p.Categories)
    .ToListAsync();
```

**When to use:** When you know you need the related data and the result set is reasonably sized.

**Danger:** Including too much creates a Cartesian explosion:
```csharp
// DANGEROUS: Cartesian product
var orders = await context.Orders
    .Include(o => o.Items)           // 10 items per order
    .Include(o => o.AuditHistory)    // 50 audit records per order
    .ToListAsync();

// With 100 orders: 100 × 10 × 50 = 50,000 rows returned!
// Better to split into separate queries
```

### Strategy 2: Explicit Loading

```csharp
// Load related data on demand
var order = await context.Orders.FindAsync(orderId);

// Load items only if needed
if (order.Status == OrderStatus.Shipped)
{
    await context.Entry(order)
        .Collection(o => o.Items)
        .LoadAsync();
}

// With filter
await context.Entry(order)
    .Collection(o => o.Items)
    .Query()
    .Where(i => i.Quantity > 10)
    .LoadAsync();
```

**When to use:** When you conditionally need related data based on runtime logic.

### Strategy 3: Projection (The Often-Best Choice)

```csharp
// Only select what you need — most efficient
var orderSummaries = await context.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Select(o => new OrderSummaryDto
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name,
        ItemCount = o.Items.Count,
        TotalAmount = o.Total.Amount,
        CreatedAt = o.CreatedAt
    })
    .ToListAsync();

// Generated SQL only selects needed columns
// No entity tracking overhead
// No extra data transferred
```

**When to use:** For read-only scenarios, especially lists and reports.

### Split Queries

```csharp
// Avoid Cartesian explosion with split queries
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.AuditHistory)
    .AsSplitQuery()  // Executes as multiple queries
    .ToListAsync();

// Executes:
// Query 1: SELECT * FROM Orders
// Query 2: SELECT * FROM OrderItems WHERE OrderId IN (...)
// Query 3: SELECT * FROM AuditHistory WHERE OrderId IN (...)

// Configure globally if your app often has multiple includes
builder.Services.AddDbContext<OrderFlowDbContext>(options =>
    options.UseSqlServer(connectionString)
        .UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

---

## Core Concept #12 — Tracking vs No-Tracking

### Why This Matters

EF Core's change tracker:
- Monitors entities for changes
- Enables lazy loading
- Ensures identity (same ID = same object instance)
- Uses memory

For read-only queries, tracking is pure overhead.

### What Goes Wrong Without This

**Memory bloat:** Loading 10,000 entities with tracking keeps them all in memory for the DbContext lifetime. Each entity is tracked with snapshots of original values, doubling memory usage.

**Stale data in long-lived contexts:** If a background service keeps a DbContext alive for hours, tracked entities become stale. Changes made by other processes aren't visible.

### No-Tracking Queries

```csharp
// Per-query no-tracking
var orders = await context.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();

// Per-query with identity resolution (for includes)
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()  // Same customer = same object
    .Include(o => o.Customer)
    .ToListAsync();

// Configure DbContext default
builder.Services.AddDbContext<OrderFlowDbContext>(options =>
    options.UseSqlServer(connectionString)
        .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

### When to Use Each

| Scenario | Tracking | No-Tracking |
|----------|----------|-------------|
| Read-only lists, reports | | ✅ |
| Editing entities | ✅ | |
| API GET endpoints returning DTOs | | ✅ |
| Background processing large datasets | | ✅ |
| Complex business logic with updates | ✅ | |

### The Attach Pattern for Updates

```csharp
// When reading with no-tracking, then needing to update:
public async Task UpdateOrderStatusAsync(Guid orderId, OrderStatus newStatus)
{
    // Option 1: Read with tracking (extra query, but safe)
    var order = await context.Orders.FindAsync(orderId);
    order.Status = newStatus;
    await context.SaveChangesAsync();

    // Option 2: Create and attach (no read query, but risky)
    var order = new Order { Id = orderId };
    context.Attach(order);
    order.Status = newStatus;
    await context.SaveChangesAsync();
    // DANGER: Only updates Status column, other properties set to defaults

    // Option 3: ExecuteUpdate (best for single-field updates, EF Core 7+)
    await context.Orders
        .Where(o => o.Id == orderId)
        .ExecuteUpdateAsync(o => o.SetProperty(x => x.Status, newStatus));
}
```

---

## Core Concept #13 — Filtering and Pagination

### Why This Matters

Every list endpoint needs filtering and pagination. Get it wrong and you'll:
- Return 10,000 results when the user wanted 10
- Crash on pages beyond the first few
- Have inconsistent results between pages

### Offset Pagination

```csharp
// Simple but has problems at scale
public async Task<PagedResult<OrderDto>> GetOrdersAsync(
    int pageNumber,
    int pageSize,
    OrderStatus? statusFilter = null)
{
    var query = context.Orders.AsNoTracking();

    if (statusFilter.HasValue)
        query = query.Where(o => o.Status == statusFilter.Value);

    var totalCount = await query.CountAsync();

    var items = await query
        .OrderByDescending(o => o.CreatedAt)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(o => new OrderDto
        {
            Id = o.Id,
            Status = o.Status.ToString(),
            CustomerName = o.Customer.Name,
            TotalAmount = o.Total.Amount,
            CreatedAt = o.CreatedAt
        })
        .ToListAsync();

    return new PagedResult<OrderDto>
    {
        Items = items,
        TotalCount = totalCount,
        PageNumber = pageNumber,
        PageSize = pageSize,
        TotalPages = (int)Math.Ceiling(totalCount / (double)pageSize)
    };
}

// Problem: Page 10000 scans 10000 * pageSize rows
// OFFSET pagination degrades as page number increases
```

### Keyset (Cursor) Pagination

```csharp
// Consistent performance regardless of page depth
public async Task<CursorPagedResult<OrderDto>> GetOrdersAsync(
    int pageSize,
    DateTime? cursorCreatedAt = null,
    Guid? cursorId = null)
{
    var query = context.Orders.AsNoTracking();

    // Apply cursor filter (continue from last seen item)
    if (cursorCreatedAt.HasValue && cursorId.HasValue)
    {
        query = query.Where(o =>
            o.CreatedAt < cursorCreatedAt.Value ||
            (o.CreatedAt == cursorCreatedAt.Value && o.Id < cursorId.Value));
    }

    var items = await query
        .OrderByDescending(o => o.CreatedAt)
        .ThenByDescending(o => o.Id)  // Tiebreaker for same timestamp
        .Take(pageSize + 1)  // Fetch one extra to check for more
        .Select(o => new OrderDto
        {
            Id = o.Id,
            Status = o.Status.ToString(),
            CustomerName = o.Customer.Name,
            TotalAmount = o.Total.Amount,
            CreatedAt = o.CreatedAt
        })
        .ToListAsync();

    var hasMore = items.Count > pageSize;
    if (hasMore)
        items.RemoveAt(items.Count - 1);

    var lastItem = items.LastOrDefault();

    return new CursorPagedResult<OrderDto>
    {
        Items = items,
        HasMore = hasMore,
        NextCursor = lastItem != null
            ? $"{lastItem.CreatedAt:o}_{lastItem.Id}"
            : null
    };
}
```

### Dynamic Filtering

```csharp
// Build queries dynamically based on filter parameters
public async Task<List<OrderDto>> SearchOrdersAsync(OrderSearchCriteria criteria)
{
    var query = context.Orders.AsNoTracking();

    // Conditional filtering — only adds WHERE clauses for provided filters
    if (criteria.CustomerId.HasValue)
        query = query.Where(o => o.CustomerId == criteria.CustomerId.Value);

    if (criteria.Status.HasValue)
        query = query.Where(o => o.Status == criteria.Status.Value);

    if (criteria.MinAmount.HasValue)
        query = query.Where(o => o.Total.Amount >= criteria.MinAmount.Value);

    if (criteria.MaxAmount.HasValue)
        query = query.Where(o => o.Total.Amount <= criteria.MaxAmount.Value);

    if (criteria.FromDate.HasValue)
        query = query.Where(o => o.CreatedAt >= criteria.FromDate.Value);

    if (criteria.ToDate.HasValue)
        query = query.Where(o => o.CreatedAt <= criteria.ToDate.Value);

    if (!string.IsNullOrWhiteSpace(criteria.SearchTerm))
    {
        query = query.Where(o =>
            o.OrderNumber.Contains(criteria.SearchTerm) ||
            o.Customer.Name.Contains(criteria.SearchTerm));
    }

    return await query
        .OrderByDescending(o => o.CreatedAt)
        .Take(criteria.MaxResults ?? 100)
        .Select(o => new OrderDto { /* ... */ })
        .ToListAsync();
}
```

---

## Core Concept #14 — Global Query Filters

### Why This Matters

Some filters should apply to every query:
- Soft delete (IsDeleted = false)
- Multi-tenancy (TenantId = current tenant)
- Data visibility (only show user's own data)

Adding these manually to every query is error-prone.

### What Goes Wrong Without This

**War story:** Multi-tenant SaaS application. Developer forgets tenant filter on one report query. Customer sees another customer's data. GDPR violation. Massive fine.

### Implementing Global Filters

```csharp
// Entity with soft delete
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}

// Configuration
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasQueryFilter(p => !p.IsDeleted);
    }
}

// Now ALL queries automatically filter out deleted products
var products = await context.Products.ToListAsync();
// SELECT * FROM Products WHERE IsDeleted = 0
```

### Multi-Tenant Filter

```csharp
public interface ITenantService
{
    Guid GetCurrentTenantId();
}

public class OrderFlowDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public OrderFlowDbContext(
        DbContextOptions<OrderFlowDbContext> options,
        ITenantService tenantService) : base(options)
    {
        _tenantService = tenantService;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply tenant filter to all tenant-scoped entities
        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenantService.GetCurrentTenantId());

        modelBuilder.Entity<Customer>()
            .HasQueryFilter(c => c.TenantId == _tenantService.GetCurrentTenantId());

        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantService.GetCurrentTenantId());
    }
}
```

### Bypassing Filters

```csharp
// When you need to see filtered data (admin scenarios)
var allProducts = await context.Products
    .IgnoreQueryFilters()
    .ToListAsync();

// Includes deleted products — use carefully!

// For admin cleanup of deleted items:
var deletedProducts = await context.Products
    .IgnoreQueryFilters()
    .Where(p => p.IsDeleted)
    .Where(p => p.DeletedAt < DateTime.UtcNow.AddDays(-90))
    .ToListAsync();
```

---

## Core Concept #15 — Compiled Queries

### Why This Matters

Every time you execute a LINQ query, EF Core:
1. Parses the expression tree
2. Generates SQL
3. Creates parameter bindings

For hot paths, this overhead adds up. Compiled queries cache this work.

### When to Use

- High-frequency queries (thousands per second)
- Simple queries that don't vary structurally
- After profiling confirms query compilation is a bottleneck

```csharp
public class OrderRepository
{
    // Compiled query — compiled once, reused forever
    private static readonly Func<OrderFlowDbContext, Guid, Task<Order?>>
        GetOrderByIdQuery = EF.CompileAsyncQuery(
            (OrderFlowDbContext context, Guid id) =>
                context.Orders
                    .Include(o => o.Items)
                    .FirstOrDefault(o => o.Id == id));

    private static readonly Func<OrderFlowDbContext, Guid, IAsyncEnumerable<Order>>
        GetOrdersByCustomerQuery = EF.CompileAsyncQuery(
            (OrderFlowDbContext context, Guid customerId) =>
                context.Orders
                    .AsNoTracking()
                    .Where(o => o.CustomerId == customerId)
                    .OrderByDescending(o => o.CreatedAt));

    private readonly OrderFlowDbContext _context;

    public OrderRepository(OrderFlowDbContext context)
    {
        _context = context;
    }

    public Task<Order?> GetByIdAsync(Guid id)
        => GetOrderByIdQuery(_context, id);

    public IAsyncEnumerable<Order> GetByCustomerAsync(Guid customerId)
        => GetOrdersByCustomerQuery(_context, customerId);
}
```

### Limitations

Compiled queries can't use:
- Dynamic includes (`Include` must be fixed at compile time)
- Skip/Take with variable values
- Conditional where clauses

---

## Hands-On: OrderFlow Query Optimization

### Task 1 — Enable SQL Logging

Configure OrderFlow to log all SQL queries with execution times. Identify the slowest queries.

### Task 2 — Fix N+1 Queries

Find and fix any N+1 query problems in the order listing endpoint.

### Task 3 — Implement Cursor Pagination

Replace offset pagination with cursor pagination for the order list API.

### Task 4 — Add Global Filters

Implement soft delete for Products with a global query filter.

### Task 5 — Benchmark Projections

Compare:
1. Loading full Order entities with tracking
2. Loading with AsNoTracking
3. Projecting to DTOs

Measure memory usage and execution time.

---

## Deliverables

1. **SQL logging configuration** showing query output with execution times
2. **Fixed N+1 queries** with before/after SQL comparison
3. **Cursor pagination implementation** for order listing
4. **Global query filter** for soft delete
5. **Benchmark results** comparing tracking vs no-tracking vs projection

---

## Resources

### Must-Read
- [EF Core Query Performance](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying)
- [Pagination in EF Core](https://learn.microsoft.com/en-us/ef/core/querying/pagination)
- [Global Query Filters](https://learn.microsoft.com/en-us/ef/core/querying/filters)

### Videos
- [Nick Chapsas: EF Core Query Performance](https://www.youtube.com/watch?v=QhmNh5EQHOA)
- [Raw Coding: N+1 Problem Explained](https://www.youtube.com/watch?v=97qbRZq5d6I)
- [Milan Jovanovic: EF Core Mistakes](https://www.youtube.com/watch?v=4DpJljpXvjQ)

### Tools
- [MiniProfiler](https://miniprofiler.com/) — Query profiling
- [Glimpse](https://github.com/Glimpse/Glimpse) — Diagnostics
- [EF Core Power Tools](https://marketplace.visualstudio.com/items?itemName=ErikEJ.EFCorePowerTools) — Visualize queries

---

## Reflection Questions

1. Why does offset pagination get slower as page numbers increase?
2. When would you use tracking queries for a read-only scenario?
3. How do global query filters affect query plan caching?
4. What's the trade-off between eager loading and projection?

---

**Next:** [Part 4 — Advanced EF Core Patterns](./part-4.md)
