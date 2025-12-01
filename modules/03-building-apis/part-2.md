# 03 — Building Web APIs with ASP.NET Core

## Part 2 / 4 — Validation, Error Handling, and Configuration

---

## Why This Matters

An API that returns 500 errors with stack traces isn't production-ready.
An API without validation lets bad data corrupt your database.
An API with hardcoded configuration can't move between environments.

This part covers the "boring" infrastructure that separates amateur APIs from professional ones.

---

## Core Concept #5 — Request Validation

### Why This Matters

Validation is your first line of defense against:
- Corrupt data entering your database
- Wasted processing on invalid requests
- Security vulnerabilities (injection attacks)
- Confusing error messages for API consumers

### What Goes Wrong Without This

**Real scenario:** A developer forgets to validate email format. Invalid emails enter the database. The email service fails silently for months. Customer support gets complaints about "never receiving notifications." Root cause? `email: "not-an-email"` in the database.

**Real scenario:** A negative quantity gets through to the order system. The order total becomes negative. The payment system credits the customer instead of charging them.

### DataAnnotations — Quick but Limited

```csharp
public record CreateOrderRequest
{
    [Required(ErrorMessage = "Customer ID is required")]
    public Guid CustomerId { get; init; }

    [Required]
    [MinLength(1, ErrorMessage = "Order must have at least one item")]
    public List<OrderItemRequest> Items { get; init; } = new();

    [Required]
    public AddressRequest ShippingAddress { get; init; } = null!;
}

public record OrderItemRequest
{
    [Required]
    public Guid ProductId { get; init; }

    [Range(1, 1000, ErrorMessage = "Quantity must be between 1 and 1000")]
    public int Quantity { get; init; }
}
```

**Limitations of DataAnnotations:**
- Can't express complex rules (e.g., "if X then Y required")
- Hard to test in isolation
- Validation logic scattered across DTOs
- No async validation

### FluentValidation — The Professional Choice

```csharp
// Install: dotnet add package FluentValidation.DependencyInjectionExtensions

public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    private readonly ICustomerRepository _customerRepo;
    private readonly IProductRepository _productRepo;

    public CreateOrderRequestValidator(
        ICustomerRepository customerRepo,
        IProductRepository productRepo)
    {
        _customerRepo = customerRepo;
        _productRepo = productRepo;

        RuleFor(x => x.CustomerId)
            .NotEmpty()
            .WithMessage("Customer ID is required")
            .MustAsync(CustomerExists)
            .WithMessage("Customer not found");

        RuleFor(x => x.Items)
            .NotEmpty()
            .WithMessage("Order must have at least one item");

        RuleForEach(x => x.Items)
            .SetValidator(new OrderItemRequestValidator(productRepo));

        RuleFor(x => x.ShippingAddress)
            .NotNull()
            .WithMessage("Shipping address is required")
            .SetValidator(new AddressRequestValidator());
    }

    private async Task<bool> CustomerExists(Guid customerId, CancellationToken ct)
    {
        return await _customerRepo.ExistsAsync(customerId, ct);
    }
}

public class OrderItemRequestValidator : AbstractValidator<OrderItemRequest>
{
    private readonly IProductRepository _productRepo;

    public OrderItemRequestValidator(IProductRepository productRepo)
    {
        _productRepo = productRepo;

        RuleFor(x => x.ProductId)
            .NotEmpty()
            .MustAsync(ProductExists)
            .WithMessage("Product not found");

        RuleFor(x => x.Quantity)
            .InclusiveBetween(1, 1000)
            .WithMessage("Quantity must be between 1 and 1000");
    }

    private async Task<bool> ProductExists(Guid productId, CancellationToken ct)
    {
        return await _productRepo.ExistsAsync(productId, ct);
    }
}

public class AddressRequestValidator : AbstractValidator<AddressRequest>
{
    public AddressRequestValidator()
    {
        RuleFor(x => x.Line1)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.City)
            .NotEmpty()
            .MaximumLength(50);

        RuleFor(x => x.PostalCode)
            .NotEmpty()
            .Matches(@"^[A-Z]{1,2}\d[A-Z\d]? ?\d[A-Z]{2}$")
            .When(x => x.Country == "GB")
            .WithMessage("Invalid UK postal code format");

        RuleFor(x => x.PostalCode)
            .NotEmpty()
            .Matches(@"^\d{5}(-\d{4})?$")
            .When(x => x.Country == "US")
            .WithMessage("Invalid US ZIP code format");

        RuleFor(x => x.Country)
            .NotEmpty()
            .Length(2)
            .WithMessage("Country must be a 2-letter ISO code");
    }
}
```

### Registering and Using FluentValidation

```csharp
// Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateOrderRequestValidator>();

// In endpoint
app.MapPost("/orders", async (
    CreateOrderRequest request,
    IValidator<CreateOrderRequest> validator,
    IOrderService service,
    CancellationToken ct) =>
{
    var validationResult = await validator.ValidateAsync(request, ct);

    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }

    var order = await service.CreateAsync(request, ct);
    return Results.Created($"/orders/{order.Id}", order);
});
```

### Automatic Validation with Endpoint Filter

```csharp
// ValidationFilter.cs
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    private readonly IValidator<T> _validator;

    public ValidationFilter(IValidator<T> validator)
    {
        _validator = validator;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var request = context.Arguments.OfType<T>().FirstOrDefault();

        if (request is null)
        {
            return await next(context);
        }

        var result = await _validator.ValidateAsync(request);

        if (!result.IsValid)
        {
            return Results.ValidationProblem(result.ToDictionary());
        }

        return await next(context);
    }
}

// Usage
app.MapPost("/orders", CreateOrder)
    .AddEndpointFilter<ValidationFilter<CreateOrderRequest>>();
```

---

## Core Concept #6 — Global Error Handling

### Why This Matters

Individual try-catch blocks in every endpoint:
- Duplicate error handling code everywhere
- Inconsistent error responses
- Easy to forget somewhere
- Can leak sensitive information

### What Goes Wrong Without This

**Real scenario:** A developer adds try-catch to most endpoints but forgets one. That endpoint throws an unhandled exception in production. The stack trace (including database connection strings) appears in the response. Security incident.

**Real scenario:** Different endpoints return errors in different formats. Frontend developers write three different error parsers. One breaks, nobody notices until customers complain.

### Global Exception Handling Middleware

```csharp
// ErrorHandlingMiddleware.cs
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ErrorHandlingMiddleware> _logger;
    private readonly IHostEnvironment _env;

    public ErrorHandlingMiddleware(
        RequestDelegate next,
        ILogger<ErrorHandlingMiddleware> logger,
        IHostEnvironment env)
    {
        _next = next;
        _logger = logger;
        _env = env;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var correlationId = context.Items["CorrelationId"]?.ToString() ?? "unknown";

        _logger.LogError(
            exception,
            "Unhandled exception. CorrelationId: {CorrelationId}, Path: {Path}",
            correlationId,
            context.Request.Path);

        var (statusCode, problemDetails) = exception switch
        {
            ValidationException validationEx => (
                StatusCodes.Status400BadRequest,
                CreateValidationProblem(validationEx, context)),

            NotFoundException notFoundEx => (
                StatusCodes.Status404NotFound,
                CreateProblem(
                    "Resource not found",
                    notFoundEx.Message,
                    StatusCodes.Status404NotFound,
                    context)),

            ConflictException conflictEx => (
                StatusCodes.Status409Conflict,
                CreateProblem(
                    conflictEx.Title,
                    conflictEx.Message,
                    StatusCodes.Status409Conflict,
                    context)),

            UnauthorizedAccessException => (
                StatusCodes.Status401Unauthorized,
                CreateProblem(
                    "Unauthorized",
                    "Authentication is required",
                    StatusCodes.Status401Unauthorized,
                    context)),

            _ => (
                StatusCodes.Status500InternalServerError,
                CreateInternalErrorProblem(exception, context, correlationId))
        };

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";

        await context.Response.WriteAsJsonAsync(problemDetails);
    }

    private ProblemDetails CreateProblem(
        string title,
        string detail,
        int statusCode,
        HttpContext context)
    {
        return new ProblemDetails
        {
            Type = $"https://api.orderflow.com/errors/{title.ToLowerInvariant().Replace(" ", "-")}",
            Title = title,
            Status = statusCode,
            Detail = detail,
            Instance = context.Request.Path
        };
    }

    private ValidationProblemDetails CreateValidationProblem(
        ValidationException ex,
        HttpContext context)
    {
        var errors = ex.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray());

        return new ValidationProblemDetails(errors)
        {
            Type = "https://api.orderflow.com/errors/validation",
            Title = "Validation failed",
            Status = StatusCodes.Status400BadRequest,
            Instance = context.Request.Path
        };
    }

    private ProblemDetails CreateInternalErrorProblem(
        Exception exception,
        HttpContext context,
        string correlationId)
    {
        var problem = new ProblemDetails
        {
            Type = "https://api.orderflow.com/errors/internal",
            Title = "An unexpected error occurred",
            Status = StatusCodes.Status500InternalServerError,
            Instance = context.Request.Path
        };

        // Add correlation ID for support
        problem.Extensions["traceId"] = correlationId;

        // Only include details in development
        if (_env.IsDevelopment())
        {
            problem.Detail = exception.Message;
            problem.Extensions["stackTrace"] = exception.StackTrace;
        }
        else
        {
            problem.Detail = "Please contact support with the trace ID";
        }

        return problem;
    }
}
```

### Custom Exception Types

```csharp
// Domain exceptions that map to HTTP status codes

public class NotFoundException : Exception
{
    public string ResourceType { get; }
    public string ResourceId { get; }

    public NotFoundException(string resourceType, string resourceId)
        : base($"{resourceType} with ID '{resourceId}' not found")
    {
        ResourceType = resourceType;
        ResourceId = resourceId;
    }
}

public class ConflictException : Exception
{
    public string Title { get; }

    public ConflictException(string title, string message) : base(message)
    {
        Title = title;
    }
}

// Example: Specific business exceptions
public class InsufficientStockException : ConflictException
{
    public Guid ProductId { get; }
    public int Requested { get; }
    public int Available { get; }

    public InsufficientStockException(Guid productId, int requested, int available)
        : base("Insufficient stock",
               $"Product {productId} has {available} units available, but {requested} were requested")
    {
        ProductId = productId;
        Requested = requested;
        Available = available;
    }
}

public class OrderAlreadyShippedException : ConflictException
{
    public OrderAlreadyShippedException(Guid orderId)
        : base("Order already shipped",
               $"Order {orderId} has already been shipped and cannot be modified")
    { }
}
```

### Using Exceptions in Services

```csharp
public class OrderService : IOrderService
{
    public async Task<OrderResponse> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(id, ct);

        if (order is null)
        {
            throw new NotFoundException("Order", id.ToString());
        }

        return MapToResponse(order);
    }

    public async Task CancelOrderAsync(Guid id, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException("Order", id.ToString());

        if (order.Status == OrderStatus.Shipped)
        {
            throw new OrderAlreadyShippedException(id);
        }

        order.Cancel();
        await _repo.UpdateAsync(order, ct);
    }
}
```

---

## Core Concept #7 — Configuration and Options Pattern

### Why This Matters

Hardcoded values mean:
- Can't change settings without redeploying
- Secrets end up in source control
- Different environments need different code branches
- No way to validate configuration at startup

### What Goes Wrong Without This

**Real scenario:** Database connection string is hardcoded. Developer commits with their local SQL Server. Production deployment connects to developer's laptop. Data goes to the wrong place.

**Real scenario:** Feature flags are hardcoded booleans. Enabling a feature requires a new build, PR review, and deployment. What should take 30 seconds takes 3 hours.

### The Options Pattern

```csharp
// Configuration classes
public class OrderFlowSettings
{
    public const string SectionName = "OrderFlow";

    public string DatabaseConnection { get; set; } = string.Empty;
    public int MaxOrderItems { get; set; } = 50;
    public decimal MinimumOrderValue { get; set; } = 10.00m;
}

public class EmailSettings
{
    public const string SectionName = "Email";

    public string SmtpHost { get; set; } = string.Empty;
    public int SmtpPort { get; set; } = 587;
    public string FromAddress { get; set; } = string.Empty;
    public bool UseSsl { get; set; } = true;
}

public class JwtSettings
{
    public const string SectionName = "Jwt";

    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpiryMinutes { get; set; } = 60;
}
```

### Binding Configuration

```json
// appsettings.json
{
  "OrderFlow": {
    "DatabaseConnection": "Server=localhost;Database=OrderFlow;...",
    "MaxOrderItems": 50,
    "MinimumOrderValue": 10.00
  },
  "Email": {
    "SmtpHost": "smtp.sendgrid.net",
    "SmtpPort": 587,
    "FromAddress": "orders@orderflow.com",
    "UseSsl": true
  },
  "Jwt": {
    "Issuer": "orderflow-api",
    "Audience": "orderflow-clients",
    "ExpiryMinutes": 60
  }
}
```

```csharp
// Program.cs
builder.Services.Configure<OrderFlowSettings>(
    builder.Configuration.GetSection(OrderFlowSettings.SectionName));

builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection(EmailSettings.SectionName));

builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName));

// Add validation
builder.Services.AddOptions<OrderFlowSettings>()
    .Bind(builder.Configuration.GetSection(OrderFlowSettings.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();  // Fail fast if invalid
```

### Using Options

```csharp
public class OrderService : IOrderService
{
    private readonly OrderFlowSettings _settings;

    // IOptions<T> — read once at startup, never changes
    public OrderService(IOptions<OrderFlowSettings> options)
    {
        _settings = options.Value;
    }

    // IOptionsSnapshot<T> — re-reads each request (scoped)
    public OrderService(IOptionsSnapshot<OrderFlowSettings> options)
    {
        _settings = options.Value;
    }

    // IOptionsMonitor<T> — notified of changes (singleton-safe)
    public OrderService(IOptionsMonitor<OrderFlowSettings> options)
    {
        _settings = options.CurrentValue;
        options.OnChange(newSettings =>
        {
            // Handle configuration change
        });
    }

    public async Task<Order> CreateAsync(CreateOrderRequest request)
    {
        if (request.Items.Count > _settings.MaxOrderItems)
        {
            throw new ValidationException(
                $"Order cannot have more than {_settings.MaxOrderItems} items");
        }

        // ...
    }
}
```

### Configuration Validation

```csharp
public class OrderFlowSettings
{
    public const string SectionName = "OrderFlow";

    [Required]
    public string DatabaseConnection { get; set; } = string.Empty;

    [Range(1, 1000)]
    public int MaxOrderItems { get; set; } = 50;

    [Range(0.01, 10000)]
    public decimal MinimumOrderValue { get; set; } = 10.00m;
}

// Custom validation
public class OrderFlowSettingsValidator : IValidateOptions<OrderFlowSettings>
{
    public ValidateOptionsResult Validate(string? name, OrderFlowSettings options)
    {
        var errors = new List<string>();

        if (string.IsNullOrWhiteSpace(options.DatabaseConnection))
        {
            errors.Add("DatabaseConnection is required");
        }

        if (options.MaxOrderItems < 1)
        {
            errors.Add("MaxOrderItems must be at least 1");
        }

        if (options.MinimumOrderValue <= 0)
        {
            errors.Add("MinimumOrderValue must be positive");
        }

        return errors.Count > 0
            ? ValidateOptionsResult.Fail(errors)
            : ValidateOptionsResult.Success;
    }
}

// Register
builder.Services.AddSingleton<IValidateOptions<OrderFlowSettings>,
    OrderFlowSettingsValidator>();
```

### Secrets Management

```csharp
// Development: Use User Secrets
// dotnet user-secrets init
// dotnet user-secrets set "Jwt:Secret" "your-secret-key"

// Production: Use environment variables
// JWT__SECRET=your-secret-key

// Azure: Use Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential());

// Never do this:
public class BadExample
{
    private readonly string _jwtSecret = "hardcoded-secret";  // ❌ NO!
}
```

---

## Core Concept #8 — Structured Logging

### Why This Matters

Console.WriteLine debugging doesn't work in production. You need:
- Searchable logs (find all errors for a specific order)
- Correlated logs (trace a request across services)
- Appropriate log levels (don't flood with debug info)
- Structured data (not just strings)

### What Goes Wrong Without This

**Real scenario:** A customer reports an issue with order ORD-12345. Developer searches logs: "Order not found." Which order? When? What user? What server? No idea — logs are unstructured strings.

**Real scenario:** Debug logging is left on in production. Log storage fills up. Alerts fire. Important errors are buried in noise.

### Structured Logging Basics

```csharp
// ❌ BAD — String concatenation, no structure
_logger.LogInformation($"Processing order {orderId} for customer {customerId}");

// ✅ GOOD — Structured with named parameters
_logger.LogInformation(
    "Processing order {OrderId} for customer {CustomerId}",
    orderId,
    customerId);

// Now you can query: OrderId = "ord-123" AND CustomerId = "cust-456"
```

### Log Levels and When to Use Them

```csharp
public class OrderService : IOrderService
{
    public async Task<Order> CreateAsync(CreateOrderRequest request, CancellationToken ct)
    {
        // TRACE: Very detailed, usually off in production
        _logger.LogTrace(
            "Entering CreateAsync with {@Request}",
            request);

        // DEBUG: Useful during development
        _logger.LogDebug(
            "Checking if customer {CustomerId} exists",
            request.CustomerId);

        // INFORMATION: Normal operations worth noting
        _logger.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            request.CustomerId,
            request.Items.Count);

        try
        {
            var order = await CreateOrderInternal(request, ct);

            _logger.LogInformation(
                "Order {OrderId} created successfully. Total: {OrderTotal}",
                order.Id,
                order.Total);

            return order;
        }
        catch (InsufficientStockException ex)
        {
            // WARNING: Something concerning but handled
            _logger.LogWarning(
                ex,
                "Insufficient stock for product {ProductId}. Requested: {Requested}, Available: {Available}",
                ex.ProductId,
                ex.Requested,
                ex.Available);
            throw;
        }
        catch (Exception ex)
        {
            // ERROR: Something failed
            _logger.LogError(
                ex,
                "Failed to create order for customer {CustomerId}",
                request.CustomerId);
            throw;
        }
    }
}
```

### Adding Context with Scopes

```csharp
public class OrderService : IOrderService
{
    public async Task ProcessOrderAsync(Guid orderId, CancellationToken ct)
    {
        // All logs within this scope will include OrderId
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = orderId,
            ["Operation"] = "ProcessOrder"
        }))
        {
            _logger.LogInformation("Starting order processing");

            await ValidateOrder(orderId, ct);
            await ProcessPayment(orderId, ct);
            await ReserveInventory(orderId, ct);
            await SendConfirmation(orderId, ct);

            _logger.LogInformation("Order processing completed");
        }
    }
}
```

### Configuring Serilog

```csharp
// Install packages:
// dotnet add package Serilog.AspNetCore
// dotnet add package Serilog.Sinks.Console
// dotnet add package Serilog.Sinks.Seq
// dotnet add package Serilog.Enrichers.Environment
// dotnet add package Serilog.Enrichers.Thread

// Program.cs
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();

builder.Host.UseSerilog();

// Request logging middleware
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
        diagnosticContext.Set("ClientIP", httpContext.Connection.RemoteIpAddress);
    };
});
```

---

## Hands-On: OrderFlow Validation and Error Handling

### Task 1 — Implement Validators

Create FluentValidation validators for:
- `CreateOrderRequest`
- `UpdateOrderRequest`
- `CreateCustomerRequest`
- `AddressRequest`

Include async validation (customer exists, product exists).

### Task 2 — Implement Error Handling Middleware

Create middleware that:
- Catches all exceptions
- Maps to appropriate status codes
- Returns ProblemDetails format
- Includes correlation IDs
- Hides stack traces in production

### Task 3 — Add Configuration

Create settings classes for:
- Database connection
- JWT configuration
- Email settings
- Rate limiting

Use validation to fail fast on startup.

### Task 4 — Implement Structured Logging

- Configure Serilog
- Add request logging middleware
- Log important business events
- Add correlation ID enrichment

---

## Deliverables

1. **FluentValidation validators** for all request types
2. **Error handling middleware** with ProblemDetails
3. **Custom exception types** for business errors
4. **Configuration classes** with validation
5. **Serilog configuration** with structured logging

---

## Resources

### Must-Read
- [FluentValidation Documentation](https://docs.fluentvalidation.net/)
- [ASP.NET Core Error Handling](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [Options Pattern](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options)

### Videos
- [Nick Chapsas: FluentValidation](https://www.youtube.com/watch?v=TgSOTnE4EKQ)
- [Milan Jovanovic: Global Error Handling](https://www.youtube.com/watch?v=4NfflZilTvk)
- [Nick Chapsas: Serilog Best Practices](https://www.youtube.com/watch?v=MYKTNvNKWs8)
- [Raw Coding: Options Pattern Deep Dive](https://www.youtube.com/watch?v=v14vB7nVGD8)

### Tools
- [Seq](https://datalust.co/seq) — Log aggregation (free for development)
- [Serilog](https://serilog.net/) — Structured logging
- [FluentValidation](https://fluentvalidation.net/)

---

## Reflection Questions

1. Why use FluentValidation instead of DataAnnotations?
2. When should validation happen in the service layer vs endpoint?
3. What information should never appear in error responses?
4. Why is structured logging better than string concatenation?

---

**Next:** [Part 3 — Data Access, Files, and Background Jobs](./part-3.md)
