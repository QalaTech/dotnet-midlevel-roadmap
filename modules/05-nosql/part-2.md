# 05 — NoSQL & Distributed Data

## Part 2 / 3 — Document Databases (MongoDB)

---

## Why This Matters

Document databases excel when:
- Data is hierarchical and self-contained
- Schema varies across documents
- You need to load entire objects in single reads
- Relationships are simple or can be denormalized

But using MongoDB like SQL leads to:
- Poor performance from excessive lookups
- Data duplication nightmares
- Queries that don't use indexes
- Aggregations that scan entire collections

Understanding document modeling is essential for NoSQL success.

---

## Core Concept #5 — Document Modeling Fundamentals

### Why This Matters

The #1 mistake with MongoDB is modeling data like a relational database. In SQL, you normalize to reduce redundancy. In MongoDB, you denormalize to reduce queries.

### What Goes Wrong Without This

**War story — The N+1 Disaster:**
Developer models MongoDB like SQL:
```javascript
// customers collection
{ "_id": "cust1", "name": "Acme Corp" }

// orders collection
{ "_id": "ord1", "customerId": "cust1", "items": [...] }

// products collection
{ "_id": "prod1", "name": "Widget", "price": 29.99 }
```

To display an order with customer and product names:
1. Query order (1 query)
2. Query customer (1 query)
3. Query each product in items (N queries)

Result: 10 products = 12 queries. Page takes 500ms.

**Better design — embedded documents:**
```javascript
{
  "_id": "ord1",
  "customer": {
    "id": "cust1",
    "name": "Acme Corp"  // Denormalized
  },
  "items": [
    {
      "productId": "prod1",
      "productName": "Widget",  // Denormalized
      "price": 29.99  // Snapshot at order time
    }
  ]
}
```
One query. Page takes 5ms.

### Embedding vs Referencing Decision Framework

```
EMBED when:
├── Data is read together (always need X with Y)
├── Child belongs to exactly one parent (1:1 or 1:few)
├── Updates to child don't need to propagate elsewhere
├── Child data is small
└── Child array won't grow unbounded

REFERENCE when:
├── Data is read independently
├── Child belongs to many parents (many:many)
├── Child changes frequently and must be current everywhere
├── Child data is large
└── Array would grow unbounded (>100 items)
```

### Real Example: OrderFlow Product Catalog

```csharp
// Product document — self-contained with embedded data
public class Product
{
    [BsonId]
    public string Id { get; set; }

    public string Sku { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }

    // Embedded: Price with currency (value object)
    public Money Price { get; set; }

    // Embedded: Always loaded with product
    public List<ProductImage> Images { get; set; } = new();

    // Embedded: Product-specific attributes
    public Dictionary<string, object> Attributes { get; set; } = new();

    // Referenced: Categories exist independently
    public List<string> CategoryIds { get; set; } = new();

    // Embedded: Denormalized category names for display
    public List<string> CategoryNames { get; set; } = new();

    // Embedded: Stock per warehouse
    public List<WarehouseStock> Stock { get; set; } = new();

    // Audit fields
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public int Version { get; set; }
}

public class ProductImage
{
    public string Url { get; set; }
    public string AltText { get; set; }
    public bool IsPrimary { get; set; }
    public int SortOrder { get; set; }
}

public class WarehouseStock
{
    public string WarehouseId { get; set; }
    public string WarehouseName { get; set; }  // Denormalized
    public int Quantity { get; set; }
    public DateTime LastUpdated { get; set; }
}
```

### Handling Schema Variations

```csharp
// Products have different attributes based on type
// Electronics
{
  "_id": "prod-electronics-1",
  "type": "electronics",
  "name": "4K TV",
  "attributes": {
    "screenSize": 55,
    "resolution": "4K",
    "smartTv": true,
    "ports": ["HDMI", "USB", "Ethernet"]
  }
}

// Clothing
{
  "_id": "prod-clothing-1",
  "type": "clothing",
  "name": "Blue T-Shirt",
  "attributes": {
    "sizes": ["S", "M", "L", "XL"],
    "colors": ["Blue", "Black", "White"],
    "material": "100% Cotton"
  }
}

// Handle in C# with dynamic or typed variants
public class Product
{
    public string Type { get; set; }
    public BsonDocument Attributes { get; set; }  // Flexible

    // Or use discriminated types
    [BsonKnownTypes(typeof(ElectronicsAttributes), typeof(ClothingAttributes))]
    public ProductAttributes TypedAttributes { get; set; }
}
```

---

## Core Concept #6 — MongoDB Driver for .NET

### Setup and Configuration

```csharp
// Program.cs
builder.Services.AddSingleton<IMongoClient>(sp =>
{
    var settings = MongoClientSettings.FromConnectionString(
        builder.Configuration.GetConnectionString("MongoDB"));

    settings.ServerApi = new ServerApi(ServerApiVersion.V1);

    // Connection pooling
    settings.MaxConnectionPoolSize = 100;
    settings.MinConnectionPoolSize = 10;

    // Timeouts
    settings.ConnectTimeout = TimeSpan.FromSeconds(10);
    settings.ServerSelectionTimeout = TimeSpan.FromSeconds(10);

    return new MongoClient(settings);
});

builder.Services.AddScoped<IMongoDatabase>(sp =>
{
    var client = sp.GetRequiredService<IMongoClient>();
    return client.GetDatabase("orderflow");
});

// Register typed collections
builder.Services.AddScoped(sp =>
{
    var database = sp.GetRequiredService<IMongoDatabase>();
    return database.GetCollection<Product>("products");
});
```

### CRUD Operations

```csharp
public class ProductRepository
{
    private readonly IMongoCollection<Product> _products;

    public ProductRepository(IMongoCollection<Product> products)
    {
        _products = products;
    }

    // Create
    public async Task CreateAsync(Product product)
    {
        product.CreatedAt = DateTime.UtcNow;
        product.UpdatedAt = DateTime.UtcNow;
        product.Version = 1;

        await _products.InsertOneAsync(product);
    }

    // Read by ID
    public async Task<Product?> GetByIdAsync(string id)
    {
        return await _products
            .Find(p => p.Id == id)
            .FirstOrDefaultAsync();
    }

    // Read with filter
    public async Task<List<Product>> GetByCategoryAsync(string categoryId)
    {
        return await _products
            .Find(p => p.CategoryIds.Contains(categoryId))
            .SortBy(p => p.Name)
            .Limit(100)
            .ToListAsync();
    }

    // Update specific fields
    public async Task<bool> UpdatePriceAsync(string id, Money newPrice)
    {
        var update = Builders<Product>.Update
            .Set(p => p.Price, newPrice)
            .Set(p => p.UpdatedAt, DateTime.UtcNow)
            .Inc(p => p.Version, 1);

        var result = await _products.UpdateOneAsync(
            p => p.Id == id,
            update);

        return result.ModifiedCount > 0;
    }

    // Update with optimistic concurrency
    public async Task<UpdateResult> UpdateAsync(Product product, int expectedVersion)
    {
        product.UpdatedAt = DateTime.UtcNow;
        product.Version = expectedVersion + 1;

        var result = await _products.ReplaceOneAsync(
            p => p.Id == product.Id && p.Version == expectedVersion,
            product);

        if (result.ModifiedCount == 0)
        {
            throw new ConcurrencyException(
                $"Product {product.Id} was modified by another request");
        }

        return result;
    }

    // Delete
    public async Task<bool> DeleteAsync(string id)
    {
        var result = await _products.DeleteOneAsync(p => p.Id == id);
        return result.DeletedCount > 0;
    }
}
```

### Array Operations

```csharp
// Add item to array
public async Task AddImageAsync(string productId, ProductImage image)
{
    var update = Builders<Product>.Update
        .Push(p => p.Images, image)
        .Set(p => p.UpdatedAt, DateTime.UtcNow);

    await _products.UpdateOneAsync(p => p.Id == productId, update);
}

// Update item in array
public async Task UpdateStockAsync(
    string productId,
    string warehouseId,
    int newQuantity)
{
    var filter = Builders<Product>.Filter.And(
        Builders<Product>.Filter.Eq(p => p.Id, productId),
        Builders<Product>.Filter.ElemMatch(
            p => p.Stock,
            s => s.WarehouseId == warehouseId));

    var update = Builders<Product>.Update
        .Set(p => p.Stock[-1].Quantity, newQuantity)
        .Set(p => p.Stock[-1].LastUpdated, DateTime.UtcNow);

    await _products.UpdateOneAsync(filter, update);
}

// Remove item from array
public async Task RemoveImageAsync(string productId, string imageUrl)
{
    var update = Builders<Product>.Update
        .PullFilter(p => p.Images, i => i.Url == imageUrl)
        .Set(p => p.UpdatedAt, DateTime.UtcNow);

    await _products.UpdateOneAsync(p => p.Id == productId, update);
}
```

---

## Core Concept #7 — Indexes and Query Performance

### Why This Matters

Without proper indexes, MongoDB scans entire collections. With millions of documents, queries take minutes instead of milliseconds.

### What Goes Wrong Without This

**Scenario:** Product search queries filter by category, price range, and text. No indexes defined. Query scans 2 million documents. Page takes 15 seconds to load.

### Index Types

```csharp
// Create indexes on startup
public class MongoIndexInitializer : IHostedService
{
    private readonly IMongoCollection<Product> _products;

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // Single field index
        await _products.Indexes.CreateOneAsync(
            new CreateIndexModel<Product>(
                Builders<Product>.IndexKeys.Ascending(p => p.Sku),
                new CreateIndexOptions { Unique = true }));

        // Compound index (order matters!)
        // Supports: { categoryIds, price }, { categoryIds }
        // Does NOT support: { price } alone
        await _products.Indexes.CreateOneAsync(
            new CreateIndexModel<Product>(
                Builders<Product>.IndexKeys
                    .Ascending(p => p.CategoryIds)
                    .Ascending(p => p.Price.Amount)));

        // Text index for search
        await _products.Indexes.CreateOneAsync(
            new CreateIndexModel<Product>(
                Builders<Product>.IndexKeys
                    .Text(p => p.Name)
                    .Text(p => p.Description)));

        // TTL index for automatic deletion
        await _products.Indexes.CreateOneAsync(
            new CreateIndexModel<Product>(
                Builders<Product>.IndexKeys.Ascending("expiresAt"),
                new CreateIndexOptions
                {
                    ExpireAfter = TimeSpan.Zero  // Delete when expiresAt passes
                }));

        // Partial index (only index active products)
        await _products.Indexes.CreateOneAsync(
            new CreateIndexModel<Product>(
                Builders<Product>.IndexKeys.Ascending(p => p.Name),
                new CreateIndexOptions<Product>
                {
                    PartialFilterExpression = Builders<Product>.Filter.Eq(
                        p => p.IsActive, true)
                }));
    }

    public Task StopAsync(CancellationToken cancellationToken)
        => Task.CompletedTask;
}
```

### Explaining Queries

```csharp
public async Task<string> ExplainSearchAsync(string searchTerm)
{
    var filter = Builders<Product>.Filter.Text(searchTerm);

    // Get explain output
    var command = new BsonDocument
    {
        { "explain", new BsonDocument
            {
                { "find", "products" },
                { "filter", filter.Render(
                    BsonSerializer.LookupSerializer<Product>(),
                    BsonSerializer.SerializerRegistry) }
            }
        },
        { "verbosity", "executionStats" }
    };

    var result = await _products.Database.RunCommandAsync<BsonDocument>(command);

    // Key metrics to check:
    // - "stage": Should be "IXSCAN" not "COLLSCAN"
    // - "totalDocsExamined": Should be close to "totalKeysExamined"
    // - "executionTimeMillis": Should be low

    return result.ToJson();
}
```

### Query Optimization Patterns

```csharp
// Use projection to limit returned fields
public async Task<List<ProductSummary>> GetProductSummariesAsync(
    string categoryId,
    int skip,
    int take)
{
    // Only retrieve needed fields
    var projection = Builders<Product>.Projection
        .Include(p => p.Id)
        .Include(p => p.Name)
        .Include(p => p.Price)
        .Include(p => p.Images);

    return await _products
        .Find(p => p.CategoryIds.Contains(categoryId))
        .Project<ProductSummary>(projection)
        .Skip(skip)
        .Limit(take)
        .ToListAsync();
}

// Covered query (answered entirely from index)
// Index: { categoryId: 1, name: 1, price: 1 }
public async Task<List<ProductNameAndPrice>> GetNamesAndPricesAsync(
    string categoryId)
{
    // All queried and projected fields are in the index
    // MongoDB never touches the documents
    var projection = Builders<Product>.Projection
        .Include(p => p.Name)
        .Include(p => p.Price)
        .Exclude(p => p.Id);

    return await _products
        .Find(p => p.CategoryIds.Contains(categoryId))
        .Project<ProductNameAndPrice>(projection)
        .ToListAsync();
}
```

---

## Core Concept #8 — Aggregation Pipelines

### Why This Matters

Aggregation pipelines are MongoDB's power feature for:
- Complex filtering and transformation
- Grouping and statistics
- Joining collections ($lookup)
- Reshaping documents

### Real-World Aggregations

```csharp
// Get category statistics
public async Task<List<CategoryStats>> GetCategoryStatsAsync()
{
    var pipeline = new BsonDocument[]
    {
        // Stage 1: Unwind category array (one doc per category)
        new BsonDocument("$unwind", "$categoryIds"),

        // Stage 2: Group by category
        new BsonDocument("$group", new BsonDocument
        {
            { "_id", "$categoryIds" },
            { "productCount", new BsonDocument("$sum", 1) },
            { "avgPrice", new BsonDocument("$avg", "$price.amount") },
            { "minPrice", new BsonDocument("$min", "$price.amount") },
            { "maxPrice", new BsonDocument("$max", "$price.amount") }
        }),

        // Stage 3: Lookup category details
        new BsonDocument("$lookup", new BsonDocument
        {
            { "from", "categories" },
            { "localField", "_id" },
            { "foreignField", "_id" },
            { "as", "category" }
        }),

        // Stage 4: Reshape output
        new BsonDocument("$project", new BsonDocument
        {
            { "categoryId", "$_id" },
            { "categoryName", new BsonDocument("$arrayElemAt",
                new BsonArray { "$category.name", 0 }) },
            { "productCount", 1 },
            { "avgPrice", new BsonDocument("$round",
                new BsonArray { "$avgPrice", 2 }) },
            { "priceRange", new BsonDocument
            {
                { "min", "$minPrice" },
                { "max", "$maxPrice" }
            }}
        }),

        // Stage 5: Sort by product count
        new BsonDocument("$sort", new BsonDocument("productCount", -1))
    };

    return await _products.Aggregate<CategoryStats>(pipeline).ToListAsync();
}

// Using strongly-typed aggregation
public async Task<List<TopSellingProduct>> GetTopSellingProductsAsync(
    DateTime from,
    DateTime to,
    int limit = 10)
{
    // Aggregate orders to find top products
    var result = await _orders.Aggregate()
        .Match(o => o.CreatedAt >= from && o.CreatedAt <= to)
        .Unwind<Order, OrderItem>(o => o.Items)
        .Group(
            key: item => item.ProductId,
            group: g => new
            {
                ProductId = g.Key,
                TotalQuantity = g.Sum(x => x.Quantity),
                TotalRevenue = g.Sum(x => x.Price * x.Quantity),
                OrderCount = g.Count()
            })
        .SortByDescending(x => x.TotalRevenue)
        .Limit(limit)
        .Lookup<Order, Product, TopSellingProduct>(
            foreignCollectionName: "products",
            localField: "_id",
            foreignField: "_id",
            @as: x => x.Product)
        .ToListAsync();

    return result;
}
```

### Search with Facets

```csharp
// Product search with faceted navigation
public async Task<SearchResult> SearchProductsAsync(
    string query,
    Dictionary<string, List<string>> filters,
    int page,
    int pageSize)
{
    var matchStage = new BsonDocument("$match",
        new BsonDocument("$text", new BsonDocument("$search", query)));

    var facetStage = new BsonDocument("$facet", new BsonDocument
    {
        // Facet 1: Results
        { "results", new BsonArray
            {
                new BsonDocument("$skip", (page - 1) * pageSize),
                new BsonDocument("$limit", pageSize),
                new BsonDocument("$project", new BsonDocument
                {
                    { "id", "$_id" },
                    { "name", 1 },
                    { "price", 1 },
                    { "image", new BsonDocument("$arrayElemAt",
                        new BsonArray { "$images", 0 }) },
                    { "score", new BsonDocument("$meta", "textScore") }
                })
            }
        },
        // Facet 2: Total count
        { "totalCount", new BsonArray
            {
                new BsonDocument("$count", "count")
            }
        },
        // Facet 3: Price ranges
        { "priceRanges", new BsonArray
            {
                new BsonDocument("$bucket", new BsonDocument
                {
                    { "groupBy", "$price.amount" },
                    { "boundaries", new BsonArray { 0, 25, 50, 100, 250, 500, 1000 } },
                    { "default", "1000+" },
                    { "output", new BsonDocument("count", new BsonDocument("$sum", 1)) }
                })
            }
        },
        // Facet 4: Categories
        { "categories", new BsonArray
            {
                new BsonDocument("$unwind", "$categoryIds"),
                new BsonDocument("$group", new BsonDocument
                {
                    { "_id", "$categoryIds" },
                    { "count", new BsonDocument("$sum", 1) }
                }),
                new BsonDocument("$sort", new BsonDocument("count", -1)),
                new BsonDocument("$limit", 10)
            }
        }
    });

    var pipeline = new[] { matchStage, facetStage };
    var result = await _products.Aggregate<BsonDocument>(pipeline).FirstOrDefaultAsync();

    return MapToSearchResult(result);
}
```

---

## Core Concept #9 — Transactions and Consistency

### Why This Matters

MongoDB supports multi-document transactions, but they come with overhead. Understanding when to use them vs when to rely on single-document atomicity is important.

### Single-Document Atomicity

```csharp
// This is atomic — no transaction needed
public async Task AddItemToCartAsync(string cartId, CartItem item)
{
    // findOneAndUpdate is atomic
    var filter = Builders<Cart>.Filter.Eq(c => c.Id, cartId);
    var update = Builders<Cart>.Update
        .Push(c => c.Items, item)
        .Set(c => c.UpdatedAt, DateTime.UtcNow)
        .Inc(c => c.TotalItems, 1);

    var options = new FindOneAndUpdateOptions<Cart>
    {
        ReturnDocument = ReturnDocument.After
    };

    var cart = await _carts.FindOneAndUpdateAsync(filter, update, options);
}
```

### Multi-Document Transactions

```csharp
// When you need to update multiple documents atomically
public async Task TransferStockAsync(
    string fromWarehouse,
    string toWarehouse,
    string productId,
    int quantity)
{
    using var session = await _client.StartSessionAsync();

    session.StartTransaction(new TransactionOptions(
        readConcern: ReadConcern.Snapshot,
        writeConcern: WriteConcern.WMajority));

    try
    {
        // Decrease source warehouse
        var decreaseResult = await _products.UpdateOneAsync(
            session,
            p => p.Id == productId &&
                 p.Stock.Any(s => s.WarehouseId == fromWarehouse && s.Quantity >= quantity),
            Builders<Product>.Update.Inc("stock.$.quantity", -quantity));

        if (decreaseResult.ModifiedCount == 0)
        {
            throw new InsufficientStockException(
                $"Not enough stock in warehouse {fromWarehouse}");
        }

        // Increase destination warehouse
        await _products.UpdateOneAsync(
            session,
            p => p.Id == productId &&
                 p.Stock.Any(s => s.WarehouseId == toWarehouse),
            Builders<Product>.Update.Inc("stock.$.quantity", quantity));

        // Log the transfer
        await _stockTransfers.InsertOneAsync(session, new StockTransfer
        {
            ProductId = productId,
            FromWarehouse = fromWarehouse,
            ToWarehouse = toWarehouse,
            Quantity = quantity,
            TransferredAt = DateTime.UtcNow
        });

        await session.CommitTransactionAsync();
    }
    catch
    {
        await session.AbortTransactionAsync();
        throw;
    }
}
```

### Transaction Best Practices

```csharp
// Retry logic for transient failures
public async Task<T> WithTransactionAsync<T>(
    Func<IClientSessionHandle, Task<T>> operation)
{
    using var session = await _client.StartSessionAsync();

    var options = new TransactionOptions(
        readConcern: ReadConcern.Snapshot,
        writeConcern: WriteConcern.WMajority,
        maxCommitTime: TimeSpan.FromSeconds(30));

    return await session.WithTransactionAsync(
        async (s, ct) => await operation(s),
        options);
}

// Usage
var result = await WithTransactionAsync(async session =>
{
    // All operations in transaction
    await _products.UpdateOneAsync(session, ...);
    await _orders.InsertOneAsync(session, ...);
    return new { Success = true };
});
```

---

## Hands-On: OrderFlow Product Catalog

### Task 1 — Design Product Document

Create the product document schema with:
- Embedded images and stock levels
- Flexible attributes for different product types
- Denormalized category names

### Task 2 — Implement Repository

Build a `ProductRepository` with:
- CRUD operations
- Optimistic concurrency
- Array manipulation methods

### Task 3 — Create Indexes

Define indexes for:
- SKU uniqueness
- Category + price compound
- Full-text search on name/description

### Task 4 — Build Aggregations

Implement:
- Category statistics (product count, avg price)
- Top selling products
- Search with facets

### Task 5 — Integration Tests

Use Testcontainers to test:
- Index performance
- Transaction behavior
- Aggregation accuracy

---

## Deliverables

1. **Product document schema** with rationale for embedding decisions
2. **Repository implementation** with CRUD and array operations
3. **Index definitions** with explain() output proving usage
4. **Aggregation pipelines** for analytics
5. **Integration tests** using Testcontainers

---

## Resources

### Must-Read
- [MongoDB .NET Driver Documentation](https://www.mongodb.com/docs/drivers/csharp/current/)
- [MongoDB Data Modeling](https://www.mongodb.com/docs/manual/data-modeling/)
- [MongoDB Aggregation](https://www.mongodb.com/docs/manual/aggregation/)

### Videos
- [MongoDB University: M001 Basics](https://university.mongodb.com/courses/M001)
- [MongoDB University: M121 Aggregation](https://university.mongodb.com/courses/M121)
- [Nick Chapsas: MongoDB in .NET](https://www.youtube.com/watch?v=5ER4_0b5n-A)

### Tools
- [MongoDB Compass](https://www.mongodb.com/products/compass) — Visual query tool
- [Mongo Express](https://github.com/mongo-express/mongo-express) — Web-based admin
- [Testcontainers.MongoDB](https://www.nuget.org/packages/Testcontainers.MongoDb)

---

## Reflection Questions

1. When would you choose referencing over embedding?
2. How do you handle schema evolution in MongoDB?
3. What's the trade-off between more indexes and write performance?
4. When are transactions worth the overhead?

---

**Next:** [Part 3 — Redis Caching & Distributed Primitives](./part-3.md)
