# 02 — HTTP & REST Fundamentals

## Part 2 / 4 — Status Codes, Error Handling, and ProblemDetails

---

## Why This Matters

Status codes are the API's way of telling clients what happened. Wrong status codes cause:
- Broken error handling in clients
- Incorrect retry behavior
- Confused developers
- Silent failures

Getting this right is one of the clearest signals of API maturity.

---

## Core Concept #6 — Status Code Categories

### The Five Families

| Range | Meaning | Examples |
|-------|---------|----------|
| 1xx | Informational | 100 Continue, 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 404 Not Found |
| 5xx | Server Error | 500 Internal Error, 503 Unavailable |

### What Goes Wrong Without This

**Real scenario:** An API returns 200 for everything, with error details in the body:
```json
{
  "success": false,
  "error": "Order not found"
}
```

Problems:
- HTTP clients think the request succeeded
- Caching proxies cache the "successful" error
- Monitoring shows 100% success rate while users complain
- Retry logic doesn't trigger

---

## Core Concept #7 — Success Codes (2xx)

### 200 OK

The default success response. Use for:
- Successful GET requests
- Successful PUT/PATCH that returns the updated resource

```http
GET /orders/123
HTTP/1.1 200 OK

{
  "id": "ord-123",
  "status": "pending"
}
```

### 201 Created

A new resource was created. **Must include Location header.**

```http
POST /orders
HTTP/1.1 201 Created
Location: /orders/ord-789

{
  "id": "ord-789",
  "status": "pending"
}
```

**What goes wrong:** Returning 200 for POST. Clients don't know a new resource exists. They don't get the Location header for the new resource.

### 202 Accepted

Request accepted for processing, but not complete yet. Use for async operations.

```http
POST /orders/ord-123/shipments
HTTP/1.1 202 Accepted
Location: /jobs/job-456

{
  "jobId": "job-456",
  "status": "pending",
  "estimatedCompletion": "2024-01-15T12:00:00Z"
}
```

### 204 No Content

Success with no response body. Common for:
- DELETE
- PUT/PATCH when not returning the updated resource

```http
DELETE /orders/ord-123
HTTP/1.1 204 No Content
```

**What goes wrong:** Returning 200 with empty body. Some clients fail parsing the empty response.

---

## Core Concept #8 — Client Error Codes (4xx)

### 400 Bad Request

The request is malformed. Use for:
- Invalid JSON
- Missing required fields
- Wrong data types
- Values outside allowed ranges

```http
POST /orders
{
  "customerId": null,  // Required but null
  "quantity": -5       // Must be positive
}

HTTP/1.1 400 Bad Request
{
  "type": "https://api.orderflow.com/errors/validation",
  "title": "Validation failed",
  "status": 400,
  "errors": {
    "customerId": ["Customer ID is required"],
    "quantity": ["Quantity must be greater than 0"]
  }
}
```

### 401 Unauthorized

The client is not authenticated. No valid credentials provided.

```http
GET /orders
Authorization: Bearer invalid-token

HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="orderflow"

{
  "type": "https://api.orderflow.com/errors/authentication",
  "title": "Authentication required",
  "status": 401,
  "detail": "The access token is expired or invalid"
}
```

**What goes wrong:** Confusing 401 with 403. 401 means "who are you?" 403 means "I know who you are, but you can't do this."

### 403 Forbidden

Authenticated but not authorized for this action.

```http
DELETE /orders/ord-123
Authorization: Bearer valid-but-not-admin-token

HTTP/1.1 403 Forbidden
{
  "type": "https://api.orderflow.com/errors/authorization",
  "title": "Not authorized",
  "status": 403,
  "detail": "Admin role required to delete orders"
}
```

### 404 Not Found

The resource doesn't exist.

```http
GET /orders/ord-nonexistent

HTTP/1.1 404 Not Found
{
  "type": "https://api.orderflow.com/errors/not-found",
  "title": "Order not found",
  "status": 404,
  "detail": "No order exists with ID 'ord-nonexistent'"
}
```

**What goes wrong:** Returning 400 for not found. 400 means the request is invalid. 404 means the request is valid but the resource doesn't exist.

### 409 Conflict

The request conflicts with the current state of the resource. **Use for business rule violations.**

```http
POST /orders/ord-123/items
{
  "productId": "prod-out-of-stock",
  "quantity": 10
}

HTTP/1.1 409 Conflict
{
  "type": "https://api.orderflow.com/errors/insufficient-stock",
  "title": "Insufficient stock",
  "status": 409,
  "detail": "Product 'prod-out-of-stock' has only 3 units available",
  "productId": "prod-out-of-stock",
  "requested": 10,
  "available": 3
}
```

Other 409 examples:
- Order already cancelled
- User already exists
- Booking already confirmed
- Version conflict (optimistic locking)

### 422 Unprocessable Entity

The request is syntactically valid but semantically wrong. Some teams use this instead of 400 for validation errors.

```http
POST /orders
{
  "customerId": "cust-123",
  "email": "not-an-email"  // Syntactically JSON, semantically wrong
}

HTTP/1.1 422 Unprocessable Entity
```

**Team decision:** Pick either 400 or 422 for validation and be consistent.

### 429 Too Many Requests

Rate limit exceeded.

```http
GET /orders

HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705312260

{
  "type": "https://api.orderflow.com/errors/rate-limit",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "You have exceeded 100 requests per minute. Try again in 30 seconds."
}
```

---

## Core Concept #9 — Server Error Codes (5xx)

### 500 Internal Server Error

Something unexpected failed. The client did nothing wrong.

```http
GET /orders/ord-123

HTTP/1.1 500 Internal Server Error
{
  "type": "https://api.orderflow.com/errors/internal",
  "title": "Internal server error",
  "status": 500,
  "detail": "An unexpected error occurred. Please try again later.",
  "traceId": "abc123-xyz789"
}
```

**Important:**
- Never leak stack traces or internal details
- Always include a trace/correlation ID for debugging
- Log the full error server-side

### 502 Bad Gateway

Your server is a proxy and the upstream failed.

### 503 Service Unavailable

Server is temporarily down (maintenance, overload).

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 300

{
  "type": "https://api.orderflow.com/errors/maintenance",
  "title": "Service unavailable",
  "status": 503,
  "detail": "The service is undergoing maintenance. Expected completion: 14:30 UTC."
}
```

### 504 Gateway Timeout

Upstream server didn't respond in time.

---

## Core Concept #10 — ProblemDetails (RFC 7807)

### Why This Matters

Without a standard error format, every API invents its own:
```json
{ "error": "Not found" }
{ "message": "Not found", "code": 404 }
{ "errors": ["Not found"] }
{ "success": false, "reason": "Not found" }
```

Clients need different code for each API. ProblemDetails fixes this.

### The Standard Format

```json
{
  "type": "https://api.orderflow.com/errors/insufficient-stock",
  "title": "Insufficient stock",
  "status": 409,
  "detail": "Product 'prod-123' has only 3 units available, but 10 were requested",
  "instance": "/orders/ord-456/items",
  "traceId": "abc123",
  "productId": "prod-123",
  "requested": 10,
  "available": 3
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | URI identifying the error type (can be documentation URL) |
| `title` | Yes | Short, human-readable summary |
| `status` | Yes | HTTP status code |
| `detail` | No | Human-readable explanation of this specific occurrence |
| `instance` | No | URI reference to the specific occurrence |
| (extensions) | No | Additional machine-readable properties |

### ASP.NET Core Implementation

```csharp
// Built-in support
app.UseExceptionHandler();
app.UseStatusCodePages();

// Or use ProblemDetailsFactory
public class OrdersEndpoint
{
    public static IResult GetOrder(Guid id, AppDbContext context)
    {
        var order = context.Orders.Find(id);
        if (order is null)
        {
            return Results.Problem(
                type: "https://api.orderflow.com/errors/not-found",
                title: "Order not found",
                statusCode: 404,
                detail: $"No order exists with ID '{id}'",
                instance: $"/orders/{id}");
        }
        return Results.Ok(order);
    }
}
```

---

## Core Concept #11 — Error Catalog

### Why This Matters

A documented error catalog:
- Helps frontend developers handle errors consistently
- Makes APIs self-documenting
- Catches missing error handling
- Guides monitoring and alerting

### Example Error Catalog

| Type | Title | Status | When |
|------|-------|--------|------|
| `/errors/validation` | Validation failed | 400 | Request body failed validation |
| `/errors/authentication` | Authentication required | 401 | Missing or invalid token |
| `/errors/authorization` | Not authorized | 403 | User lacks permission |
| `/errors/not-found` | Resource not found | 404 | Resource doesn't exist |
| `/errors/insufficient-stock` | Insufficient stock | 409 | Not enough inventory |
| `/errors/order-already-shipped` | Order already shipped | 409 | Can't modify shipped order |
| `/errors/duplicate-email` | Email already registered | 409 | Email taken |
| `/errors/rate-limit` | Rate limit exceeded | 429 | Too many requests |
| `/errors/internal` | Internal server error | 500 | Unexpected failure |

---

## Hands-On: OrderFlow Error Catalog

Create a complete error catalog for OrderFlow:

### Validation Errors (400)
- Missing required fields
- Invalid email format
- Negative quantities
- Unknown product ID format

### Business Rule Errors (409)
- Insufficient stock
- Order already cancelled
- Order already shipped
- Customer credit limit exceeded
- Product discontinued

### Authentication/Authorization (401/403)
- Token expired
- Token invalid
- Admin required
- Customer can only view own orders

### Not Found (404)
- Order not found
- Product not found
- Customer not found

---

## Deliverables

1. **Error Catalog** documenting all OrderFlow errors
2. **ProblemDetails Examples** for each error type
3. **Status Code Decision Tree** — flowchart for choosing the right code
4. **Client Error Handling Guide** — how consumers should handle each error type

---

## Resources

### Must-Read
- [RFC 7807 — Problem Details](https://datatracker.ietf.org/doc/html/rfc7807) — The standard
- [HTTP Status Codes](https://httpstatuses.io/) — Quick reference

### Videos
- [Nick Chapsas: Error Handling in ASP.NET Core](https://www.youtube.com/watch?v=a1ye9eGTB98)
- [Milan Jovanovic: Problem Details](https://www.youtube.com/watch?v=4NfflZilTvk)

### Documentation
- [ASP.NET Core Error Handling](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [Stripe API Errors](https://stripe.com/docs/api/errors) — Excellent example

---

## Reflection Questions

1. What's the difference between 400, 409, and 422?
2. When should you use 404 vs 410 (Gone)?
3. Why include a trace ID in error responses?
4. How does consistent error formatting help frontend development?

---

**Next:** [Part 3 — Pagination, Filtering, and Sorting](./part-3.md)
