# Part 1 — Architecture Patterns

## Why This Matters

Architecture patterns are vocabulary. When a senior engineer says "we're using Clean Architecture," everyone on the team should understand:
- Where business logic lives
- What depends on what
- How to add new features

Without shared vocabulary:
- Every developer invents their own structure
- Code reviews become philosophical debates
- New team members take months to become productive

---

## What Goes Wrong Without This

### The "Put It Anywhere" Syndrome

```
Developer: "Where should I add this new validation logic?"
Team: *silence*
Developer: *puts it in the controller*

Six months later:
- Controller has 2,000 lines
- Same validation duplicated in 3 places
- Unit testing requires HTTP context mocking
```

### The Dependency Disaster

```csharp
// Domain layer referencing infrastructure
public class Order
{
    public void Save()
    {
        var context = new SqlDbContext();  // Domain depends on SQL!
        context.Orders.Add(this);
        context.SaveChanges();
    }
}

// Later:
// "We need to switch to PostgreSQL"
// "We need to add caching"
// "We need to publish events"
// All require changing the domain model
```

### The "Clean Architecture" Cargo Cult

```
Team: "Let's use Clean Architecture!"
Reality:
- 47 projects in the solution
- 12 layers of abstraction
- IOrderRepository, IOrderRepositoryAsync, IOrderReadRepository...
- Simple CRUD takes 8 files to implement

"We have Clean Architecture but dirty productivity"
```

---

## Architecture Patterns Compared

### The Patterns at a Glance

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE PATTERNS                        │
├─────────────────┬──────────────────┬───────────────────────────┤
│    Layered      │     Clean /      │     Vertical Slice        │
│   Traditional   │    Hexagonal     │                           │
├─────────────────┼──────────────────┼───────────────────────────┤
│ UI              │    Frameworks    │  Feature: Create Order    │
│ ↓               │    ↓        ↓    │  ┌─────────────────────┐  │
│ Business        │ Adapters ← Core  │  │ Endpoint            │  │
│ ↓               │    ↓        ↓    │  │ Handler             │  │
│ Data Access     │    Adapters      │  │ Domain Logic        │  │
│                 │                  │  │ Repository          │  │
│                 │                  │  └─────────────────────┘  │
├─────────────────┼──────────────────┼───────────────────────────┤
│ Easy to start   │ Clear boundaries │ Features are cohesive    │
│ Common pattern  │ Testable         │ Easy to delete           │
│ But: becomes    │ But: ceremony    │ But: possible            │
│ a ball of mud   │ for simple cases │ duplication              │
└─────────────────┴──────────────────┴───────────────────────────┘
```

### Traditional Layered Architecture

```
┌────────────────────────┐
│     Presentation       │  Controllers, Views
├────────────────────────┤
│     Business Logic     │  Services, Rules
├────────────────────────┤
│     Data Access        │  Repositories, DbContext
├────────────────────────┤
│       Database         │
└────────────────────────┘

Dependencies flow downward.
Each layer only talks to the layer below.
```

**When it works**: Simple CRUD apps, small teams, short-lived projects.

**When it fails**: Complex domains, multiple integration points, teams that need to work independently.

### Clean / Hexagonal Architecture

```
                    ┌─────────────────┐
                    │   Controllers   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    ▼                    │
        │    ┌───────────────────────────────┐    │
        │    │        Application            │    │
        │    │   (Use Cases / Handlers)      │    │
        │    └───────────────┬───────────────┘    │
        │                    │                    │
        │    ┌───────────────▼───────────────┐    │
        │    │          Domain               │    │
        │    │  (Entities, Value Objects,    │    │
        │    │   Domain Services, Events)    │    │
        │    └───────────────────────────────┘    │
        │                    ▲                    │
        └────────────────────┼────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────┴────┐         ┌─────┴─────┐        ┌────┴────┐
   │   DB    │         │  Message  │        │External │
   │Adapter  │         │  Queue    │        │  APIs   │
   └─────────┘         └───────────┘        └─────────┘

Key rule: Dependencies point INWARD
Domain knows nothing about databases, queues, or HTTP
```

### Vertical Slice Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         OrderFlow                            │
├───────────────┬───────────────┬───────────────┬─────────────┤
│ Create Order  │ Cancel Order  │ Get Order     │ List Orders │
├───────────────┼───────────────┼───────────────┼─────────────┤
│ Endpoint      │ Endpoint      │ Endpoint      │ Endpoint    │
│ Validator     │ Validator     │ (none)        │ (none)      │
│ Handler       │ Handler       │ Handler       │ Handler     │
│ Domain        │ Domain        │ Query         │ Query       │
│ Repository    │ Events        │ DTO           │ Pagination  │
└───────────────┴───────────────┴───────────────┴─────────────┘

Each feature is a self-contained unit.
Features don't share code unless explicitly needed.
```

### Modular Monolith

```
┌─────────────────────────────────────────────────────────────┐
│                    Single Deployment Unit                    │
├─────────────────┬─────────────────┬─────────────────────────┤
│   Orders        │   Inventory     │   Payments              │
│   Module        │   Module        │   Module                │
├─────────────────┼─────────────────┼─────────────────────────┤
│ • API           │ • API           │ • API                   │
│ • Domain        │ • Domain        │ • Domain                │
│ • Data          │ • Data          │ • Data                  │
│ • Events        │ • Events        │ • Events                │
├─────────────────┴─────────────────┴─────────────────────────┤
│                   Shared Kernel                             │
│              (Common types, Integration Events)             │
└─────────────────────────────────────────────────────────────┘

Modules communicate via:
1. Public APIs (interfaces)
2. Integration events
NOT by reaching into each other's internals
```

---

## Implementing Clean Architecture

### Project Structure

```
OrderFlow/
├── OrderFlow.Domain/           # Core domain (no dependencies)
│   ├── Entities/
│   │   └── Order.cs
│   ├── ValueObjects/
│   │   └── Money.cs
│   ├── Events/
│   │   └── OrderCreated.cs
│   └── Interfaces/
│       └── IOrderRepository.cs
│
├── OrderFlow.Application/      # Use cases (depends on Domain)
│   ├── Orders/
│   │   ├── Commands/
│   │   │   └── CreateOrder.cs
│   │   └── Queries/
│   │       └── GetOrder.cs
│   └── Common/
│       └── Interfaces/
│           └── ICurrentUser.cs
│
├── OrderFlow.Infrastructure/   # External concerns (depends on Application)
│   ├── Persistence/
│   │   ├── OrderDbContext.cs
│   │   └── OrderRepository.cs
│   └── Services/
│       └── DateTimeService.cs
│
└── OrderFlow.Api/              # Entry point (depends on all)
    ├── Controllers/
    └── Program.cs
```

### The Dependency Rule in Code

```csharp
// Domain Layer - ZERO external dependencies
namespace OrderFlow.Domain.Entities;

public class Order
{
    public int Id { get; private set; }
    public int CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();

    public static Order Create(int customerId, List<OrderLineDto> lines)
    {
        if (!lines.Any())
            throw new DomainException("Order must have at least one line");

        var order = new Order
        {
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };

        foreach (var line in lines)
        {
            order.AddLine(line.ProductId, line.Quantity, line.UnitPrice);
        }

        order._events.Add(new OrderCreated(order.Id, customerId, order.Total));
        return order;
    }

    public void AddLine(int productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Cannot modify non-pending order");

        var line = new OrderLine(productId, quantity, unitPrice);
        _lines.Add(line);
        RecalculateTotal();
    }

    private void RecalculateTotal()
    {
        Total = _lines.Sum(l => l.LineTotal);
    }
}
```

```csharp
// Application Layer - Depends ONLY on Domain
namespace OrderFlow.Application.Orders.Commands;

public record CreateOrderCommand(
    int CustomerId,
    List<OrderLineDto> Lines) : IRequest<Result<int>>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Result<int>>
{
    private readonly IOrderRepository _repository;  // Interface from Domain
    private readonly IUnitOfWork _unitOfWork;       // Interface from Domain
    private readonly IPublisher _publisher;

    public CreateOrderHandler(
        IOrderRepository repository,
        IUnitOfWork unitOfWork,
        IPublisher publisher)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _publisher = publisher;
    }

    public async Task<Result<int>> Handle(
        CreateOrderCommand request,
        CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Lines);

        _repository.Add(order);
        await _unitOfWork.SaveChangesAsync(ct);

        // Publish domain events
        foreach (var @event in order.DomainEvents)
        {
            await _publisher.Publish(@event, ct);
        }

        return Result.Success(order.Id);
    }
}
```

```csharp
// Infrastructure Layer - Implements Domain interfaces
namespace OrderFlow.Infrastructure.Persistence;

public class OrderRepository : IOrderRepository
{
    private readonly OrderDbContext _context;

    public OrderRepository(OrderDbContext context)
    {
        _context = context;
    }

    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct)
    {
        return await _context.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);
    }

    public void Add(Order order)
    {
        _context.Orders.Add(order);
    }
}
```

---

## Implementing Vertical Slice Architecture

### Project Structure

```
OrderFlow/
├── OrderFlow.Api/
│   ├── Features/
│   │   ├── Orders/
│   │   │   ├── CreateOrder.cs      # Endpoint + Handler + Validator
│   │   │   ├── GetOrder.cs
│   │   │   ├── CancelOrder.cs
│   │   │   └── ListOrders.cs
│   │   └── Customers/
│   │       ├── CreateCustomer.cs
│   │       └── GetCustomer.cs
│   ├── Common/
│   │   ├── Behaviors/              # Cross-cutting concerns
│   │   │   ├── ValidationBehavior.cs
│   │   │   └── LoggingBehavior.cs
│   │   └── Models/
│   │       └── Result.cs
│   └── Program.cs
└── OrderFlow.Domain/              # Shared domain (kept minimal)
    └── Entities/
        └── Order.cs
```

### A Complete Vertical Slice

```csharp
// Features/Orders/CreateOrder.cs - Everything in one file!
namespace OrderFlow.Api.Features.Orders;

// Request
public record CreateOrderRequest(
    int CustomerId,
    List<OrderLineRequest> Lines);

public record OrderLineRequest(
    int ProductId,
    int Quantity,
    decimal UnitPrice);

// Response
public record CreateOrderResponse(int OrderId, decimal Total);

// Validator
public class CreateOrderValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).GreaterThan(0);
        RuleFor(x => x.Lines).NotEmpty();
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(x => x.ProductId).GreaterThan(0);
            line.RuleFor(x => x.Quantity).GreaterThan(0);
            line.RuleFor(x => x.UnitPrice).GreaterThan(0);
        });
    }
}

// Handler (MediatR)
public class CreateOrderHandler : IRequestHandler<CreateOrderRequest, Result<CreateOrderResponse>>
{
    private readonly OrderDbContext _context;

    public CreateOrderHandler(OrderDbContext context)
    {
        _context = context;
    }

    public async Task<Result<CreateOrderResponse>> Handle(
        CreateOrderRequest request,
        CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Lines);

        _context.Orders.Add(order);
        await _context.SaveChangesAsync(ct);

        return Result.Success(new CreateOrderResponse(order.Id, order.Total));
    }
}

// Endpoint (Minimal API)
public static class CreateOrderEndpoint
{
    public static void Map(WebApplication app)
    {
        app.MapPost("/api/orders", async (
            CreateOrderRequest request,
            IMediator mediator) =>
        {
            var result = await mediator.Send(request);
            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value.OrderId}", result.Value)
                : Results.BadRequest(result.Error);
        })
        .WithName("CreateOrder")
        .WithTags("Orders")
        .Produces<CreateOrderResponse>(StatusCodes.Status201Created)
        .ProducesProblem(StatusCodes.Status400BadRequest);
    }
}
```

---

## Implementing Modular Monolith

### Project Structure

```
OrderFlow/
├── src/
│   ├── Modules/
│   │   ├── Orders/
│   │   │   ├── OrderFlow.Orders.Api/
│   │   │   │   └── OrdersModule.cs
│   │   │   ├── OrderFlow.Orders.Application/
│   │   │   ├── OrderFlow.Orders.Domain/
│   │   │   └── OrderFlow.Orders.Infrastructure/
│   │   ├── Inventory/
│   │   │   └── ... (same structure)
│   │   └── Payments/
│   │       └── ... (same structure)
│   ├── Shared/
│   │   └── OrderFlow.Shared.Contracts/
│   │       └── IntegrationEvents/
│   │           └── OrderCreatedIntegrationEvent.cs
│   └── Host/
│       └── OrderFlow.Api/
│           └── Program.cs
```

### Module Registration

```csharp
// Modules/Orders/OrderFlow.Orders.Api/OrdersModule.cs
namespace OrderFlow.Orders.Api;

public static class OrdersModule
{
    public static IServiceCollection AddOrdersModule(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Register this module's services
        services.AddDbContext<OrdersDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("Orders")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(OrdersModule).Assembly));

        return services;
    }

    public static IEndpointRouteBuilder MapOrdersEndpoints(
        this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .WithTags("Orders");

        group.MapPost("/", CreateOrder.Handle);
        group.MapGet("/{id}", GetOrder.Handle);
        group.MapPost("/{id}/cancel", CancelOrder.Handle);

        return app;
    }
}

// Host/OrderFlow.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add modules
builder.Services.AddOrdersModule(builder.Configuration);
builder.Services.AddInventoryModule(builder.Configuration);
builder.Services.AddPaymentsModule(builder.Configuration);

var app = builder.Build();

// Map module endpoints
app.MapOrdersEndpoints();
app.MapInventoryEndpoints();
app.MapPaymentsEndpoints();

app.Run();
```

### Inter-Module Communication

```csharp
// Shared/OrderFlow.Shared.Contracts/IntegrationEvents/OrderCreatedIntegrationEvent.cs
namespace OrderFlow.Shared.Contracts.IntegrationEvents;

// This is the contract modules agree on
public record OrderCreatedIntegrationEvent(
    int OrderId,
    int CustomerId,
    List<OrderLineItem> Items,
    decimal Total,
    DateTime CreatedAt);

// Orders module publishes
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Result<int>>
{
    private readonly IPublisher _publisher;

    public async Task<Result<int>> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Lines);
        _context.Orders.Add(order);
        await _context.SaveChangesAsync(ct);

        // Publish integration event for other modules
        await _publisher.Publish(new OrderCreatedIntegrationEvent(
            order.Id,
            order.CustomerId,
            order.Lines.Select(l => new OrderLineItem(l.ProductId, l.Quantity)).ToList(),
            order.Total,
            DateTime.UtcNow), ct);

        return Result.Success(order.Id);
    }
}

// Inventory module subscribes
public class OrderCreatedHandler : INotificationHandler<OrderCreatedIntegrationEvent>
{
    private readonly IInventoryService _inventory;

    public async Task Handle(OrderCreatedIntegrationEvent notification, CancellationToken ct)
    {
        await _inventory.ReserveStockAsync(notification.Items, ct);
    }
}
```

---

## Architecture Testing with NetArchTest

### Enforcing Dependency Rules

```csharp
// Tests/ArchitectureTests/DependencyTests.cs
using NetArchTest.Rules;

public class DependencyTests
{
    private const string DomainNamespace = "OrderFlow.Domain";
    private const string ApplicationNamespace = "OrderFlow.Application";
    private const string InfrastructureNamespace = "OrderFlow.Infrastructure";
    private const string ApiNamespace = "OrderFlow.Api";

    [Fact]
    public void Domain_Should_Not_Depend_On_Application()
    {
        var result = Types
            .InAssembly(typeof(Order).Assembly)
            .ShouldNot()
            .HaveDependencyOn(ApplicationNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful,
            $"Domain has forbidden dependency: {string.Join(", ", result.FailingTypeNames ?? [])}");
    }

    [Fact]
    public void Domain_Should_Not_Depend_On_Infrastructure()
    {
        var result = Types
            .InAssembly(typeof(Order).Assembly)
            .ShouldNot()
            .HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Domain_Should_Not_Depend_On_External_Libraries()
    {
        var result = Types
            .InAssembly(typeof(Order).Assembly)
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .And()
            .ShouldNot()
            .HaveDependencyOn("MassTransit")
            .And()
            .ShouldNot()
            .HaveDependencyOn("FluentValidation")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Application_Should_Not_Depend_On_Infrastructure()
    {
        var result = Types
            .InAssembly(typeof(CreateOrderCommand).Assembly)
            .ShouldNot()
            .HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Handlers_Should_Not_Return_Entities()
    {
        var result = Types
            .InAssembly(typeof(CreateOrderHandler).Assembly)
            .That()
            .ImplementInterface(typeof(IRequestHandler<,>))
            .Should()
            .NotHaveDependencyOn("OrderFlow.Domain.Entities")
            .GetResult();

        // This ensures handlers return DTOs, not domain entities
    }
}
```

### Module Isolation Tests

```csharp
[Fact]
public void Orders_Module_Should_Not_Reference_Inventory_Internals()
{
    var result = Types
        .InAssembly(typeof(OrdersModule).Assembly)
        .ShouldNot()
        .HaveDependencyOn("OrderFlow.Inventory.Domain")
        .And()
        .ShouldNot()
        .HaveDependencyOn("OrderFlow.Inventory.Infrastructure")
        .GetResult();

    // Orders can only reference Inventory through shared contracts
    Assert.True(result.IsSuccessful);
}
```

---

## Architecture Decision Records (ADRs)

### Why ADRs Matter

```
Six months later:
Developer: "Why did we use Event Sourcing here?"
Original dev: "I don't remember"
ADR: "Decision 003: We chose Event Sourcing for the Order
      aggregate because of audit requirements from Finance.
      Alternative considered: Soft deletes with trigger logging.
      Trade-off: More complexity, but immutable audit trail."

Future decisions are better informed.
```

### ADR Template

```markdown
# ADR 003: Architecture Pattern for OrderFlow

## Status
Accepted

## Context
We're building OrderFlow, an order management system. The team is
growing from 3 to 8 developers. We need an architecture that:
- Allows multiple developers to work without conflicts
- Makes testing straightforward
- Enables future service extraction if needed

## Decision
We will use a **Modular Monolith** with vertical slices inside each module.

## Alternatives Considered

### Option 1: Traditional Layered Architecture
- Pros: Simple, familiar pattern
- Cons: Tends toward God services, cross-cutting changes

### Option 2: Full Clean Architecture
- Pros: Clear separation, highly testable
- Cons: Too much ceremony for our current size

### Option 3: Microservices
- Pros: Independent deployment, team autonomy
- Cons: Premature for our team size, operational overhead

## Consequences
### Positive
- Teams can own modules without stepping on each other
- Can extract to services later without major refactoring
- Single deployment simplifies operations

### Negative
- Must enforce module boundaries (automated tests required)
- Shared database means careful schema coordination
- All modules deploy together (no independent scaling yet)

## Notes
- Review this decision when we reach 15+ engineers
- Track module boundary violations in CI
```

---

## Choosing the Right Pattern

### Decision Matrix

| Factor | Layered | Clean | Vertical Slice | Modular Monolith |
|--------|---------|-------|----------------|------------------|
| Team size 1-3 | Good | Overkill | Great | Overkill |
| Team size 4-10 | OK | Good | Good | Great |
| Team size 10+ | Poor | Good | OK | Good/Split |
| Simple CRUD | Great | Overkill | OK | Overkill |
| Complex domain | Poor | Great | OK | Great |
| Many integrations | Poor | Great | OK | Good |
| Need to split later | Hard | Medium | Easy | Easy |

### The Honest Truth

**Start simple. Add structure when you feel pain.**

```
Week 1-4:   "Just make it work" (folders by type)
Week 4-12:  "This is getting messy" (extract services)
Week 12+:   "We need boundaries" (modular structure)

Don't start with 47 projects on day 1.
```

---

## Hands-On Exercise: Refactor to Vertical Slices

### Step 1: Audit Current Structure

Look at OrderFlow's current state:
- How many files touch a single feature?
- Are there "God" services?
- Can you delete a feature without breaking others?

### Step 2: Choose One Feature to Refactor

Pick a feature like "Create Order" and consolidate:
- Endpoint
- Validator
- Handler
- Any feature-specific domain logic

### Step 3: Write an ADR

Document:
- Why you chose this approach
- What alternatives you considered
- What the trade-offs are

### Step 4: Add Architecture Tests

Ensure your boundaries are enforced:
- Domain doesn't reference infrastructure
- Modules don't reach into each other

---

## Deliverables

1. **Architecture Diagram**: Before/after comparison of OrderFlow structure
2. **Refactored Feature**: One feature implemented as a vertical slice
3. **ADRs**: At least 2 documenting architecture decisions
4. **Architecture Tests**: NetArchTest tests enforcing boundaries

---

## Reflection Questions

1. What problems does Clean Architecture solve that Layered doesn't?
2. When would Vertical Slice be worse than Clean Architecture?
3. How do you know when it's time to extract a microservice?
4. What's more important: the pattern or the principles behind it?

---

## Resources

### Architecture Patterns
- [Jason Taylor - Clean Architecture with .NET](https://github.com/jasontaylordev/CleanArchitecture)
- [Jimmy Bogard - Vertical Slice Architecture](https://www.youtube.com/watch?v=SUiWfhAhgQw)
- [Steve Smith - Modular Monoliths](https://www.youtube.com/watch?v=5OjqD-ow8GE)

### Tools
- [NetArchTest](https://github.com/BenMorris/NetArchTest)
- [MediatR](https://github.com/jbogard/MediatR)
- [FluentValidation](https://docs.fluentvalidation.net/)

### ADRs
- [ADR GitHub Organization](https://adr.github.io/)
- [Michael Nygard - Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)

---

**Next:** [Part 2 — Domain-Driven Design](./part-2.md)
