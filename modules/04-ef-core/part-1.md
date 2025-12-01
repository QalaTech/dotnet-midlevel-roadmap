# 04 — Entity Framework Core & Data Access

## Part 1 / 5 — Database Design and Modeling

---

## Why This Module Exists

Every backend application has data. How you store, retrieve, and modify that data determines:
- Whether your app survives growth
- How fast features can be built
- Whether data corruption is possible
- How painful migrations become

EF Core is powerful, but misusing it creates performance nightmares and maintenance disasters. This module teaches you to use it correctly — and know when not to use it.

---

## What You'll Build

OrderFlow's data layer:
- Complete database schema with proper relationships
- EF Core configuration with fluent API
- Migration strategy for safe schema evolution
- Performance-optimized queries
- Hybrid EF Core + Dapper for hot paths

---

## Core Concept #1 — Database Design Principles

### Why This Matters

A bad database design lives with you forever:
- Schema changes in production are risky
- Bad indexes mean slow queries
- Missing constraints mean corrupted data
- Poor relationships mean complex application code

### What Goes Wrong Without This

**Real scenario:** A developer stores order items as a JSON blob in the orders table. Queries are simple at first. Then: "Show me all orders containing product X." The query scans every order, parsing JSON. With 100,000 orders, the query takes 30 seconds.

**Real scenario:** No foreign keys because "the application handles it." A bug in the order deletion code orphans 50,000 order items. Nobody notices for months. Reports are wrong. Customers are billed incorrectly.

### Normalization (When and Why)

Normalization eliminates data redundancy:

```
❌ DENORMALIZED (problems waiting to happen)

Orders Table:
| OrderId | CustomerName | CustomerEmail     | ProductName | ProductPrice |
|---------|--------------|-------------------|-------------|--------------|
| 1       | Acme Corp    | orders@acme.com   | Widget      | 29.99        |
| 2       | Acme Corp    | orders@acme.com   | Gadget      | 49.99        |

Problem: Customer updates their email. Now you have to update every order.
Problem: Customer email typo in one order but not another. Which is correct?

✅ NORMALIZED (single source of truth)

Customers Table:
| CustomerId | Name      | Email             |
|------------|-----------|-------------------|
| 1          | Acme Corp | orders@acme.com   |

Orders Table:
| OrderId | CustomerId |
|---------|------------|
| 1       | 1          |
| 2       | 1          |

Update email once, everywhere is correct.
```

### When to Denormalize

Sometimes denormalization is the right choice:
- **Reporting tables** — Precomputed aggregates for dashboards
- **Audit trails** — Snapshot data at the time of the event
- **Performance-critical reads** — After proving normalization is too slow

```csharp
// Good denormalization: Order snapshot preserves historical data
public class OrderItem
{
    public Guid Id { get; set; }
    public Guid OrderId { get; set; }
    public Guid ProductId { get; set; }

    // Denormalized: Preserve price at time of order
    // If product price changes, order total shouldn't change
    public decimal UnitPrice { get; set; }
    public string ProductName { get; set; }  // Preserve even if product is renamed
}
```

### Constraints: Let the Database Help You

```sql
-- Primary key: Unique identifier
ALTER TABLE Orders ADD CONSTRAINT PK_Orders PRIMARY KEY (Id);

-- Foreign key: Referential integrity
ALTER TABLE OrderItems ADD CONSTRAINT FK_OrderItems_Orders
    FOREIGN KEY (OrderId) REFERENCES Orders(Id);

-- Unique constraint: Business rules
ALTER TABLE Products ADD CONSTRAINT UQ_Products_Sku UNIQUE (Sku);

-- Check constraint: Value validation
ALTER TABLE OrderItems ADD CONSTRAINT CK_OrderItems_Quantity
    CHECK (Quantity > 0 AND Quantity <= 10000);

-- Default: Consistent initialization
ALTER TABLE Orders ADD CONSTRAINT DF_Orders_Status
    DEFAULT 'Pending' FOR Status;
```

### Why Foreign Keys Matter

```csharp
// Without FK: This silently creates orphaned data
var item = new OrderItem
{
    OrderId = Guid.NewGuid(),  // Order doesn't exist
    ProductId = someProduct.Id,
    Quantity = 1
};
context.OrderItems.Add(item);
await context.SaveChangesAsync();  // Succeeds! Data is now corrupt.

// With FK: Database catches the error
// SqlException: The INSERT statement conflicted with the
// FOREIGN KEY constraint "FK_OrderItems_Orders"
```

---

## Core Concept #2 — EF Core Entity Configuration

### Why This Matters

EF Core needs to know how your classes map to database tables. You can:
1. Use conventions (magic, implicit)
2. Use data annotations (attributes on classes)
3. Use Fluent API (explicit configuration)

**Recommendation:** Use Fluent API. It keeps your domain classes clean and puts all database concerns in one place.

### What Goes Wrong Without This

**Real scenario:** A developer adds a new property `InternalNotes` to the Order class. EF Core convention creates a nullable `nvarchar(max)` column. In production, this column grows to hold megabytes of data per row. Queries slow down because the column is always loaded.

**Real scenario:** Conventions name a column `OrderOrderItems` because of nested naming. Nobody notices until a DBA asks why the column has such a strange name.

### Fluent API Configuration

```csharp
// OrderFlow.Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // Table name
        builder.ToTable("Orders");

        // Primary key
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .ValueGeneratedNever();  // Application generates GUIDs

        // Required properties with constraints
        builder.Property(o => o.Status)
            .HasConversion<string>()  // Store enum as string
            .HasMaxLength(50)
            .IsRequired();

        builder.Property(o => o.CreatedAt)
            .IsRequired();

        // Value object mapping (owned type)
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

        builder.OwnsOne(o => o.ShippingAddress, address =>
        {
            address.Property(a => a.Line1).HasMaxLength(100).IsRequired();
            address.Property(a => a.Line2).HasMaxLength(100);
            address.Property(a => a.City).HasMaxLength(50).IsRequired();
            address.Property(a => a.PostalCode).HasMaxLength(20).IsRequired();
            address.Property(a => a.Country).HasMaxLength(2).IsRequired();
        });

        // Relationships
        builder.HasOne<Customer>()
            .WithMany()
            .HasForeignKey(o => o.CustomerId)
            .OnDelete(DeleteBehavior.Restrict);  // Don't cascade delete

        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade);  // Delete items with order

        // Indexes
        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => o.Status);
        builder.HasIndex(o => o.CreatedAt);

        // Composite index for common query pattern
        builder.HasIndex(o => new { o.CustomerId, o.Status, o.CreatedAt })
            .HasDatabaseName("IX_Orders_Customer_Status_CreatedAt");
    }
}

// Product configuration with unique constraint
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Sku)
            .HasMaxLength(50)
            .IsRequired();

        builder.HasIndex(p => p.Sku)
            .IsUnique()
            .HasDatabaseName("UQ_Products_Sku");

        builder.Property(p => p.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.OwnsOne(p => p.Price, money =>
        {
            money.Property(m => m.Amount)
                .HasColumnName("Price")
                .HasPrecision(18, 2)
                .IsRequired();

            money.Property(m => m.Currency)
                .HasColumnName("PriceCurrency")
                .HasMaxLength(3)
                .IsRequired();
        });

        // Soft delete filter
        builder.HasQueryFilter(p => !p.IsDeleted);
    }
}
```

### Registering Configurations

```csharp
public class OrderFlowDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(
            typeof(OrderFlowDbContext).Assembly);

        // Or apply individually
        // modelBuilder.ApplyConfiguration(new OrderConfiguration());
    }
}
```

---

## Core Concept #3 — Relationships

### Why This Matters

Relationships define how entities connect. Getting them wrong means:
- N+1 query problems
- Incorrect cascade behavior
- Data integrity issues
- Confusing navigation properties

### What Goes Wrong Without This

**Real scenario:** Developer configures Order → Customer as cascade delete. Deleting a customer deletes all their orders. Business requirement was to prevent deletion if orders exist. Hundreds of orders disappear before anyone notices.

**Real scenario:** Both ends of a relationship have navigation properties but only one is configured. EF Core creates two foreign keys. Database has redundant columns.

### Relationship Types

```csharp
// ONE-TO-MANY: Customer has many Orders
public class Customer
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public ICollection<Order> Orders { get; set; } = new List<Order>();
}

public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public Customer Customer { get; set; }  // Navigation property
}

// Configuration
builder.HasMany(c => c.Orders)
    .WithOne(o => o.Customer)
    .HasForeignKey(o => o.CustomerId)
    .OnDelete(DeleteBehavior.Restrict);  // Can't delete customer with orders

// ONE-TO-ONE: Order has one ShipmentDetails
public class Order
{
    public Guid Id { get; set; }
    public ShipmentDetails? Shipment { get; set; }
}

public class ShipmentDetails
{
    public Guid Id { get; set; }
    public Guid OrderId { get; set; }  // FK and PK same column
    public Order Order { get; set; }
    public string TrackingNumber { get; set; }
    public DateTime? ShippedAt { get; set; }
}

// Configuration
builder.HasOne(o => o.Shipment)
    .WithOne(s => s.Order)
    .HasForeignKey<ShipmentDetails>(s => s.OrderId);

// MANY-TO-MANY: Products have many Categories, Categories have many Products
public class Product
{
    public Guid Id { get; set; }
    public ICollection<Category> Categories { get; set; } = new List<Category>();
}

public class Category
{
    public Guid Id { get; set; }
    public ICollection<Product> Products { get; set; } = new List<Product>();
}

// EF Core 5+ creates join table automatically
// Or explicit join entity for additional properties:
public class ProductCategory
{
    public Guid ProductId { get; set; }
    public Guid CategoryId { get; set; }
    public DateTime AddedAt { get; set; }
    public Product Product { get; set; }
    public Category Category { get; set; }
}
```

### Delete Behaviors

| Behavior | What happens when parent is deleted |
|----------|-------------------------------------|
| `Cascade` | Children are deleted too |
| `Restrict` | Exception if children exist |
| `SetNull` | FK set to null (requires nullable FK) |
| `NoAction` | Database decides (usually error) |
| `ClientSetNull` | FK set to null in tracked entities only |

```csharp
// Common patterns:
// Order → OrderItems: Cascade (items belong to order)
// Order → Customer: Restrict (can't delete customer with orders)
// Order → AssignedEmployee: SetNull (employee leaves, orders remain)
```

---

## Core Concept #4 — Indexing Strategy

### Why This Matters

Indexes make queries fast. But:
- Too few indexes → slow reads
- Too many indexes → slow writes
- Wrong indexes → indexes that don't help

### What Goes Wrong Without This

**Real scenario:** Table has 10 million rows. Query filters by `Status` and `CreatedAt`. No index exists on these columns. Query takes 45 seconds, scanning every row.

**Real scenario:** Developer adds an index on every column "just in case." Insert performance drops by 80% because every insert updates 15 indexes.

### Index Types

```csharp
// Single column index
builder.HasIndex(o => o.CustomerId)
    .HasDatabaseName("IX_Orders_CustomerId");

// Composite index (order matters!)
// Good for: WHERE CustomerId = X AND Status = Y
// Good for: WHERE CustomerId = X
// NOT good for: WHERE Status = Y (leftmost column not used)
builder.HasIndex(o => new { o.CustomerId, o.Status })
    .HasDatabaseName("IX_Orders_Customer_Status");

// Unique index (also enforces uniqueness)
builder.HasIndex(p => p.Sku)
    .IsUnique()
    .HasDatabaseName("UQ_Products_Sku");

// Filtered index (partial index)
// Only indexes pending orders, much smaller
builder.HasIndex(o => o.CreatedAt)
    .HasFilter("[Status] = 'Pending'")
    .HasDatabaseName("IX_Orders_CreatedAt_Pending");

// Covering index (includes all columns needed for query)
// Query can be satisfied entirely from index, no table lookup
builder.HasIndex(o => o.CustomerId)
    .IncludeProperties(o => new { o.Status, o.CreatedAt, o.TotalAmount })
    .HasDatabaseName("IX_Orders_Customer_Covering");
```

### Choosing Indexes

**Ask these questions:**
1. What queries does this table serve?
2. What columns appear in WHERE clauses?
3. What columns appear in ORDER BY?
4. What columns appear in JOIN conditions?

```csharp
// Query: Get recent orders for a customer
var orders = await context.Orders
    .Where(o => o.CustomerId == customerId)
    .Where(o => o.CreatedAt > cutoffDate)
    .OrderByDescending(o => o.CreatedAt)
    .Take(10)
    .ToListAsync();

// Index needed: (CustomerId, CreatedAt DESC)
// This index supports both the filter and the sort
```

---

## Hands-On: OrderFlow Database Design

### Task 1 — Create the Schema

Design and configure entities for:
- **Customers** — Name, email, credit limit
- **Products** — SKU, name, price, stock quantity
- **Orders** — Customer, status, shipping address, total
- **OrderItems** — Product, quantity, unit price (denormalized)

### Task 2 — Configure Relationships

Set up:
- Customer → Orders (one-to-many, restrict delete)
- Order → OrderItems (one-to-many, cascade delete)
- OrderItem → Product (many-to-one, restrict delete)

### Task 3 — Add Indexes

Create indexes for common queries:
- Orders by customer
- Orders by status
- Products by SKU (unique)
- Recent pending orders

### Task 4 — Create Initial Migration

```bash
cd src/OrderFlow.Infrastructure
dotnet ef migrations add InitialCreate -s ../OrderFlow.Api
dotnet ef database update -s ../OrderFlow.Api

# Review the generated SQL
dotnet ef migrations script -s ../OrderFlow.Api
```

---

## Deliverables

1. **Entity configurations** using Fluent API for all OrderFlow entities
2. **Relationship configuration** with appropriate delete behaviors
3. **Index strategy document** explaining why each index exists
4. **Initial migration** with review of generated SQL

---

## Resources

### Must-Read
- [EF Core Modeling](https://learn.microsoft.com/en-us/ef/core/modeling/)
- [EF Core Relationships](https://learn.microsoft.com/en-us/ef/core/modeling/relationships)
- [Indexing in SQL Server](https://use-the-index-luke.com/)

### Videos
- [Nick Chapsas: EF Core Relationships](https://www.youtube.com/watch?v=IYSB9zoXxBE)
- [Milan Jovanovic: EF Core Configuration](https://www.youtube.com/watch?v=xDp2WPXBB5U)
- [Brent Ozar: How to Think Like the SQL Engine](https://www.youtube.com/watch?v=5e0T3bWPQHs)

### Tools
- [dbdiagram.io](https://dbdiagram.io/) — Database diagramming
- [EF Core Power Tools](https://marketplace.visualstudio.com/items?itemName=ErikEJ.EFCorePowerTools)
- [Azure Data Studio](https://azure.microsoft.com/en-us/products/data-studio/)

---

## Reflection Questions

1. When should you denormalize, and what are the risks?
2. Why does index column order matter for composite indexes?
3. What delete behavior would you use for Order → Customer?
4. How do you decide between owned types and separate tables?

---

**Next:** [Part 2 — Migrations and Schema Evolution](./part-2.md)
