# 02 — HTTP & REST Fundamentals

## Part 3 / 4 — Pagination, Filtering, Sorting, and Caching

---

## Why This Matters

Real APIs don't return 10 items. They return 100,000. Without proper pagination, filtering, and caching:
- Clients crash trying to load huge responses
- Servers fall over from expensive queries
- Users wait forever
- Networks saturate

These patterns make APIs usable at scale.

---

## Core Concept #12 — Pagination

### Why You Need It

**What goes wrong:** An endpoint returns all orders. On day one, that's 50 orders. Six months later, it's 500,000. The mobile app crashes. The server times out. Everyone blames "the API is slow."

The API was never designed for real usage.

### Offset Pagination

The simple approach: skip N items, take M items.

```http
GET /orders?page=3&pageSize=20
```

```json
{
  "data": [...],
  "pagination": {
    "page": 3,
    "pageSize": 20,
    "totalCount": 1543,
    "totalPages": 78
  }
}
```

**Pros:**
- Simple to implement
- Easy for clients to understand
- Can jump to any page

**Cons:**
- Expensive at large offsets (OFFSET 50000 scans 50000 rows)
- Inconsistent with concurrent writes (items shift between pages)

**When to use:** Admin interfaces, small datasets, reports.

### Cursor Pagination

Uses a pointer (cursor) to the last item seen.

```http
GET /orders?cursor=ord_abc123&limit=20
```

```json
{
  "data": [...],
  "pagination": {
    "hasMore": true,
    "nextCursor": "ord_xyz789"
  }
}
```

**Pros:**
- Consistent performance regardless of position
- Stable with concurrent writes
- Scales to millions of items

**Cons:**
- Can't jump to arbitrary page
- More complex to implement

**When to use:** Public APIs, mobile apps, large datasets, real-time data.

### Cursor Implementation

```csharp
// Cursor is typically the ID of the last item
public async Task<PagedResult<Order>> GetOrdersAsync(string? cursor, int limit)
{
    var query = _context.Orders.OrderByDescending(o => o.CreatedAt);

    if (cursor is not null)
    {
        var cursorOrder = await _context.Orders.FindAsync(Guid.Parse(cursor));
        if (cursorOrder is not null)
        {
            query = query.Where(o => o.CreatedAt < cursorOrder.CreatedAt);
        }
    }

    var items = await query.Take(limit + 1).ToListAsync();
    var hasMore = items.Count > limit;

    return new PagedResult<Order>
    {
        Data = items.Take(limit).ToList(),
        HasMore = hasMore,
        NextCursor = hasMore ? items[limit - 1].Id.ToString() : null
    };
}
```

### Keyset Pagination

Cursor pagination using composite keys for stable ordering.

```http
GET /orders?after_created=2024-01-15T10:00:00Z&after_id=ord_abc123&limit=20
```

This handles the case where multiple items have the same timestamp.

---

## Core Concept #13 — Filtering

### Why This Matters

**What goes wrong:** A client needs orders from one customer. The API only supports "get all orders." The client fetches all 500,000 orders and filters in memory. Memory exhaustion, slow UI, angry users.

### Basic Filtering

Query parameters for simple filters:

```http
GET /orders?status=pending
GET /orders?customerId=cust-123
GET /orders?status=pending&customerId=cust-123  # AND
```

### Range Filters

For dates, amounts, quantities:

```http
GET /orders?createdAfter=2024-01-01&createdBefore=2024-02-01
GET /orders?minTotal=100&maxTotal=500
```

Naming conventions:
- `minX` / `maxX` — inclusive range
- `afterX` / `beforeX` — for dates
- `fromX` / `toX` — alternative

### Multiple Values

For OR conditions:

```http
GET /orders?status=pending,processing,shipped
# OR
GET /orders?status=pending&status=processing&status=shipped
```

Pick one approach and document it.

### Full-Text Search

Separate from filters — usually different implementation:

```http
GET /products?q=widget
GET /products?search=wireless%20keyboard
```

### Complex Filtering (When Needed)

Some APIs need complex queries. Options:

```http
# OData style
GET /orders?$filter=total gt 100 and status eq 'pending'

# GraphQL (separate approach entirely)
POST /graphql
{ orders(filter: { total: { gt: 100 }, status: "pending" }) { ... } }

# Custom operators
GET /orders?filter[total][gt]=100&filter[status][eq]=pending
```

**Recommendation:** Start simple. Add complexity only when needed.

---

## Core Concept #14 — Sorting

### Why This Matters

**What goes wrong:** An API returns orders but the sort order is undefined. Client A sees orders by creation date. Client B, after a database optimization, sees them alphabetically. Reports break. UIs show wrong data.

### Basic Sorting

```http
GET /orders?sort=createdAt
GET /orders?sort=-createdAt          # Descending (prefix with -)
GET /orders?sortBy=createdAt&order=desc  # Alternative
```

### Multi-Field Sorting

```http
GET /orders?sort=status,-createdAt
# Sort by status ascending, then by createdAt descending
```

### Allowed Sort Fields

Don't expose all fields. Document which fields are sortable:

```json
{
  "allowedSortFields": ["createdAt", "total", "status", "customerName"]
}
```

### Stable Sorting

Always include a unique field as tiebreaker (usually ID):

```csharp
var query = _context.Orders
    .OrderBy(o => o.Status)
    .ThenByDescending(o => o.CreatedAt)
    .ThenBy(o => o.Id);  // Tiebreaker for stable pagination
```

---

## Core Concept #15 — Response Shaping

### Field Selection

Let clients request only the fields they need:

```http
GET /orders?fields=id,status,total
GET /orders?fields=id,customer.name,items.productName
```

**Pros:**
- Reduces payload size
- Improves mobile performance

**Cons:**
- More complex to implement
- Can conflict with caching

### Expansion/Embedding

Let clients include related resources:

```http
GET /orders/ord-123?expand=customer,items
GET /orders/ord-123?include=customer,items.product
```

```json
{
  "id": "ord-123",
  "customerId": "cust-456",
  "customer": {           // Expanded
    "id": "cust-456",
    "name": "Acme Corp"
  },
  "items": [              // Expanded
    {
      "productId": "prod-789",
      "product": {        // Nested expansion
        "id": "prod-789",
        "name": "Widget"
      }
    }
  ]
}
```

---

## Core Concept #16 — HTTP Caching

### Why This Matters

Caching is free performance:
- Reduces server load
- Improves response times
- Saves bandwidth

**What goes wrong:** No caching headers. Every request hits the database. Load balancers can't help. CDNs are useless. Users wait.

### Cache-Control Header

```http
# Cache for 1 hour, private to the user
Cache-Control: private, max-age=3600

# Cache for 1 day, can be shared (CDN)
Cache-Control: public, max-age=86400

# Never cache (user-specific, real-time data)
Cache-Control: no-store
```

| Directive | Meaning |
|-----------|---------|
| `public` | Can be cached by CDN/proxies |
| `private` | Only browser can cache (user-specific data) |
| `max-age=N` | Fresh for N seconds |
| `no-store` | Never cache |
| `no-cache` | Cache but always revalidate |

### ETags (Entity Tags)

ETags let clients check if a resource changed:

```http
GET /products/prod-123
HTTP/1.1 200 OK
ETag: "abc123"

{...}
```

Later:
```http
GET /products/prod-123
If-None-Match: "abc123"

HTTP/1.1 304 Not Modified
(no body - client uses cached version)
```

### Last-Modified

Similar to ETags, using timestamps:

```http
GET /products/prod-123
HTTP/1.1 200 OK
Last-Modified: Wed, 15 Jan 2024 10:30:00 GMT

{...}
```

Later:
```http
GET /products/prod-123
If-Modified-Since: Wed, 15 Jan 2024 10:30:00 GMT

HTTP/1.1 304 Not Modified
```

### What to Cache

| Resource | Cache Strategy |
|----------|----------------|
| Product catalog | `public, max-age=3600` + ETag |
| User profile | `private, max-age=300` |
| Order list | `private, no-cache` (always revalidate) |
| Current stock | `no-store` (real-time) |
| Static content | `public, max-age=86400` + immutable |

---

## Hands-On: OrderFlow Query API

Design the query interface for OrderFlow:

### Orders Endpoint

```http
# List orders with pagination and filters
GET /orders
  ?page=1
  &pageSize=20
  &status=pending,processing
  &customerId=cust-123
  &createdAfter=2024-01-01
  &sort=-createdAt
  &expand=customer

# Response
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 45,
    "totalPages": 3
  }
}
```

### Products Endpoint

```http
# List products with cursor pagination
GET /products
  ?cursor=prod_abc123
  &limit=50
  &category=electronics
  &inStock=true
  &q=wireless
  &sort=name

# Response
{
  "data": [...],
  "pagination": {
    "hasMore": true,
    "nextCursor": "prod_xyz789"
  }
}
```

---

## Deliverables

1. **Pagination strategy document** — Which style for which endpoints
2. **Filter specification** — All supported filters with examples
3. **Sort specification** — Allowed sort fields, default orders
4. **Caching strategy** — Cache-Control for each endpoint type
5. **OpenAPI spec** — Document all query parameters

---

## Resources

### Must-Read
- [Stripe API Pagination](https://stripe.com/docs/api/pagination) — Excellent cursor example
- [JSON:API Specification](https://jsonapi.org/) — Standardized filtering/pagination

### Videos
- [Hussein Nasser: Pagination Deep Dive](https://www.youtube.com/watch?v=WUICbOOtAic)
- [Nick Chapsas: Caching in ASP.NET Core](https://www.youtube.com/watch?v=0WvGwOoK-CI)

### Documentation
- [MDN: HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Microsoft: Response Caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/response)

---

## Reflection Questions

1. When would you choose cursor over offset pagination?
2. How do filters interact with authorization (customer seeing only their orders)?
3. What's the tradeoff between flexible filtering and query performance?
4. When should you NOT cache an endpoint?

---

**Next:** [Part 4 — Versioning, Contracts, and Long-Term API Design](./part-4.md)
