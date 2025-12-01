# 02 — HTTP & REST Fundamentals

## Part 4 / 4 — Versioning, Contracts, and Long-Term API Design

---

## Why This Matters

APIs are contracts. Once clients depend on them, changes become expensive:
- Breaking changes break clients
- Mobile apps can't be updated remotely
- Enterprise integrations require months of coordination
- Documentation falls out of sync

This part teaches you to design APIs that survive years of evolution.

---

## Core Concept #17 — API Versioning

### Why You Need It

**What goes wrong:** The team needs to change the response format. They update the API. Every mobile app crashes. Support is flooded. Emergency rollback. The CEO asks what happened.

### Versioning Strategies

#### URL Path Versioning

```http
GET /v1/orders
GET /v2/orders
```

**Pros:**
- Obvious and explicit
- Easy to route
- Clear in logs

**Cons:**
- Not "pure" REST
- Clients must update URLs

**When to use:** Public APIs, when you want version to be unmissable.

#### Header Versioning

```http
GET /orders
Api-Version: 2
```

**Pros:**
- Clean URLs
- Version negotiation possible

**Cons:**
- Hidden (developers miss it)
- Harder to test in browser

**When to use:** Internal APIs, when you want flexible version handling.

#### Query Parameter Versioning

```http
GET /orders?api-version=2
```

**Pros:**
- Easy to add to any client
- Visible in logs

**Cons:**
- Pollutes query string
- Less common

### Implementation

```csharp
// URL versioning in ASP.NET Core
var v1 = app.MapGroup("/v1");
v1.MapGet("/orders", GetOrdersV1);

var v2 = app.MapGroup("/v2");
v2.MapGet("/orders", GetOrdersV2);

// With Microsoft.AspNetCore.Mvc.Versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});
```

### Version Lifecycle

```
v1 (stable) → v2 (stable) → v3 (stable)
     ↓            ↓
  deprecated   deprecated → sunset
```

---

## Core Concept #18 — Breaking vs Non-Breaking Changes

### What Goes Wrong

**Real scenario:** A developer "improves" the API by renaming `customerId` to `customer_id`. It's a small change. Every client breaks. The "improvement" cost 3 days of emergency fixes.

### Breaking Changes (NEVER Do Without Versioning)

| Change | Why It Breaks |
|--------|---------------|
| Rename field | Clients look for old name |
| Remove field | Clients expect it to exist |
| Change field type | Parsing fails |
| Change URL structure | Requests go to wrong place |
| Change status codes | Error handling breaks |
| Add required field to request | Old requests fail validation |
| Remove allowed values | Valid requests become invalid |

### Non-Breaking Changes (Always Safe)

| Change | Why It's Safe |
|--------|---------------|
| Add optional field to response | Clients ignore unknown fields |
| Add new endpoint | Doesn't affect existing endpoints |
| Add optional query parameter | Old requests work without it |
| Add new status code (for new case) | Existing cases unchanged |
| Add new allowed value | Old values still work |
| Improve performance | Transparent to clients |

### The Robustness Principle

> Be conservative in what you send, liberal in what you accept.

```csharp
// Accept both old and new formats
public record CreateOrderRequest
{
    public string? CustomerId { get; init; }

    // Accept old name too (deprecated)
    [JsonPropertyName("customer_id")]
    [Obsolete("Use CustomerId")]
    public string? CustomerIdLegacy { get; init; }

    public string ResolvedCustomerId =>
        CustomerId ?? CustomerIdLegacy ?? throw new ValidationException("Customer ID required");
}
```

---

## Core Concept #19 — API Contracts and OpenAPI

### Why This Matters

An API without a contract is like a function without a signature. Clients guess. They guess wrong. They blame you.

### OpenAPI (Swagger)

OpenAPI is a machine-readable API description:

```yaml
openapi: 3.0.3
info:
  title: OrderFlow API
  version: 1.0.0

paths:
  /orders:
    get:
      summary: List orders
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, processing, shipped]
      responses:
        '200':
          description: List of orders
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderList'
        '401':
          $ref: '#/components/responses/Unauthorized'
```

### Contract-First vs Code-First

**Contract-First:**
1. Design OpenAPI spec
2. Generate server stubs
3. Implement handlers
4. Spec is always accurate

**Code-First:**
1. Write code
2. Generate OpenAPI from code
3. Spec reflects reality
4. Risk of undocumented behavior

**Recommendation:** Use code-first with strict annotations. Generate and commit the spec. Diff changes in PR review.

### ASP.NET Core OpenAPI

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "OrderFlow API",
        Version = "v1",
        Description = "Order management API for B2B customers"
    });
});

app.UseSwagger();
app.UseSwaggerUI();

// Endpoint with documentation
app.MapGet("/orders/{id}", async (Guid id, OrderService service) =>
{
    var order = await service.GetByIdAsync(id);
    return order is null ? Results.NotFound() : Results.Ok(order);
})
.WithName("GetOrder")
.WithDescription("Retrieves a single order by ID")
.Produces<OrderResponse>(200)
.Produces<ProblemDetails>(404)
.WithOpenApi();
```

---

## Core Concept #20 — Rate Limiting

### Why This Matters

Without rate limits:
- One client can consume all resources
- Abusive scripts can DOS your API
- Costs spiral out of control
- Legitimate users suffer

### Rate Limit Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 543
X-RateLimit-Reset: 1705316400

HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

### Implementation in ASP.NET Core

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;

    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueLimit = 10;
    });

    options.AddSlidingWindowLimiter("strict", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6;
        opt.PermitLimit = 60;
    });
});

app.UseRateLimiter();

app.MapGet("/orders", GetOrders)
   .RequireRateLimiting("api");
```

### What to Rate Limit

| Endpoint | Strategy |
|----------|----------|
| Public APIs | Per API key, moderate limits |
| Authentication | Per IP, strict limits (prevent brute force) |
| Expensive operations | Per user, low limits |
| Read endpoints | Per user, generous limits |
| Webhooks | Per source, moderate limits |

---

## Core Concept #21 — Idempotency

### Why This Matters

Networks fail. Clients retry. Without idempotency:
- Retry creates duplicate order
- Customer charged twice
- Inventory double-decremented

### Idempotency Keys

Clients send a unique key with mutating requests:

```http
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "customerId": "cust-123", ... }
```

Server behavior:
1. First request: Process and store result with key
2. Same key again: Return stored result (don't reprocess)

### Implementation

```csharp
public class IdempotencyMiddleware
{
    private readonly IIdempotencyStore _store;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        if (!context.Request.Headers.TryGetValue("Idempotency-Key", out var key))
        {
            await next(context);
            return;
        }

        var existingResponse = await _store.GetAsync(key!);
        if (existingResponse is not null)
        {
            // Return cached response
            context.Response.StatusCode = existingResponse.StatusCode;
            await context.Response.WriteAsJsonAsync(existingResponse.Body);
            return;
        }

        // Capture response
        var originalBody = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;

        await next(context);

        // Store for future replays
        memoryStream.Seek(0, SeekOrigin.Begin);
        var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();

        await _store.SetAsync(key!, new StoredResponse
        {
            StatusCode = context.Response.StatusCode,
            Body = responseBody
        }, TimeSpan.FromHours(24));

        memoryStream.Seek(0, SeekOrigin.Begin);
        await memoryStream.CopyToAsync(originalBody);
    }
}
```

---

## Core Concept #22 — Deprecation Strategy

### Why This Matters

APIs need to evolve. Old versions need to die. But clients need time to migrate.

### Deprecation Process

1. **Announce** — Document deprecation in changelog, headers, response body
2. **Set Timeline** — Give clients 6-12 months (depends on API type)
3. **Warn** — Add deprecation headers to responses
4. **Monitor** — Track usage of deprecated endpoints
5. **Sunset** — Return 410 Gone after deadline

### Deprecation Headers

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jun 2025 00:00:00 GMT
Link: </v2/orders>; rel="successor-version"
```

### Communication

```markdown
## Deprecation Notice: /v1/orders

**Deprecated:** January 15, 2025
**Sunset:** June 1, 2025

### What's Changing
The `/v1/orders` endpoint is being replaced by `/v2/orders`.

### Migration Guide
1. Update endpoint URL from `/v1/orders` to `/v2/orders`
2. Change `customer_id` to `customerId` in requests
3. Handle new `fulfillmentStatus` field in responses

### Questions?
Contact api-support@orderflow.com
```

---

## Hands-On: OrderFlow API Specification

Create the complete API specification for OrderFlow:

### 1. OpenAPI Document

```yaml
openapi: 3.0.3
info:
  title: OrderFlow API
  version: 1.0.0
  description: B2B order management API

servers:
  - url: https://api.orderflow.com/v1

paths:
  /orders:
    get:
      # Full specification
    post:
      # Full specification
  /orders/{id}:
    get:
      # Full specification
    patch:
      # Full specification
    delete:
      # Full specification
  # ... all endpoints

components:
  schemas:
    Order:
      # Full schema
    CreateOrderRequest:
      # Full schema
  responses:
    NotFound:
      # Reusable response
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
```

### 2. Versioning Strategy Document

Document how OrderFlow handles versions:
- URL versioning (`/v1`, `/v2`)
- Deprecation timelines
- Breaking change policy
- Migration guides

### 3. Rate Limiting Policy

| Endpoint Type | Limit | Window |
|---------------|-------|--------|
| Read (GET) | 1000/minute | Per API key |
| Write (POST/PUT) | 100/minute | Per API key |
| Auth | 10/minute | Per IP |

---

## Module Summary

After completing Module 02, you can:

| Skill | Evidence |
|-------|----------|
| Design REST APIs | Resource-based URLs, proper verbs |
| Use correct status codes | Error catalog with right codes |
| Implement pagination | Cursor and offset strategies |
| Handle filtering/sorting | Composable query parameters |
| Version APIs safely | Non-breaking changes, deprecation |
| Document with OpenAPI | Complete, accurate specification |

Your OrderFlow API design becomes the blueprint for Module 03.

---

## Deliverables

1. **Complete OpenAPI specification** for OrderFlow
2. **Versioning strategy document**
3. **Rate limiting policy**
4. **Deprecation process document**
5. **API style guide** for your team

---

## Resources

### Must-Read
- [Stripe API Design](https://stripe.com/docs/api) — Study their patterns
- [GitHub API Versioning](https://docs.github.com/en/rest/overview/api-versions)
- [Microsoft API Guidelines](https://github.com/microsoft/api-guidelines)

### Videos
- [CodeOpinion: API Versioning](https://www.youtube.com/watch?v=8yxyS7g8iJc)
- [Nick Chapsas: OpenAPI in ASP.NET Core](https://www.youtube.com/watch?v=9bCLq5Cz4W0)

### Tools
- [Swagger Editor](https://editor.swagger.io/) — Design OpenAPI specs
- [Spectral](https://stoplight.io/spectral) — API linting
- [Prism](https://stoplight.io/prism) — Mock API from OpenAPI

---

## Reflection Questions

1. When should you create a new API version vs adding optional fields?
2. How do you balance API stability with the need to improve?
3. What's the cost of NOT having rate limiting?
4. How would you communicate a breaking change to enterprise customers?

---

**Next:** [Module 03 — Building APIs with ASP.NET Core](../03-building-apis/part-1.md)
