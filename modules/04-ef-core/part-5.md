# 04 — Entity Framework Core & Data Access

## Part 5 / 5 — Dapper and Raw SQL

---

## Why This Matters

EF Core is excellent for most scenarios, but sometimes you need to drop down to raw SQL:
- Performance-critical read paths where EF overhead matters
- Complex queries that LINQ can't express cleanly
- Legacy stored procedures you can't change
- Bulk operations before EF Core 7

Knowing when and how to use Dapper alongside EF Core makes you a complete data access developer.

---

## Core Concept #21 — When to Use Dapper

### Why This Matters

Dapper vs EF Core isn't a religious debate. It's an engineering decision based on trade-offs:

| Aspect | EF Core | Dapper |
|--------|---------|--------|
| Development speed | Faster (no SQL writing) | Slower |
| Query expressiveness | Limited by LINQ | Full SQL power |
| Performance overhead | Higher (tracking, etc.) | Minimal |
| Maintainability | Refactoring-friendly | SQL string management |
| Complex mappings | Built-in (owned types, etc.) | Manual |
| Compile-time safety | Strong | Weak (runtime SQL errors) |

### What Goes Wrong Without This

**Over-engineering:** Developer uses Dapper for everything because "it's faster." Spends hours writing mapping code. Changes to data model require updating SQL in 47 places. Bugs everywhere.

**Under-optimizing:** Developer uses EF Core for everything. Dashboard query that aggregates millions of rows takes 30 seconds because EF Core pulls entities into memory for aggregation.

### Decision Framework

```
Use EF Core when:
├── CRUD operations on entities
├── Relationships and navigation properties needed
├── Change tracking useful (audit, validation)
├── Domain logic involves entities
└── Query complexity is moderate

Use Dapper when:
├── Read-only reporting/analytics
├── Complex aggregations (GROUP BY, window functions)
├── Performance is critical (profiled, not assumed)
├── Stored procedures required
├── Very high throughput requirements
└── Queries don't map cleanly to entities
```

### Real Example: OrderFlow Dashboard

```csharp
// EF Core version — pulls all data, aggregates in memory
var stats = await context.Orders
    .Where(o => o.CreatedAt >= startDate)
    .GroupBy(o => o.Status)
    .Select(g => new { Status = g.Key, Count = g.Count(), Total = g.Sum(o => o.Total.Amount) })
    .ToListAsync();
// Works, but with 1M orders, pulls significant data

// Dapper version — database does the heavy lifting
var stats = await connection.QueryAsync<OrderStatusSummary>("""
    SELECT
        Status,
        COUNT(*) AS OrderCount,
        SUM(TotalAmount) AS TotalAmount,
        AVG(TotalAmount) AS AverageAmount,
        MIN(CreatedAt) AS FirstOrder,
        MAX(CreatedAt) AS LastOrder
    FROM Orders
    WHERE CreatedAt >= @StartDate
    GROUP BY Status
    ORDER BY OrderCount DESC
    """,
    new { StartDate = startDate });
// Database handles aggregation, minimal data transferred
```

---

## Core Concept #22 — Dapper Fundamentals

### Why This Matters

Dapper is a micro-ORM. It does one thing well: maps SQL results to objects. Understanding its patterns prevents common mistakes.

### Basic Queries

```csharp
using Dapper;

public class OrderDapperRepository
{
    private readonly IDbConnection _connection;

    public OrderDapperRepository(IDbConnection connection)
    {
        _connection = connection;
    }

    // Single result
    public async Task<Order?> GetByIdAsync(Guid id)
    {
        return await _connection.QueryFirstOrDefaultAsync<Order>(
            "SELECT * FROM Orders WHERE Id = @Id",
            new { Id = id });
    }

    // Multiple results
    public async Task<IEnumerable<Order>> GetByCustomerAsync(Guid customerId)
    {
        return await _connection.QueryAsync<Order>(
            "SELECT * FROM Orders WHERE CustomerId = @CustomerId ORDER BY CreatedAt DESC",
            new { CustomerId = customerId });
    }

    // Scalar value
    public async Task<int> CountPendingAsync()
    {
        return await _connection.ExecuteScalarAsync<int>(
            "SELECT COUNT(*) FROM Orders WHERE Status = @Status",
            new { Status = "Pending" });
    }

    // Execute (INSERT, UPDATE, DELETE)
    public async Task<int> UpdateStatusAsync(Guid id, string newStatus)
    {
        return await _connection.ExecuteAsync(
            "UPDATE Orders SET Status = @Status, UpdatedAt = @UpdatedAt WHERE Id = @Id",
            new { Id = id, Status = newStatus, UpdatedAt = DateTime.UtcNow });
    }
}
```

### Parameterization — Security First

```csharp
// ✅ CORRECT — Parameterized query
var orders = await connection.QueryAsync<Order>(
    "SELECT * FROM Orders WHERE CustomerId = @CustomerId",
    new { CustomerId = customerId });

// ❌ DANGEROUS — SQL injection vulnerability
var orders = await connection.QueryAsync<Order>(
    $"SELECT * FROM Orders WHERE CustomerId = '{customerId}'");
// Attacker can inject: "'; DROP TABLE Orders; --"

// ✅ Dynamic column names (rare, use carefully)
var validColumns = new HashSet<string> { "CreatedAt", "TotalAmount", "Status" };
if (!validColumns.Contains(sortColumn))
    throw new ArgumentException("Invalid sort column");

var sql = $"SELECT * FROM Orders ORDER BY {sortColumn}";  // Safe — whitelist validated
```

### Multiple Result Sets

```csharp
// Stored procedure returns multiple result sets
public async Task<OrderWithDetails> GetOrderWithDetailsAsync(Guid orderId)
{
    using var multi = await _connection.QueryMultipleAsync(
        "GetOrderWithDetails",
        new { OrderId = orderId },
        commandType: CommandType.StoredProcedure);

    var order = await multi.ReadFirstOrDefaultAsync<Order>();
    var items = await multi.ReadAsync<OrderItem>();
    var customer = await multi.ReadFirstOrDefaultAsync<Customer>();

    return new OrderWithDetails
    {
        Order = order,
        Items = items.ToList(),
        Customer = customer
    };
}
```

---

## Core Concept #23 — Multi-Mapping (Joins)

### Why This Matters

Dapper can map joined query results to multiple objects. This is where it differs from simple mapping — you control how related data is assembled.

### One-to-Many Mapping

```csharp
public async Task<IEnumerable<Order>> GetOrdersWithItemsAsync(Guid customerId)
{
    var orderDictionary = new Dictionary<Guid, Order>();

    await _connection.QueryAsync<Order, OrderItem, Order>(
        """
        SELECT o.*, i.*
        FROM Orders o
        LEFT JOIN OrderItems i ON o.Id = i.OrderId
        WHERE o.CustomerId = @CustomerId
        ORDER BY o.CreatedAt DESC, i.Id
        """,
        (order, item) =>
        {
            if (!orderDictionary.TryGetValue(order.Id, out var existingOrder))
            {
                existingOrder = order;
                existingOrder.Items = new List<OrderItem>();
                orderDictionary.Add(order.Id, existingOrder);
            }

            if (item != null)
            {
                existingOrder.Items.Add(item);
            }

            return existingOrder;
        },
        new { CustomerId = customerId },
        splitOn: "Id");  // Column where OrderItem starts

    return orderDictionary.Values;
}
```

### Many-to-Many with Custom Mapping

```csharp
public async Task<IEnumerable<Product>> GetProductsWithCategoriesAsync()
{
    var productDictionary = new Dictionary<Guid, Product>();

    await _connection.QueryAsync<Product, Category, Product>(
        """
        SELECT p.*, c.*
        FROM Products p
        LEFT JOIN ProductCategories pc ON p.Id = pc.ProductId
        LEFT JOIN Categories c ON pc.CategoryId = c.Id
        ORDER BY p.Name, c.Name
        """,
        (product, category) =>
        {
            if (!productDictionary.TryGetValue(product.Id, out var existingProduct))
            {
                existingProduct = product;
                existingProduct.Categories = new List<Category>();
                productDictionary.Add(product.Id, existingProduct);
            }

            if (category != null)
            {
                existingProduct.Categories.Add(category);
            }

            return existingProduct;
        },
        splitOn: "Id");

    return productDictionary.Values;
}
```

### Complex Joins with Multiple Split Points

```csharp
public async Task<IEnumerable<Order>> GetOrdersWithCustomerAndItemsAsync()
{
    var orderDictionary = new Dictionary<Guid, Order>();

    await _connection.QueryAsync<Order, Customer, OrderItem, Order>(
        """
        SELECT o.*, c.*, i.*
        FROM Orders o
        INNER JOIN Customers c ON o.CustomerId = c.Id
        LEFT JOIN OrderItems i ON o.Id = i.OrderId
        ORDER BY o.CreatedAt DESC
        """,
        (order, customer, item) =>
        {
            if (!orderDictionary.TryGetValue(order.Id, out var existingOrder))
            {
                existingOrder = order;
                existingOrder.Customer = customer;
                existingOrder.Items = new List<OrderItem>();
                orderDictionary.Add(order.Id, existingOrder);
            }

            if (item != null)
            {
                existingOrder.Items.Add(item);
            }

            return existingOrder;
        },
        splitOn: "Id,Id");  // Split at Customer.Id, then OrderItem.Id

    return orderDictionary.Values;
}
```

---

## Core Concept #24 — Performance Patterns

### Why This Matters

The point of using Dapper is performance. These patterns maximize that benefit.

### Buffered vs Unbuffered Queries

```csharp
// Default: Buffered (loads all results into memory)
var orders = await connection.QueryAsync<Order>(sql);
// Good for small result sets, random access needed

// Unbuffered: Streams results one at a time
var orders = connection.Query<Order>(sql, buffered: false);
// Good for large result sets, sequential processing

// Unbuffered async with IAsyncEnumerable (Dapper.AOT or custom)
public async IAsyncEnumerable<Order> StreamOrdersAsync()
{
    using var reader = await connection.ExecuteReaderAsync(
        "SELECT * FROM Orders ORDER BY CreatedAt DESC");

    var parser = reader.GetRowParser<Order>();

    while (await reader.ReadAsync())
    {
        yield return parser(reader);
    }
}
```

### Bulk Insert with Table-Valued Parameters

```csharp
// SQL Server: Use TVP for bulk inserts
public async Task BulkInsertOrderItemsAsync(Guid orderId, IEnumerable<OrderItem> items)
{
    // Create DataTable matching TVP structure
    var table = new DataTable();
    table.Columns.Add("ProductId", typeof(Guid));
    table.Columns.Add("Quantity", typeof(int));
    table.Columns.Add("UnitPrice", typeof(decimal));

    foreach (var item in items)
    {
        table.Rows.Add(item.ProductId, item.Quantity, item.UnitPrice);
    }

    var parameters = new DynamicParameters();
    parameters.Add("@OrderId", orderId);
    parameters.Add("@Items", table.AsTableValuedParameter("OrderItemType"));

    await connection.ExecuteAsync(
        "InsertOrderItems",
        parameters,
        commandType: CommandType.StoredProcedure);
}

// Corresponding SQL Server TVP and procedure:
// CREATE TYPE OrderItemType AS TABLE (ProductId uniqueidentifier, Quantity int, UnitPrice decimal(18,2))
// CREATE PROCEDURE InsertOrderItems @OrderId uniqueidentifier, @Items OrderItemType READONLY AS
// INSERT INTO OrderItems (Id, OrderId, ProductId, Quantity, UnitPrice)
// SELECT NEWID(), @OrderId, ProductId, Quantity, UnitPrice FROM @Items
```

### Query Caching with Prepared Statements

```csharp
// Dapper caches parsed queries by SQL string
// Identical SQL strings reuse cached execution plan

// ✅ Same SQL, different parameters — cached
await connection.QueryAsync<Order>(
    "SELECT * FROM Orders WHERE Status = @Status",
    new { Status = "Pending" });

await connection.QueryAsync<Order>(
    "SELECT * FROM Orders WHERE Status = @Status",
    new { Status = "Shipped" });  // Reuses cached plan

// ❌ Dynamic SQL — no caching, new plan each time
await connection.QueryAsync<Order>(
    $"SELECT * FROM Orders WHERE Status = '{status}'");  // Bad!
```

---

## Core Concept #25 — Hybrid EF Core + Dapper

### Why This Matters

You don't have to choose one or the other. Use EF Core for writes and complex domain operations; use Dapper for read-heavy reporting. The key is sharing connections and transactions correctly.

### Sharing the Connection

```csharp
public class HybridOrderRepository : IOrderRepository
{
    private readonly OrderFlowDbContext _context;

    public HybridOrderRepository(OrderFlowDbContext context)
    {
        _context = context;
    }

    // EF Core for writes (change tracking, validation, events)
    public async Task<Order> CreateAsync(Order order)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
        return order;
    }

    // Dapper for complex reads (reporting, dashboards)
    public async Task<IEnumerable<OrderSummary>> GetDashboardSummaryAsync(DateTime from, DateTime to)
    {
        // Get the connection from EF Core's context
        var connection = _context.Database.GetDbConnection();

        return await connection.QueryAsync<OrderSummary>(
            """
            SELECT
                CAST(CreatedAt AS DATE) AS Date,
                Status,
                COUNT(*) AS OrderCount,
                SUM(TotalAmount) AS TotalAmount,
                AVG(TotalAmount) AS AverageAmount
            FROM Orders
            WHERE CreatedAt BETWEEN @From AND @To
            GROUP BY CAST(CreatedAt AS DATE), Status
            ORDER BY Date DESC, Status
            """,
            new { From = from, To = to });
    }
}
```

### Sharing Transactions

```csharp
public async Task ProcessOrderWithReportingAsync(Guid orderId)
{
    // Start EF Core transaction
    await using var transaction = await _context.Database.BeginTransactionAsync();

    try
    {
        // EF Core write
        var order = await _context.Orders.FindAsync(orderId);
        order.Status = OrderStatus.Processing;
        await _context.SaveChangesAsync();

        // Dapper read using same transaction
        var connection = _context.Database.GetDbConnection();
        var dbTransaction = transaction.GetDbTransaction();

        var summary = await connection.QueryFirstAsync<OrderSummary>(
            "SELECT * FROM vw_OrderSummary WHERE OrderId = @OrderId",
            new { OrderId = orderId },
            transaction: dbTransaction);

        // More EF Core writes
        var auditLog = new AuditLog { OrderId = orderId, Action = "Processed", Summary = summary.ToString() };
        _context.AuditLogs.Add(auditLog);
        await _context.SaveChangesAsync();

        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

### Repository Pattern with Hybrid Support

```csharp
public interface IOrderRepository
{
    // EF Core operations
    Task<Order?> GetByIdAsync(Guid id);
    Task<Order> CreateAsync(Order order);
    Task UpdateAsync(Order order);

    // Dapper operations (read-only, reporting)
    Task<IEnumerable<OrderListItem>> GetListAsync(OrderListQuery query);
    Task<OrderStatistics> GetStatisticsAsync(DateTime from, DateTime to);
}

public class HybridOrderRepository : IOrderRepository
{
    private readonly OrderFlowDbContext _context;
    private IDbConnection Connection => _context.Database.GetDbConnection();

    public HybridOrderRepository(OrderFlowDbContext context)
    {
        _context = context;
    }

    // EF Core — full entity with tracking
    public async Task<Order?> GetByIdAsync(Guid id)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
    }

    // EF Core — change tracking for proper updates
    public async Task<Order> CreateAsync(Order order)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
        return order;
    }

    // Dapper — lightweight DTOs for lists
    public async Task<IEnumerable<OrderListItem>> GetListAsync(OrderListQuery query)
    {
        var sql = """
            SELECT
                o.Id,
                o.OrderNumber,
                o.Status,
                o.TotalAmount,
                o.CreatedAt,
                c.Name AS CustomerName
            FROM Orders o
            INNER JOIN Customers c ON o.CustomerId = c.Id
            WHERE (@CustomerId IS NULL OR o.CustomerId = @CustomerId)
              AND (@Status IS NULL OR o.Status = @Status)
            ORDER BY o.CreatedAt DESC
            OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY
            """;

        return await Connection.QueryAsync<OrderListItem>(sql, query);
    }

    // Dapper — complex aggregation
    public async Task<OrderStatistics> GetStatisticsAsync(DateTime from, DateTime to)
    {
        return await Connection.QueryFirstAsync<OrderStatistics>(
            """
            SELECT
                COUNT(*) AS TotalOrders,
                SUM(TotalAmount) AS TotalRevenue,
                AVG(TotalAmount) AS AverageOrderValue,
                COUNT(DISTINCT CustomerId) AS UniqueCustomers
            FROM Orders
            WHERE CreatedAt BETWEEN @From AND @To
            """,
            new { From = from, To = to });
    }
}
```

---

## Hands-On: OrderFlow Hybrid Data Access

### Task 1 — Benchmark EF Core vs Dapper

Create a benchmark comparing:
1. EF Core with tracking
2. EF Core with `AsNoTracking`
3. Dapper

For the same query returning 1,000 orders with items.

### Task 2 — Dashboard Query with Dapper

Implement a dashboard endpoint using Dapper that returns:
- Orders by status (last 30 days)
- Revenue by day
- Top 10 customers by order value

### Task 3 — Hybrid Repository

Create a repository that uses:
- EF Core for all write operations
- Dapper for list/search operations
- Shared transactions for complex operations

### Task 4 — Streaming Export

Implement a CSV export that streams 100,000 orders using Dapper's unbuffered mode without loading them all into memory.

### Task 5 — Stored Procedure Integration

Call an existing stored procedure using Dapper, mapping the results to DTOs.

---

## Deliverables

1. **Benchmark results** with analysis of when each approach is appropriate
2. **Dashboard API** using Dapper for aggregations
3. **Hybrid repository** demonstrating EF Core + Dapper coexistence
4. **Streaming export** for large data sets
5. **Documentation** describing when to use each approach in OrderFlow

---

## Resources

### Must-Read
- [Dapper GitHub Repository](https://github.com/DapperLib/Dapper)
- [Dapper Tutorial](https://www.learndapper.com/)
- [EF Core Raw SQL Queries](https://learn.microsoft.com/en-us/ef/core/querying/raw-sql)

### Videos
- [Nick Chapsas: Dapper vs EF Core](https://www.youtube.com/watch?v=F7GTmxBmlaU)
- [Raw Coding: When to Use Dapper](https://www.youtube.com/watch?v=txPjsBNkhyE)
- [Tim Corey: Dapper CRUD in C#](https://www.youtube.com/watch?v=Et2khGnrIqc)

### Benchmarking
- [BenchmarkDotNet](https://benchmarkdotnet.org/)
- [Realistic Data Access Benchmarks](https://github.com/exceptionnotfound/DapperVsEFCoreBenchmarks)
- [TechEmpower Benchmarks](https://www.techempower.com/benchmarks/)

### Libraries
- [Dapper.Contrib](https://github.com/DapperLib/Dapper.Contrib) — CRUD helpers
- [SqlKata](https://sqlkata.com/) — Query builder for Dapper
- [Dommel](https://github.com/henkmollema/Dommel) — Simple CRUD with Dapper

---

## Reflection Questions

1. At what scale does the EF Core overhead become noticeable?
2. How do you maintain SQL queries in a large codebase?
3. What are the risks of using raw SQL versus LINQ?
4. How do you test Dapper queries effectively?

---

## Module Summary

You've learned the complete data access stack:

1. **Database Design** — Normalization, constraints, relationships
2. **EF Core Configuration** — Fluent API, owned types, indexes
3. **Migrations** — Safe patterns, rollback strategies, team workflows
4. **Querying** — N+1 prevention, tracking, global filters
5. **Advanced Patterns** — Interceptors, value converters, concurrency
6. **Dapper** — When to use it, hybrid patterns

**Key Takeaways:**
- EF Core is your default choice — it's productive and safe
- Drop to Dapper when you have evidence (not assumptions) that EF Core is the bottleneck
- Always use parameterized queries — never concatenate user input into SQL
- Migrations are version control for your database — treat them with respect
- The best data access code is the code that's easy to understand and change

---

**Next Module:** [05 — NoSQL and Document Databases](../05-nosql/README.md)
