# 04 — Entity Framework Core & Data Access

## Part 4 / 5 — Advanced EF Core Patterns

---

## Why This Matters

EF Core has powerful advanced features that:
- Separate cross-cutting concerns from business logic
- Model complex domain concepts cleanly
- Handle concurrency without data loss
- Automate auditing and timestamps

Most developers never learn these features and write repetitive, fragile code instead.

---

## Core Concept #16 — Interceptors

### Why This Matters

Interceptors let you hook into EF Core's pipeline to:
- Add automatic auditing
- Log slow queries
- Modify commands before execution
- Add retry logic for transient failures

Without interceptors, you'd sprinkle this logic throughout your codebase.

### What Goes Wrong Without This

**Real scenario:** Team needs to track who modified every record and when. Developer adds `UpdatedBy` and `UpdatedAt` to every entity. Then adds code to set these fields in every repository method. Developer forgets to update one method. Audit trail has gaps. Compliance audit fails.

### SaveChanges Interceptor — Automatic Auditing

```csharp
public interface IAuditableEntity
{
    DateTime CreatedAt { get; set; }
    string CreatedBy { get; set; }
    DateTime? UpdatedAt { get; set; }
    string? UpdatedBy { get; set; }
}

public class AuditableInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUser;
    private readonly TimeProvider _timeProvider;

    public AuditableInterceptor(
        ICurrentUserService currentUser,
        TimeProvider timeProvider)
    {
        _currentUser = currentUser;
        _timeProvider = timeProvider;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context is null)
            return base.SavingChangesAsync(eventData, result, cancellationToken);

        var now = _timeProvider.GetUtcNow().UtcDateTime;
        var userId = _currentUser.UserId ?? "system";

        foreach (var entry in eventData.Context.ChangeTracker.Entries<IAuditableEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = now;
                    entry.Entity.CreatedBy = userId;
                    break;

                case EntityState.Modified:
                    entry.Entity.UpdatedAt = now;
                    entry.Entity.UpdatedBy = userId;
                    // Prevent modification of created fields
                    entry.Property(e => e.CreatedAt).IsModified = false;
                    entry.Property(e => e.CreatedBy).IsModified = false;
                    break;
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

// Registration
builder.Services.AddScoped<AuditableInterceptor>();
builder.Services.AddDbContext<OrderFlowDbContext>((sp, options) =>
    options.UseSqlServer(connectionString)
        .AddInterceptors(sp.GetRequiredService<AuditableInterceptor>()));
```

### Command Interceptor — Slow Query Logging

```csharp
public class SlowQueryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<SlowQueryInterceptor> _logger;
    private readonly TimeSpan _threshold = TimeSpan.FromMilliseconds(500);

    public SlowQueryInterceptor(ILogger<SlowQueryInterceptor> logger)
    {
        _logger = logger;
    }

    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Duration > _threshold)
        {
            _logger.LogWarning(
                "Slow query detected ({Duration}ms): {CommandText}",
                eventData.Duration.TotalMilliseconds,
                command.CommandText);
        }

        return base.ReaderExecutedAsync(command, eventData, result, cancellationToken);
    }
}
```

### Connection Interceptor — Multi-Tenancy

```csharp
public class TenantConnectionInterceptor : DbConnectionInterceptor
{
    private readonly ITenantService _tenantService;

    public TenantConnectionInterceptor(ITenantService tenantService)
    {
        _tenantService = tenantService;
    }

    public override async ValueTask<InterceptionResult> ConnectionOpeningAsync(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result,
        CancellationToken cancellationToken = default)
    {
        // Set session context for row-level security
        var tenantId = _tenantService.GetCurrentTenantId();

        await connection.OpenAsync(cancellationToken);

        await using var command = connection.CreateCommand();
        command.CommandText = $"EXEC sp_set_session_context 'TenantId', '{tenantId}'";
        await command.ExecuteNonQueryAsync(cancellationToken);

        return InterceptionResult.Suppress();  // We already opened the connection
    }
}
```

---

## Core Concept #17 — Value Converters

### Why This Matters

Value converters translate between .NET types and database types. They enable:
- Storing enums as strings (readable in database)
- Encrypting sensitive data at rest
- Using strongly-typed IDs
- Handling legacy database formats

### What Goes Wrong Without This

**Real scenario:** Database stores boolean as 'Y'/'N' strings (legacy system). Developer uses `bool` in C#. EF Core throws on first query because it can't convert.

### Enum to String Conversion

```csharp
public enum OrderStatus
{
    Pending,
    Processing,
    Shipped,
    Delivered,
    Cancelled
}

// Configuration
builder.Property(o => o.Status)
    .HasConversion<string>()  // Store as 'Pending', not 0
    .HasMaxLength(50);

// Or explicit converter for custom formatting
builder.Property(o => o.Status)
    .HasConversion(
        v => v.ToString().ToLowerInvariant(),  // To database
        v => Enum.Parse<OrderStatus>(v, ignoreCase: true));  // From database
```

### Strongly-Typed IDs

```csharp
// Strongly-typed ID prevents mixing up different ID types
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString();
}

public readonly record struct CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.NewGuid());
}

// Entity
public class Order
{
    public OrderId Id { get; set; }
    public CustomerId CustomerId { get; set; }  // Compiler prevents: order.CustomerId = someOrderId
}

// Value converter
public class OrderIdConverter : ValueConverter<OrderId, Guid>
{
    public OrderIdConverter()
        : base(
            id => id.Value,           // To database
            guid => new OrderId(guid))  // From database
    { }
}

// Configuration
builder.Property(o => o.Id)
    .HasConversion<OrderIdConverter>();

// Or register for all entities
protected override void ConfigureConventions(ModelConfigurationBuilder builder)
{
    builder.Properties<OrderId>()
        .HaveConversion<OrderIdConverter>();

    builder.Properties<CustomerId>()
        .HaveConversion<CustomerIdConverter>();
}
```

### Encrypting Sensitive Data

```csharp
public class EncryptedStringConverter : ValueConverter<string, string>
{
    public EncryptedStringConverter(IEncryptionService encryption)
        : base(
            plainText => encryption.Encrypt(plainText),
            cipherText => encryption.Decrypt(cipherText))
    { }
}

// Usage
builder.Property(c => c.SocialSecurityNumber)
    .HasConversion(new EncryptedStringConverter(encryptionService))
    .HasMaxLength(500);  // Encrypted text is longer

// CAUTION: Encrypted columns can't be queried with WHERE clauses
// Use this only for data that's displayed, not filtered
```

### JSON Column Conversion (EF Core 7+)

```csharp
public class OrderMetadata
{
    public string? Source { get; set; }
    public Dictionary<string, string> Tags { get; set; } = new();
    public List<string> Notes { get; set; } = new();
}

public class Order
{
    public Guid Id { get; set; }
    public OrderMetadata Metadata { get; set; } = new();
}

// Configuration — stores as JSON column
builder.OwnsOne(o => o.Metadata, m =>
{
    m.ToJson();
});

// Querying JSON (EF Core 7+ with SQL Server)
var orders = await context.Orders
    .Where(o => o.Metadata.Source == "Website")
    .ToListAsync();
```

---

## Core Concept #18 — Concurrency Handling

### Why This Matters

When two users edit the same record simultaneously:
- Last write wins → Data loss
- Optimistic concurrency → Conflict detected, user decides
- Pessimistic locking → Performance issues, deadlocks

Most applications need optimistic concurrency.

### What Goes Wrong Without This

**War story:** Two customer service reps open the same order. Rep A changes shipping address. Rep B changes quantity. Rep B saves first. Rep A saves second, overwriting Rep B's quantity change. Customer receives wrong quantity.

### Row Version (Timestamp) Concurrency

```csharp
public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public decimal TotalAmount { get; set; }

    // SQL Server automatically updates this on every change
    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// Configuration
builder.Property(o => o.RowVersion)
    .IsRowVersion();

// Usage
public async Task<Result> UpdateOrderAsync(Guid orderId, UpdateOrderCommand command)
{
    var order = await context.Orders.FindAsync(orderId);

    order.Status = command.NewStatus;
    order.TotalAmount = command.NewAmount;

    try
    {
        await context.SaveChangesAsync();
        return Result.Success();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // Someone else modified this record
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync();

        if (databaseValues == null)
        {
            return Result.Failure("Order was deleted by another user");
        }

        var dbOrder = (Order)databaseValues.ToObject();

        return Result.Conflict(
            $"Order was modified by another user. " +
            $"Current status: {dbOrder.Status}, " +
            $"Your status: {command.NewStatus}");
    }
}
```

### Property-Level Concurrency Token

```csharp
// Check specific properties instead of entire row
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public int StockQuantity { get; set; }

    [ConcurrencyCheck]  // Only check this property
    public decimal Price { get; set; }
}

// Configuration equivalent
builder.Property(p => p.Price)
    .IsConcurrencyToken();
```

### Handling Conflicts in APIs

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> UpdateOrder(
    Guid id,
    UpdateOrderRequest request,
    [FromHeader(Name = "If-Match")] string? etag)
{
    var order = await context.Orders.FindAsync(id);
    if (order == null)
        return NotFound();

    // Check ETag matches current version
    var currentEtag = Convert.ToBase64String(order.RowVersion);
    if (etag != null && etag != currentEtag)
    {
        return StatusCode(412, new ProblemDetails
        {
            Title = "Precondition Failed",
            Detail = "The resource has been modified since you last retrieved it"
        });
    }

    order.Status = request.Status;

    try
    {
        await context.SaveChangesAsync();

        // Return new ETag in response
        Response.Headers.ETag = Convert.ToBase64String(order.RowVersion);
        return Ok(order);
    }
    catch (DbUpdateConcurrencyException)
    {
        return Conflict(new ProblemDetails
        {
            Title = "Conflict",
            Detail = "The resource was modified by another request"
        });
    }
}
```

---

## Core Concept #19 — Owned Types and Value Objects

### Why This Matters

Domain-Driven Design uses Value Objects for concepts that:
- Are defined by their attributes, not an ID
- Are immutable
- Represent a cohesive concept (Money, Address, DateRange)

EF Core's owned types map these elegantly.

### What Goes Wrong Without This

**Real scenario:** Developer stores money as `decimal Amount` and `string Currency` as separate properties. Code accidentally adds USD to EUR. Bug found in production after incorrect charges.

### Modeling Money as Value Object

```csharp
public record Money
{
    public decimal Amount { get; init; }
    public string Currency { get; init; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative", nameof(amount));
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new ArgumentException("Currency must be 3-letter ISO code", nameof(currency));

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public static Money Zero(string currency) => new(0, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                $"Cannot add {Currency} to {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    public Money Multiply(int quantity)
        => new(Amount * quantity, Currency);
}

// Entity using the value object
public class Order
{
    public Guid Id { get; set; }
    public Money Total { get; private set; }

    public void SetTotal(Money total)
    {
        Total = total ?? throw new ArgumentNullException(nameof(total));
    }
}
```

### Configuring Owned Types

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.OwnsOne(o => o.Total, money =>
        {
            money.Property(m => m.Amount)
                .HasColumnName("TotalAmount")
                .HasPrecision(18, 2)
                .IsRequired();

            money.Property(m => m.Currency)
                .HasColumnName("TotalCurrency")
                .HasMaxLength(3)
                .IsRequired();
        });

        // Owned types are required by default
        // Make optional if needed:
        builder.Navigation(o => o.Total).IsRequired(false);
    }
}

// Result: Single table with columns: Id, TotalAmount, TotalCurrency
```

### Address as Owned Type

```csharp
public record Address
{
    public string Line1 { get; init; }
    public string? Line2 { get; init; }
    public string City { get; init; }
    public string PostalCode { get; init; }
    public string Country { get; init; }

    private Address() { }  // For EF Core

    public Address(string line1, string city, string postalCode, string country)
    {
        Line1 = line1 ?? throw new ArgumentNullException(nameof(line1));
        City = city ?? throw new ArgumentNullException(nameof(city));
        PostalCode = postalCode ?? throw new ArgumentNullException(nameof(postalCode));
        Country = country ?? throw new ArgumentNullException(nameof(country));
    }
}

public class Order
{
    public Guid Id { get; set; }
    public Address ShippingAddress { get; private set; }
    public Address? BillingAddress { get; private set; }  // Optional
}

// Configuration
builder.OwnsOne(o => o.ShippingAddress, address =>
{
    address.Property(a => a.Line1)
        .HasColumnName("ShippingLine1")
        .HasMaxLength(100);
    address.Property(a => a.City)
        .HasColumnName("ShippingCity")
        .HasMaxLength(50);
    // ... other properties
});

builder.OwnsOne(o => o.BillingAddress, address =>
{
    address.Property(a => a.Line1)
        .HasColumnName("BillingLine1")
        .HasMaxLength(100);
    // ... other properties
});
```

### Collection of Owned Types

```csharp
public class Order
{
    public Guid Id { get; set; }
    public ICollection<OrderLine> Lines { get; } = new List<OrderLine>();
}

public record OrderLine
{
    public string ProductSku { get; init; }
    public string ProductName { get; init; }
    public int Quantity { get; init; }
    public Money UnitPrice { get; init; }
}

// Configuration — creates separate table
builder.OwnsMany(o => o.Lines, line =>
{
    line.ToTable("OrderLines");

    line.WithOwner().HasForeignKey("OrderId");

    line.Property(l => l.ProductSku).HasMaxLength(50);
    line.Property(l => l.ProductName).HasMaxLength(200);

    line.OwnsOne(l => l.UnitPrice, money =>
    {
        money.Property(m => m.Amount).HasPrecision(18, 2);
        money.Property(m => m.Currency).HasMaxLength(3);
    });
});
```

---

## Core Concept #20 — Bulk Operations (EF Core 7+)

### Why This Matters

Traditional EF Core loads entities, modifies them, then saves. For bulk operations:
- Loading 100,000 entities wastes memory
- Tracking changes is slow
- Round-trips for each batch hurt performance

EF Core 7+ adds `ExecuteUpdate` and `ExecuteDelete` for direct database operations.

### What Goes Wrong Without This

**Real scenario:** Developer writes code to deactivate all products from a discontinued supplier:
```csharp
var products = await context.Products
    .Where(p => p.SupplierId == supplierId)
    .ToListAsync();  // Loads 50,000 products into memory

foreach (var product in products)
{
    product.IsActive = false;
}

await context.SaveChangesAsync();  // Generates 50,000 UPDATE statements
// Takes 15 minutes, times out
```

### Bulk Update

```csharp
// Single UPDATE statement, no entity loading
await context.Products
    .Where(p => p.SupplierId == supplierId)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.IsActive, false)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));

// Generated SQL:
// UPDATE Products SET IsActive = 0, UpdatedAt = @p0
// WHERE SupplierId = @p1

// Complex updates
await context.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Where(o => o.CreatedAt < DateTime.UtcNow.AddDays(-30))
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(o => o.Status, OrderStatus.Cancelled)
        .SetProperty(o => o.CancelledAt, DateTime.UtcNow)
        .SetProperty(o => o.CancellationReason, "Auto-cancelled: stale order"));
```

### Bulk Delete

```csharp
// Single DELETE statement
await context.Products
    .Where(p => p.IsDeleted)
    .Where(p => p.DeletedAt < DateTime.UtcNow.AddDays(-90))
    .ExecuteDeleteAsync();

// Generated SQL:
// DELETE FROM Products
// WHERE IsDeleted = 1 AND DeletedAt < @p0
```

### Cautions with Bulk Operations

```csharp
// ⚠️ Bypasses change tracker — interceptors and events don't fire
// ⚠️ Doesn't respect owned types nicely
// ⚠️ No cascade behavior — you must handle related entities

// If you need auditing, do it in the query:
await context.Orders
    .Where(o => o.Id == orderId)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(o => o.Status, newStatus)
        .SetProperty(o => o.UpdatedAt, DateTime.UtcNow)
        .SetProperty(o => o.UpdatedBy, currentUserId));
```

---

## Hands-On: OrderFlow Advanced Patterns

### Task 1 — Implement Auditing Interceptor

Create an interceptor that automatically sets `CreatedAt`, `CreatedBy`, `UpdatedAt`, and `UpdatedBy` on all auditable entities.

### Task 2 — Add Strongly-Typed IDs

Replace `Guid` IDs with strongly-typed IDs (`OrderId`, `CustomerId`, `ProductId`) throughout the domain model.

### Task 3 — Implement Money Value Object

Create a `Money` value object with currency validation and arithmetic operations. Use it for Order totals and Product prices.

### Task 4 — Add Concurrency Handling

Add row versioning to Orders and implement proper conflict detection in the API.

### Task 5 — Bulk Status Update

Create an endpoint that cancels all pending orders older than 30 days using `ExecuteUpdateAsync`.

---

## Deliverables

1. **Auditing interceptor** that works across all entities
2. **Strongly-typed ID implementation** with value converters
3. **Money value object** with owned type configuration
4. **Concurrency handling** with ETag support in API
5. **Bulk operations** for order management

---

## Resources

### Must-Read
- [EF Core Interceptors](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/interceptors)
- [Value Conversions](https://learn.microsoft.com/en-us/ef/core/modeling/value-conversions)
- [Owned Entity Types](https://learn.microsoft.com/en-us/ef/core/modeling/owned-entities)
- [Concurrency Tokens](https://learn.microsoft.com/en-us/ef/core/saving/concurrency)

### Videos
- [Nick Chapsas: Strongly Typed IDs](https://www.youtube.com/watch?v=z4SB5BkQX7M)
- [Milan Jovanovic: EF Core Interceptors](https://www.youtube.com/watch?v=2TrJVpABPVg)
- [Raw Coding: EF Core 7 Bulk Operations](https://www.youtube.com/watch?v=4w0EHk-qr6Y)

### Libraries
- [StronglyTypedId](https://github.com/andrewlock/StronglyTypedId) — Source generator for typed IDs
- [EFCore.NamingConventions](https://github.com/efcore/EFCore.NamingConventions) — Snake_case, etc.
- [EntityFrameworkCore.Triggered](https://github.com/koenbeuk/EntityFrameworkCore.Triggered) — Alternative to interceptors

---

## Reflection Questions

1. When would you use an interceptor vs overriding SaveChanges?
2. What are the downsides of strongly-typed IDs?
3. How do optimistic and pessimistic concurrency differ in user experience?
4. Why can't bulk operations use interceptors?

---

**Next:** [Part 5 — Dapper and Raw SQL](./part-5.md)
