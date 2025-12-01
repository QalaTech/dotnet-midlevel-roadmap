# 03 — Building Web APIs with ASP.NET Core

## Part 1 / 4 — The Request Pipeline, Endpoints, and Dependency Injection

---

## Why This Module Exists

Knowing REST theory (Module 02) is useless if you can't implement it correctly.

This module bridges design and implementation. You'll build the OrderFlow API we designed, learning:
- How requests flow through ASP.NET Core
- Why middleware order matters (and when it breaks things)
- How dependency injection actually works
- When Minimal APIs vs Controllers make sense

By the end, you'll have a production-structured API, not tutorial code.

---

## What You'll Build

The OrderFlow API from Module 02 becomes real code:
- Orders, Products, Customers endpoints
- Proper validation with FluentValidation
- Global error handling with ProblemDetails
- Authentication and authorization
- Background job processing

---

## Core Concept #1 — The ASP.NET Core Request Pipeline

### Why This Matters

Every HTTP request passes through a pipeline of middleware. Understanding this pipeline explains:
- Why authentication must come before authorization
- Why your custom middleware might not see certain requests
- Why some errors get caught and others don't
- How to debug "my endpoint isn't being hit" problems

### What Goes Wrong Without This

**Real scenario:** A developer adds custom logging middleware. It works for most requests but misses authentication failures. Why? The logging middleware was added after `UseAuthentication()`, so rejected requests never reach it.

**Real scenario:** CORS headers aren't being set on error responses. The error occurs in the endpoint, but the CORS middleware only runs on successful responses because it's configured incorrectly.

### The Pipeline Visualized

```
Request arrives
    ↓
[UseHttpsRedirection]     → Redirects HTTP to HTTPS
    ↓
[UseStaticFiles]          → Serves static files, short-circuits if found
    ↓
[UseRouting]              → Determines which endpoint matches
    ↓
[UseCors]                 → Adds CORS headers
    ↓
[UseAuthentication]       → Identifies who the user is
    ↓
[UseAuthorization]        → Checks if user can access resource
    ↓
[UseRateLimiter]          → Enforces rate limits
    ↓
[Endpoint Execution]      → Your code runs here
    ↓
Response flows back up through middleware (reverse order)
```

### Order Matters (A Lot)

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// WRONG ORDER - authentication happens after routing tries to authorize
app.UseAuthorization();  // ❌ Will fail - no user identity yet
app.UseAuthentication();

// CORRECT ORDER
app.UseAuthentication(); // ✅ First identify the user
app.UseAuthorization();  // ✅ Then check permissions
```

### Middleware That Short-Circuits

Some middleware stops the pipeline early:

```csharp
app.UseStaticFiles();  // If file found, returns it immediately
                       // Request never reaches your endpoints

app.UseRouting();
app.UseAuthentication();  // If token invalid, returns 401 immediately
app.UseAuthorization();   // If forbidden, returns 403 immediately

app.MapGet("/orders", GetOrders);  // Only reached if everything above passed
```

### Custom Middleware

```csharp
// RequestTimingMiddleware.cs
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        // Add correlation ID for request tracing
        var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
                            ?? Guid.NewGuid().ToString();
        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers.Append("X-Correlation-ID", correlationId);

        try
        {
            await _next(context);  // Call next middleware
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation(
                "Request {Method} {Path} completed in {ElapsedMs}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}

// Program.cs - add early in pipeline to capture all requests
app.UseMiddleware<RequestTimingMiddleware>();
```

---

## Core Concept #2 — Minimal APIs vs Controllers

### Why This Matters

ASP.NET Core offers two styles for defining endpoints. Understanding both helps you:
- Choose the right approach for your project
- Read codebases using either style
- Migrate between styles when needed

### What Goes Wrong Without This

**Real scenario:** A team using Controllers wants to add a simple health check. They create a full `HealthController` class with attributes, inheritance, and constructor injection — 50 lines for one endpoint that could be 1 line with Minimal APIs.

**Real scenario:** A team builds everything with Minimal APIs. Endpoints multiply, `Program.cs` becomes 500 lines, and nobody can find anything.

### Minimal APIs — When to Use

Best for:
- Microservices with few endpoints
- Simple APIs (health checks, webhooks)
- Learning — makes the pipeline explicit
- Teams that prefer functional style

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();

app.MapGet("/orders/{id:guid}", async (Guid id, IOrderService service) =>
{
    var order = await service.GetByIdAsync(id);
    return order is null ? Results.NotFound() : Results.Ok(order);
});

app.MapPost("/orders", async (CreateOrderRequest request, IOrderService service) =>
{
    var order = await service.CreateAsync(request);
    return Results.Created($"/orders/{order.Id}", order);
});

app.Run();
```

### Controllers — When to Use

Best for:
- Large APIs with many endpoints
- Teams familiar with MVC patterns
- When you need extensive attribute-based configuration
- APIs with complex routing requirements

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _service;

    public OrdersController(IOrderService service)
    {
        _service = service;
    }

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderResponse>> GetById(Guid id)
    {
        var order = await _service.GetByIdAsync(id);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    public async Task<ActionResult<OrderResponse>> Create(CreateOrderRequest request)
    {
        var order = await _service.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }
}
```

### Organizing Minimal APIs at Scale

Don't put everything in `Program.cs`. Use extension methods:

```csharp
// Endpoints/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("/", GetOrders);
        group.MapGet("/{id:guid}", GetOrderById);
        group.MapPost("/", CreateOrder);
        group.MapPatch("/{id:guid}", UpdateOrder);
        group.MapDelete("/{id:guid}", DeleteOrder);
        group.MapPost("/{id:guid}/items", AddOrderItem);
    }

    private static async Task<IResult> GetOrders(
        [AsParameters] OrderQueryParameters query,
        IOrderService service,
        CancellationToken ct)
    {
        var orders = await service.GetOrdersAsync(query, ct);
        return Results.Ok(orders);
    }

    private static async Task<IResult> GetOrderById(
        Guid id,
        IOrderService service,
        CancellationToken ct)
    {
        var order = await service.GetByIdAsync(id, ct);
        return order is null
            ? Results.Problem(
                title: "Order not found",
                statusCode: 404,
                detail: $"No order exists with ID '{id}'")
            : Results.Ok(order);
    }

    // ... other handlers
}

// Program.cs
app.MapOrderEndpoints();
app.MapProductEndpoints();
app.MapCustomerEndpoints();
```

---

## Core Concept #3 — Dependency Injection (The Real Understanding)

### Why This Matters

Dependency injection is the foundation of testable, maintainable code. But most developers use it without understanding:
- What "lifetime" actually means
- Why some services can't be injected into others
- How to diagnose DI-related bugs
- When to use which lifetime

### What Goes Wrong Without This

**Real scenario:** A developer registers `DbContext` as Singleton "for performance." The application works for one user. Under load, requests fail with "A second operation started on this context" — DbContext isn't thread-safe.

**Real scenario:** A Scoped service is injected into a Singleton service. The Scoped service gets "captured" and lives forever, holding onto a dead database connection. Requests start failing hours after deployment.

### Service Lifetimes Explained

```csharp
// SINGLETON — One instance for the entire application lifetime
// Use for: Stateless services, caches, configuration
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddSingleton<ISystemClock, SystemClock>();

// SCOPED — One instance per HTTP request (or scope)
// Use for: Database contexts, unit of work, per-request state
builder.Services.AddScoped<OrderFlowDbContext>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// TRANSIENT — New instance every time it's requested
// Use for: Lightweight, stateless services
builder.Services.AddTransient<IEmailBuilder, EmailBuilder>();
builder.Services.AddTransient<IDateTimeProvider, DateTimeProvider>();
```

### The Captive Dependency Problem

```csharp
// ❌ WRONG — Scoped service captured in Singleton
public class NotificationService  // Registered as Singleton
{
    private readonly IOrderRepository _repo;  // Registered as Scoped

    public NotificationService(IOrderRepository repo)
    {
        _repo = repo;  // This instance lives forever!
                       // Its DbContext connection dies, but it's still used
    }
}

// ✅ CORRECT — Use IServiceScopeFactory
public class NotificationService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public NotificationService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task NotifyOrderShippedAsync(Guid orderId)
    {
        using var scope = _scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        var order = await repo.GetByIdAsync(orderId);
        // ...
    }
}
```

### Enable Scope Validation (Catch Problems in Development)

```csharp
var builder = WebApplication.CreateBuilder(args);

// This throws if you inject Scoped into Singleton
if (builder.Environment.IsDevelopment())
{
    builder.Host.UseDefaultServiceProvider(options =>
    {
        options.ValidateScopes = true;
        options.ValidateOnBuild = true;
    });
}
```

### DI in Minimal APIs

```csharp
// Services are injected as parameters
app.MapGet("/orders/{id}", async (
    Guid id,                           // From route
    IOrderService service,              // From DI
    ILogger<Program> logger,            // From DI
    CancellationToken ct) =>            // From framework
{
    logger.LogInformation("Fetching order {OrderId}", id);
    return await service.GetByIdAsync(id, ct);
});

// Use [FromServices] for clarity (optional)
app.MapPost("/orders", async (
    CreateOrderRequest request,
    [FromServices] IOrderService service,
    [FromServices] IValidator<CreateOrderRequest> validator) =>
{
    // ...
});
```

---

## Core Concept #4 — Project Structure for Real APIs

### Why This Matters

Tutorial projects put everything in one folder. Real projects need structure that:
- Makes code easy to find
- Separates concerns clearly
- Scales to 100+ files without chaos
- Allows multiple developers to work without conflicts

### What Goes Wrong Without This

**Real scenario:** All 50 endpoints are in `Program.cs`. Adding a new endpoint means scrolling through 2000 lines. Merge conflicts happen constantly. Nobody wants to touch the file.

**Real scenario:** "Clean Architecture" with 15 empty abstraction layers. Creating a simple CRUD endpoint requires editing 8 files. Development velocity drops to zero.

### OrderFlow Project Structure

```
src/
├── OrderFlow.Api/                    # Presentation layer
│   ├── Endpoints/
│   │   ├── Orders/
│   │   │   ├── OrderEndpoints.cs
│   │   │   ├── CreateOrderRequest.cs
│   │   │   ├── OrderResponse.cs
│   │   │   └── OrderQueryParameters.cs
│   │   ├── Products/
│   │   └── Customers/
│   ├── Middleware/
│   │   ├── ErrorHandlingMiddleware.cs
│   │   ├── RequestTimingMiddleware.cs
│   │   └── CorrelationIdMiddleware.cs
│   ├── Extensions/
│   │   ├── ServiceCollectionExtensions.cs
│   │   └── WebApplicationExtensions.cs
│   ├── Program.cs
│   └── appsettings.json
│
├── OrderFlow.Application/            # Business logic
│   ├── Services/
│   │   ├── IOrderService.cs
│   │   ├── OrderService.cs
│   │   ├── IProductService.cs
│   │   └── ProductService.cs
│   ├── Validators/
│   │   ├── CreateOrderRequestValidator.cs
│   │   └── UpdateOrderRequestValidator.cs
│   └── DependencyInjection.cs
│
├── OrderFlow.Domain/                 # Domain entities
│   ├── Entities/
│   │   ├── Order.cs
│   │   ├── OrderItem.cs
│   │   ├── Product.cs
│   │   └── Customer.cs
│   ├── ValueObjects/
│   │   ├── Money.cs
│   │   └── Address.cs
│   └── Enums/
│       └── OrderStatus.cs
│
├── OrderFlow.Infrastructure/         # External concerns
│   ├── Persistence/
│   │   ├── OrderFlowDbContext.cs
│   │   └── Configurations/
│   │       ├── OrderConfiguration.cs
│   │       └── ProductConfiguration.cs
│   ├── Repositories/
│   │   ├── IOrderRepository.cs
│   │   ├── OrderRepository.cs
│   │   └── ProductRepository.cs
│   └── DependencyInjection.cs
│
tests/
├── OrderFlow.Api.Tests/
│   └── Endpoints/
├── OrderFlow.Application.Tests/
└── OrderFlow.Infrastructure.Tests/
```

### Clean Program.cs

```csharp
// Program.cs — Clean, scannable, tells a story
var builder = WebApplication.CreateBuilder(args);

// Add services (each extension method registers related services)
builder.Services
    .AddApplicationServices()
    .AddInfrastructureServices(builder.Configuration)
    .AddApiServices();

var app = builder.Build();

// Configure pipeline (order matters!)
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<RequestTimingMiddleware>();
app.UseMiddleware<ErrorHandlingMiddleware>();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

// Map endpoints
app.MapOrderEndpoints();
app.MapProductEndpoints();
app.MapCustomerEndpoints();
app.MapHealthChecks("/health");

app.Run();
```

---

## Hands-On: OrderFlow API Foundation

### Task 1 — Create the Project Structure

Create the solution structure above. Use the .NET CLI:

```bash
dotnet new sln -n OrderFlow
dotnet new webapi -n OrderFlow.Api -o src/OrderFlow.Api
dotnet new classlib -n OrderFlow.Application -o src/OrderFlow.Application
dotnet new classlib -n OrderFlow.Domain -o src/OrderFlow.Domain
dotnet new classlib -n OrderFlow.Infrastructure -o src/OrderFlow.Infrastructure

dotnet sln add src/OrderFlow.Api
dotnet sln add src/OrderFlow.Application
dotnet sln add src/OrderFlow.Domain
dotnet sln add src/OrderFlow.Infrastructure

# Add project references
cd src/OrderFlow.Api
dotnet add reference ../OrderFlow.Application
dotnet add reference ../OrderFlow.Infrastructure

cd ../OrderFlow.Application
dotnet add reference ../OrderFlow.Domain

cd ../OrderFlow.Infrastructure
dotnet add reference ../OrderFlow.Domain
```

### Task 2 — Implement the Domain

Create the domain entities from Module 01:

```csharp
// OrderFlow.Domain/Entities/Order.cs
public class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public Address ShippingAddress { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    public static Order Create(Guid customerId, Address shippingAddress)
    {
        return new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            Total = Money.Zero("USD"),
            ShippingAddress = shippingAddress,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Cannot modify non-pending order");

        var item = OrderItem.Create(Id, product, quantity);
        _items.Add(item);
        RecalculateTotal();
    }

    private void RecalculateTotal()
    {
        Total = _items.Aggregate(
            Money.Zero(Total.Currency),
            (sum, item) => sum + item.LineTotal);
    }
}
```

### Task 3 — Create a Basic Endpoint

```csharp
// OrderFlow.Api/Endpoints/Orders/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders");

        group.MapGet("/{id:guid}", GetOrderById)
            .WithName("GetOrder")
            .WithDescription("Retrieves a single order by ID")
            .Produces<OrderResponse>(200)
            .ProducesProblem(404);
    }

    private static async Task<IResult> GetOrderById(
        Guid id,
        IOrderService service,
        CancellationToken ct)
    {
        var order = await service.GetByIdAsync(id, ct);

        return order is null
            ? Results.Problem(
                type: "https://api.orderflow.com/errors/not-found",
                title: "Order not found",
                statusCode: 404,
                detail: $"No order exists with ID '{id}'")
            : Results.Ok(order);
    }
}
```

---

## Deliverables

1. **OrderFlow solution** with correct project structure
2. **Domain entities** with value objects and invariants
3. **Basic endpoints** for Orders (GET by ID, GET list, POST create)
4. **Clean Program.cs** using extension methods
5. **Request timing middleware** that logs all requests

---

## Resources

### Must-Read
- [ASP.NET Core Fundamentals](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/) — Official docs, essential
- [Dependency Injection Guidelines](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines)

### Videos
- [Nick Chapsas: Minimal APIs in .NET 7](https://www.youtube.com/watch?v=eRJF0zzEikM)
- [Nick Chapsas: Why I Stopped Using Minimal APIs](https://www.youtube.com/watch?v=X9WF6A2L8L4) — Balanced view
- [Raw Coding: ASP.NET Core Middleware](https://www.youtube.com/watch?v=5eifH7LEnGo)
- [Milan Jovanovic: Clean Architecture in .NET](https://www.youtube.com/watch?v=gGa7SLk2-xQ)

### Tools
- [Bruno](https://www.usebruno.com/) — Open-source API client
- [REST Client for VS Code](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)
- [HTTP Files in Visual Studio](https://learn.microsoft.com/en-us/aspnet/core/test/http-files)

### Reference Projects
- [eShop Reference Application](https://github.com/dotnet/eshop) — Microsoft's reference
- [Clean Architecture Template](https://github.com/jasontaylordev/CleanArchitecture)
- [Ardalis Clean Architecture](https://github.com/ardalis/CleanArchitecture)

---

## Reflection Questions

1. Why can't you inject a Scoped service into a Singleton service directly?
2. When would you choose Minimal APIs over Controllers?
3. Why does middleware order matter?
4. What happens if you add `UseAuthorization` before `UseAuthentication`?

---

**Next:** [Part 2 — Validation, Error Handling, and Configuration](./part-2.md)
