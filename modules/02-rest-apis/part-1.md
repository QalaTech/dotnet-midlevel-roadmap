# 02 — HTTP & REST Fundamentals

## Part 1 / 4 — HTTP Verbs, Resources, and API Design Principles

---

## Why This Module Exists

Every backend developer "knows" REST. Few understand it deeply.

The difference shows up in:
- APIs that break when requirements change
- Status codes that confuse frontend teams
- Endpoints that don't compose well
- Designs that require awkward workarounds

This module teaches REST as a **design discipline**, not just a set of conventions.

---

## What You'll Build (OrderFlow)

In this module, you'll design the **API contract** for OrderFlow. No code yet — just documentation:
- Resource hierarchy (orders, products, customers, line items)
- Endpoint design for every operation
- Request/response shapes
- Error catalog
- Versioning strategy

This API spec becomes the blueprint for Module 03.

---

## Core Concept #1 — Resources, Not Actions

### Why This Matters

REST is about **resources** (nouns), not **actions** (verbs). HTTP already has verbs (GET, POST, PUT, DELETE). Your URLs shouldn't repeat them.

### What Goes Wrong Without This

**Real scenario:** An API has these endpoints:
```
POST /createOrder
POST /updateOrder
POST /deleteOrder
POST /getOrderById
```

This is RPC disguised as HTTP. Problems:
- No caching (everything is POST)
- No standard tooling support
- Endpoints multiply as features grow
- No clear resource model

**Real scenario:** An API mixes resources with actions:
```
GET /orders                 (good)
POST /orders                (good)
POST /orders/123/process    (action - what does this mean?)
POST /orders/123/approve    (action)
POST /orders/123/reject     (action)
```

Now every business action becomes an endpoint. The API grows uncontrollably.

### The RESTful Way

Resources are nouns. Operations use HTTP verbs:

```
GET    /orders           → List orders
POST   /orders           → Create order
GET    /orders/123       → Get order
PUT    /orders/123       → Replace order
PATCH  /orders/123       → Partially update order
DELETE /orders/123       → Delete order
```

For complex operations, model them as resources:

```
# Instead of POST /orders/123/cancel
POST /orders/123/cancellations
{
  "reason": "Customer request",
  "refundRequested": true
}

# Now cancellations are queryable
GET /orders/123/cancellations
```

---

## Core Concept #2 — HTTP Verbs (The Real Rules)

### Why This Matters

Each HTTP verb has specific semantics. Using them correctly enables:
- Caching (GET is cacheable)
- Retry safety (GET, PUT, DELETE are idempotent)
- Browser/client behavior
- Proxy handling

### What Goes Wrong Without This

**Real scenario:** A developer uses POST for everything "because it's safer." Result:
- No browser caching
- No GET request bookmarks
- Clients can't safely retry failed requests
- Load balancers can't cache responses

**Real scenario:** PUT is used for partial updates. Client sends `{ "name": "New Name" }`. Server replaces entire resource, wiping out all other fields.

### The Verbs Explained

#### GET — Retrieve

```http
GET /orders/123
```

- **Idempotent:** Yes (same result every time)
- **Safe:** Yes (no side effects)
- **Cacheable:** Yes
- **Request body:** No (technically allowed but never used)
- **Response:** The resource

#### POST — Create (or Trigger Action)

```http
POST /orders
Content-Type: application/json

{
  "customerId": "abc-123",
  "items": [
    { "productId": "prod-456", "quantity": 2 }
  ]
}
```

- **Idempotent:** No (calling twice creates two resources)
- **Safe:** No
- **Cacheable:** No
- **Request body:** Yes
- **Response:** 201 Created + Location header

#### PUT — Replace Entire Resource

```http
PUT /orders/123
Content-Type: application/json

{
  "customerId": "abc-123",
  "items": [...],
  "shippingAddress": {...},
  "billingAddress": {...}
}
```

- **Idempotent:** Yes (same result whether called once or 100 times)
- **Safe:** No
- **Important:** Replaces the ENTIRE resource. Missing fields become null.

#### PATCH — Partial Update

```http
PATCH /orders/123
Content-Type: application/json

{
  "shippingAddress": {
    "city": "London"
  }
}
```

- **Idempotent:** Should be (design it that way)
- **Safe:** No
- **Important:** Updates only the fields provided

#### DELETE — Remove Resource

```http
DELETE /orders/123
```

- **Idempotent:** Yes (deleting twice = same result)
- **Safe:** No
- **Response:** 204 No Content (usually)

### Idempotency Matters

Idempotent operations can be safely retried. This is critical for:
- Network failures (did my request succeed?)
- Distributed systems (at-least-once delivery)
- Client retry logic

```
# Safe to retry
GET /orders/123          → Always returns same order
PUT /orders/123 {...}    → Replaces with same data
DELETE /orders/123       → Already deleted = still deleted

# NOT safe to retry
POST /orders {...}       → Creates duplicate order!
```

---

## Core Concept #3 — URL Design

### Why This Matters

Good URLs are:
- Predictable (developers can guess them)
- Hierarchical (relationships are clear)
- Consistent (same patterns everywhere)
- Stable (don't change when implementation changes)

### What Goes Wrong Without This

**Real scenario:** An API evolves organically:
```
GET /getBooks
GET /api/v1/fetch-authors
GET /BookReviews
GET /ORDERS
POST /order_create
```

Frontend developers spend hours reading docs for every endpoint. New team members take weeks to learn the API. Bugs happen because patterns aren't consistent.

### URL Design Rules

**1. Use plural nouns for collections**
```
GET /orders          (not /order)
GET /products        (not /product)
GET /customers       (not /customer)
```

**2. Use IDs in path for single resources**
```
GET /orders/123
GET /products/abc-456
GET /customers/cust_789
```

**3. Use nesting for relationships**
```
GET /orders/123/items           → Items in order 123
GET /customers/456/orders       → Orders for customer 456
GET /products/789/reviews       → Reviews for product 789
```

**4. Limit nesting depth (max 2-3 levels)**
```
# Good
GET /orders/123/items/456

# Too deep — hard to understand
GET /customers/1/orders/2/items/3/adjustments/4
```

**5. Use query parameters for filtering, not paths**
```
# Good
GET /orders?status=pending&customerId=123

# Bad — these aren't different resources
GET /orders/pending
GET /orders/customer/123
```

---

## Core Concept #4 — Request/Response Design

### Why This Matters

Clear request/response shapes:
- Reduce integration errors
- Enable code generation
- Make APIs self-documenting
- Allow breaking change detection

### What Goes Wrong Without This

**Real scenario:** An API returns different shapes for the same resource:
```json
// GET /orders/123
{ "id": 123, "customer_id": "abc", "orderTotal": 99.99 }

// GET /orders
{ "orders": [{ "orderId": 123, "customerId": "abc", "total": 99.99 }] }

// POST /orders response
{ "order_id": 123, "cust": "abc", "amount": 99.99 }
```

Same data, three different field names. Frontend developers write three different parsers.

### Design Principles

**1. Consistent naming (pick one style)**
```json
// camelCase (recommended for JSON)
{
  "orderId": "ord-123",
  "customerId": "cust-456",
  "createdAt": "2024-01-15T10:30:00Z",
  "lineItems": [...]
}
```

**2. Wrap collections**
```json
// Good — extensible
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 156
  }
}

// Bad — can't add metadata later
[...]
```

**3. Use ISO 8601 for dates**
```json
{
  "createdAt": "2024-01-15T10:30:00Z",      // UTC
  "scheduledFor": "2024-01-20T14:00:00+01:00" // With timezone
}

// Never: "01/15/2024" or "15-Jan-2024" or Unix timestamps
```

**4. Use consistent ID formats**
```json
// Pick one style and stick to it
{ "id": "ord_abc123" }       // Prefixed (good for debugging)
{ "id": "550e8400-e29b-41d4-a716-446655440000" } // UUID
{ "id": 12345 }              // Integer (avoid — harder to shard)
```

---

## Core Concept #5 — Resource Relationships

### Why This Matters

Resources relate to each other. How you model these relationships affects:
- Query efficiency (N+1 problems)
- API complexity
- Client-side state management

### Relationship Patterns

**1. Embedded (for small, always-needed data)**
```json
GET /orders/123

{
  "id": "ord-123",
  "customer": {
    "id": "cust-456",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "items": [
    { "productId": "prod-789", "productName": "Widget", "quantity": 2 }
  ]
}
```

**2. Referenced (for large or optional data)**
```json
GET /orders/123

{
  "id": "ord-123",
  "customerId": "cust-456",
  "itemIds": ["item-1", "item-2"]
}

// Client fetches related resources separately
GET /customers/cust-456
GET /orders/123/items
```

**3. Expandable (client chooses)**
```json
GET /orders/123?expand=customer,items

{
  "id": "ord-123",
  "customerId": "cust-456",
  "customer": { ... },  // Expanded
  "items": [ ... ]      // Expanded
}
```

### Anti-Patterns

```json
// BAD: Exposing internal IDs
{
  "id": "ord-123",
  "_internalCustomerRowId": 54321,  // Never expose database IDs
  "postgresTimestamp": 1705312200   // Never expose DB internals
}

// BAD: Inconsistent relationships
{
  "customer": { "id": "cust-1", "name": "..." },  // Embedded
  "shippingAddressId": "addr-2"                   // Referenced
  // Pick one approach
}
```

---

## Hands-On: OrderFlow API Design

Design the resource model for OrderFlow. Create documentation (not code) for:

### Resources

| Resource | Description |
|----------|-------------|
| `/customers` | Business customers placing orders |
| `/products` | Items available for purchase |
| `/orders` | Customer orders with line items |
| `/orders/{id}/items` | Line items within an order |

### For Each Endpoint, Document

1. **HTTP verb and URL**
2. **Request body** (if applicable)
3. **Success response** with status code
4. **Possible error responses**
5. **Query parameters** (for GET collections)

### Example Format

```markdown
## Create Order

POST /orders

### Request
```json
{
  "customerId": "cust-123",
  "items": [
    { "productId": "prod-456", "quantity": 2 }
  ],
  "shippingAddress": {
    "line1": "123 Main St",
    "city": "London",
    "postalCode": "SW1A 1AA",
    "country": "GB"
  }
}
```

### Response (201 Created)
Location: /orders/ord-789

```json
{
  "id": "ord-789",
  "status": "pending",
  "customerId": "cust-123",
  ...
}
```

### Errors
- 400: Invalid request (missing required fields)
- 404: Customer or product not found
- 409: Insufficient stock
```

---

## Deliverables

1. **OrderFlow API Design Document** with:
   - Resource hierarchy diagram
   - All endpoints for CRUD operations
   - Request/response examples
   - Error catalog

2. **API Style Guide** documenting:
   - Naming conventions
   - Date formats
   - ID formats
   - Collection wrapping

3. **Postman/Bruno Collection** (optional but valuable):
   - Example requests for each endpoint
   - Environment variables

---

## Resources

### Must-Read
- [RFC 7231 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc7231) — The actual HTTP spec (read sections 4-6)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) — Battle-tested patterns

### Videos
- [Nick Chapsas: Clean REST APIs](https://www.youtube.com/watch?v=eRJF0zzEikM)
- [CodeOpinion: REST API Design](https://www.youtube.com/watch?v=P0a7PwRNLVU)

### Tools
- [Stoplight Studio](https://stoplight.io/studio) — OpenAPI design tool
- [Bruno](https://www.usebruno.com/) — Open-source API client (Postman alternative)

### Example APIs to Study
- [Stripe API](https://stripe.com/docs/api) — Gold standard for API design
- [GitHub REST API](https://docs.github.com/en/rest) — Good resource modeling

---

## Reflection Questions

1. When would you use PUT vs PATCH?
2. How do you decide whether to embed or reference a related resource?
3. What makes an API "predictable"?
4. Why does idempotency matter for distributed systems?

---

**Next:** [Part 2 — Status Codes, Errors, Pagination, and Filtering](./part-2.md)
