# 03 — Building Web APIs with ASP.NET Core

## Part 4 / 4 — Authentication, Testing, and Production Readiness

---

## Why This Matters

An API without authentication is a public API.
An API without tests is a liability.
An API without documentation is unusable.
An API without observability is a black box.

This part covers everything needed to ship a production API.

---

## Core Concept #13 — JWT Authentication

### Why This Matters

Authentication answers: "Who is this?"
Authorization answers: "Can they do this?"

Most modern APIs use JWT (JSON Web Tokens) for stateless authentication.

### What Goes Wrong Without This

**Real scenario:** API key is passed in query string. It appears in server logs, browser history, and referer headers. Third parties see the key.

**Real scenario:** JWT is never validated on the server. Anyone can forge tokens. All data becomes accessible to attackers.

### JWT Setup in ASP.NET Core

```csharp
// Install: dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]!)),
            ClockSkew = TimeSpan.Zero  // No tolerance for expired tokens
        };

        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                if (context.Exception is SecurityTokenExpiredException)
                {
                    context.Response.Headers.Append("Token-Expired", "true");
                }
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

// Pipeline order matters!
app.UseAuthentication();
app.UseAuthorization();
```

### Token Generation

```csharp
public class TokenService : ITokenService
{
    private readonly JwtSettings _settings;
    private readonly ILogger<TokenService> _logger;

    public TokenService(IOptions<JwtSettings> settings, ILogger<TokenService> logger)
    {
        _settings = settings.Value;
        _logger = logger;
    }

    public string GenerateToken(User user, IEnumerable<string> roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new("customerId", user.CustomerId?.ToString() ?? ""),
        };

        claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_settings.ExpiryMinutes),
            signingCredentials: credentials);

        _logger.LogInformation("Generated token for user {UserId}", user.Id);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### Protecting Endpoints

```csharp
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization();  // All endpoints require auth

        group.MapGet("/", GetOrders);
        group.MapGet("/{id:guid}", GetOrderById);
        group.MapPost("/", CreateOrder);

        // Admin only
        group.MapDelete("/{id:guid}", DeleteOrder)
            .RequireAuthorization("AdminOnly");
    }
}

// Authorization policies
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("CanManageOrders", policy =>
        policy.RequireClaim("permission", "orders:write"));

    // Resource-based authorization
    options.AddPolicy("OrderOwner", policy =>
        policy.Requirements.Add(new OrderOwnerRequirement()));
});
```

### Resource-Based Authorization

```csharp
// Requirement
public class OrderOwnerRequirement : IAuthorizationRequirement { }

// Handler
public class OrderOwnerHandler : AuthorizationHandler<OrderOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderOwnerRequirement requirement,
        Order order)
    {
        var userIdClaim = context.User.FindFirst(ClaimTypes.NameIdentifier);
        var customerIdClaim = context.User.FindFirst("customerId");

        if (userIdClaim is null || customerIdClaim is null)
        {
            return Task.CompletedTask;  // Not authorized
        }

        // User owns the order OR is admin
        if (order.CustomerId.ToString() == customerIdClaim.Value ||
            context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Usage in endpoint
private static async Task<IResult> GetOrderById(
    Guid id,
    IOrderService service,
    IAuthorizationService authService,
    ClaimsPrincipal user,
    CancellationToken ct)
{
    var order = await service.GetByIdAsync(id, ct);
    if (order is null) return Results.NotFound();

    var authResult = await authService.AuthorizeAsync(user, order, "OrderOwner");
    if (!authResult.Succeeded)
    {
        return Results.Forbid();
    }

    return Results.Ok(order);
}
```

---

## Core Concept #14 — Integration Testing

### Why This Matters

Unit tests verify individual components. Integration tests verify they work together:
- Database queries actually work
- Validation rejects bad data
- Auth rejects unauthorized requests
- Endpoints return correct responses

### What Goes Wrong Without This

**Real scenario:** All unit tests pass. Deployment happens. Nothing works because the DI container is misconfigured — something unit tests with mocks never catch.

**Real scenario:** Endpoint works locally with SQLite. Production uses SQL Server. Queries fail because of syntax differences. No integration tests caught it.

### WebApplicationFactory Setup

```csharp
// Install: dotnet add package Microsoft.AspNetCore.Mvc.Testing

// OrderFlowApiFactory.cs
public class OrderFlowApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Replace database with test database
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<OrderFlowDbContext>));

            if (descriptor != null)
            {
                services.Remove(descriptor);
            }

            services.AddDbContext<OrderFlowDbContext>(options =>
            {
                options.UseSqlite("DataSource=:memory:");
            });

            // Replace external services with fakes
            services.AddScoped<IEmailService, FakeEmailService>();
            services.AddScoped<IPaymentGateway, FakePaymentGateway>();

            // Ensure database is created
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();
            db.Database.OpenConnection();
            db.Database.EnsureCreated();
        });
    }

    public HttpClient CreateAuthenticatedClient(string userId, string[] roles)
    {
        var client = CreateClient();
        var token = GenerateTestToken(userId, roles);
        client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
        return client;
    }

    private string GenerateTestToken(string userId, string[] roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId),
            new("customerId", "test-customer"),
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes("test-secret-key-that-is-at-least-32-characters"));

        var token = new JwtSecurityToken(
            issuer: "test",
            audience: "test",
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### Integration Tests

```csharp
public class OrderEndpointTests : IClassFixture<OrderFlowApiFactory>
{
    private readonly OrderFlowApiFactory _factory;
    private readonly HttpClient _client;

    public OrderEndpointTests(OrderFlowApiFactory factory)
    {
        _factory = factory;
        _client = factory.CreateAuthenticatedClient("user-123", new[] { "User" });
    }

    [Fact]
    public async Task GetOrders_ReturnsPagedResult()
    {
        // Act
        var response = await _client.GetAsync("/orders?page=1&pageSize=10");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var result = await response.Content
            .ReadFromJsonAsync<PagedResponse<OrderResponse>>();

        result.Should().NotBeNull();
        result!.Pagination.Page.Should().Be(1);
        result.Pagination.PageSize.Should().Be(10);
    }

    [Fact]
    public async Task CreateOrder_WithValidData_ReturnsCreated()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = Guid.NewGuid(),
            Items = new List<OrderItemRequest>
            {
                new() { ProductId = Guid.NewGuid(), Quantity = 2 }
            },
            ShippingAddress = new AddressRequest
            {
                Line1 = "123 Test St",
                City = "London",
                PostalCode = "SW1A 1AA",
                Country = "GB"
            }
        };

        // Act
        var response = await _client.PostAsJsonAsync("/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
        order.Should().NotBeNull();
        order!.Status.Should().Be("Pending");
    }

    [Fact]
    public async Task CreateOrder_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = Guid.Empty,  // Invalid
            Items = new List<OrderItemRequest>()  // Empty
        };

        // Act
        var response = await _client.PostAsJsonAsync("/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);

        var problem = await response.Content
            .ReadFromJsonAsync<ValidationProblemDetails>();

        problem.Should().NotBeNull();
        problem!.Errors.Should().ContainKey("CustomerId");
        problem.Errors.Should().ContainKey("Items");
    }

    [Fact]
    public async Task GetOrder_Unauthorized_Returns401()
    {
        // Arrange - use unauthenticated client
        var client = _factory.CreateClient();

        // Act
        var response = await client.GetAsync("/orders");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task DeleteOrder_AsNonAdmin_Returns403()
    {
        // Arrange
        var client = _factory.CreateAuthenticatedClient("user-123", new[] { "User" });

        // Act
        var response = await client.DeleteAsync($"/orders/{Guid.NewGuid()}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    [Fact]
    public async Task DeleteOrder_AsAdmin_Succeeds()
    {
        // Arrange
        var client = _factory.CreateAuthenticatedClient("admin-123", new[] { "Admin" });
        // First create an order to delete...

        // Act
        var response = await client.DeleteAsync($"/orders/{orderId}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);
    }
}
```

### Testing with Testcontainers (Real Database)

```csharp
// Install: dotnet add package Testcontainers.MsSql

public class SqlServerOrderFlowApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();
    }

    public new async Task DisposeAsync()
    {
        await _sqlContainer.DisposeAsync();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<OrderFlowDbContext>));

            if (descriptor != null)
                services.Remove(descriptor);

            services.AddDbContext<OrderFlowDbContext>(options =>
            {
                options.UseSqlServer(_sqlContainer.GetConnectionString());
            });
        });
    }
}
```

---

## Core Concept #15 — OpenAPI Documentation

### Why This Matters

Documentation is how clients know what to send and what to expect. Without it:
- Frontend developers guess at request formats
- Integration partners spend hours in trial and error
- Breaking changes go unnoticed

### Setting Up Swagger/OpenAPI

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "OrderFlow API",
        Version = "v1",
        Description = "B2B order management API",
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "api-support@orderflow.com"
        }
    });

    // Add JWT auth to Swagger
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });

    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    if (File.Exists(xmlPath))
    {
        options.IncludeXmlComments(xmlPath);
    }
});

// Enable in pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "OrderFlow API v1");
        options.RoutePrefix = string.Empty;  // Swagger at root
    });
}
```

### Documenting Endpoints

```csharp
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders")
            .WithOpenApi();

        group.MapGet("/", GetOrders)
            .WithName("GetOrders")
            .WithSummary("List all orders")
            .WithDescription("Returns a paginated list of orders for the authenticated user")
            .Produces<PagedResponse<OrderResponse>>(200)
            .ProducesProblem(401)
            .ProducesProblem(500);

        group.MapPost("/", CreateOrder)
            .WithName("CreateOrder")
            .WithSummary("Create a new order")
            .WithDescription("Creates a new order and returns the created resource")
            .Accepts<CreateOrderRequest>("application/json")
            .Produces<OrderResponse>(201)
            .ProducesValidationProblem()
            .ProducesProblem(401)
            .ProducesProblem(409);

        group.MapGet("/{id:guid}", GetOrderById)
            .WithName("GetOrderById")
            .WithSummary("Get order by ID")
            .Produces<OrderResponse>(200)
            .ProducesProblem(404)
            .ProducesProblem(401)
            .ProducesProblem(403);
    }
}
```

---

## Core Concept #16 — Health Checks and Observability

### Why This Matters

In production, you need to know:
- Is the API healthy?
- Can it reach its dependencies?
- How fast is it responding?
- What's happening right now?

### Health Checks

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<OrderFlowDbContext>(
        name: "database",
        tags: new[] { "db", "sql" })
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: new[] { "cache" })
    .AddUrlGroup(
        new Uri("https://api.stripe.com/v1/health"),
        name: "stripe",
        tags: new[] { "external" });

// Endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Just checks if app responds
});
```

### Metrics with OpenTelemetry

```csharp
// Install: dotnet add package OpenTelemetry.Extensions.Hosting
// Install: dotnet add package OpenTelemetry.Instrumentation.AspNetCore
// Install: dotnet add package OpenTelemetry.Instrumentation.Http
// Install: dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddAspNetCoreInstrumentation()
               .AddHttpClientInstrumentation()
               .AddRuntimeInstrumentation()
               .AddPrometheusExporter();
    })
    .WithTracing(tracing =>
    {
        tracing.AddAspNetCoreInstrumentation()
               .AddHttpClientInstrumentation()
               .AddEntityFrameworkCoreInstrumentation()
               .AddOtlpExporter();
    });

app.MapPrometheusScrapingEndpoint();  // /metrics endpoint
```

### Custom Metrics

```csharp
public class OrderMetrics
{
    private readonly Counter<long> _ordersCreated;
    private readonly Histogram<double> _orderTotal;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("OrderFlow.Orders");

        _ordersCreated = meter.CreateCounter<long>(
            "orders.created",
            description: "Number of orders created");

        _orderTotal = meter.CreateHistogram<double>(
            "orders.total",
            unit: "USD",
            description: "Order total amount distribution");
    }

    public void OrderCreated(decimal total, string customerType)
    {
        _ordersCreated.Add(1, new KeyValuePair<string, object?>("customer_type", customerType));
        _orderTotal.Record((double)total);
    }
}

// Usage in service
public class OrderService : IOrderService
{
    private readonly OrderMetrics _metrics;

    public async Task<OrderResponse> CreateAsync(CreateOrderRequest request, CancellationToken ct)
    {
        var order = await CreateOrderInternal(request, ct);
        _metrics.OrderCreated(order.Total.Amount, order.Customer.Type);
        return MapToResponse(order);
    }
}
```

---

## Core Concept #17 — Deployment Checklist

### Production Configuration

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Environment-specific configuration
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables()
    .AddUserSecrets<Program>(optional: true);

// Use Azure Key Vault in production
if (builder.Environment.IsProduction())
{
    var keyVaultUrl = builder.Configuration["KeyVault:Url"];
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUrl!),
        new DefaultAzureCredential());
}

var app = builder.Build();

// Security headers
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");
    await next();
});

// HTTPS in production
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
    app.UseHsts();
}
```

### Dockerfile

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj and restore
COPY ["src/OrderFlow.Api/OrderFlow.Api.csproj", "OrderFlow.Api/"]
COPY ["src/OrderFlow.Application/OrderFlow.Application.csproj", "OrderFlow.Application/"]
COPY ["src/OrderFlow.Domain/OrderFlow.Domain.csproj", "OrderFlow.Domain/"]
COPY ["src/OrderFlow.Infrastructure/OrderFlow.Infrastructure.csproj", "OrderFlow.Infrastructure/"]
RUN dotnet restore "OrderFlow.Api/OrderFlow.Api.csproj"

# Copy everything and build
COPY src/ .
RUN dotnet publish "OrderFlow.Api/OrderFlow.Api.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

# Security: Run as non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

COPY --from=build /app/publish .

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl --fail http://localhost:8080/health/live || exit 1

ENTRYPOINT ["dotnet", "OrderFlow.Api.dll"]
```

### Production Checklist

```markdown
## Pre-Launch Checklist

### Security
- [ ] All secrets in Key Vault/environment variables (not in code)
- [ ] JWT secret is strong (32+ characters, random)
- [ ] HTTPS enforced in production
- [ ] CORS configured correctly (not allowing *)
- [ ] Rate limiting enabled
- [ ] Input validation on all endpoints
- [ ] No stack traces in production error responses

### Reliability
- [ ] Health checks configured
- [ ] Connection pooling configured
- [ ] Retry policies on external HTTP calls
- [ ] Circuit breakers on external dependencies
- [ ] Graceful shutdown handling

### Observability
- [ ] Structured logging configured
- [ ] Log levels appropriate for production
- [ ] Metrics exposed (/metrics endpoint)
- [ ] Distributed tracing enabled
- [ ] Alerts configured for error rates

### Documentation
- [ ] OpenAPI spec up to date
- [ ] README with setup instructions
- [ ] Deployment runbook documented
- [ ] API changelog maintained

### Testing
- [ ] Integration tests pass
- [ ] Load testing completed
- [ ] Security scan completed
- [ ] API contract tests pass
```

---

## Module Summary

After completing Module 03, you can:

| Skill | Evidence |
|-------|----------|
| Build production APIs | OrderFlow implementation |
| Handle authentication | JWT with role-based access |
| Write integration tests | WebApplicationFactory tests |
| Document APIs | OpenAPI specification |
| Deploy with confidence | Docker + health checks |

Your OrderFlow API is now production-ready for Module 04 (EF Core).

---

## Deliverables

1. **Complete OrderFlow API** with all CRUD endpoints
2. **JWT authentication** with role-based authorization
3. **Integration test suite** with 80%+ coverage
4. **OpenAPI documentation** with examples
5. **Dockerfile** with health checks
6. **Production checklist** completed

---

## Resources

### Must-Read
- [ASP.NET Core Security](https://learn.microsoft.com/en-us/aspnet/core/security/)
- [Integration Testing](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)
- [Health Checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)

### Videos
- [Nick Chapsas: JWT Authentication](https://www.youtube.com/watch?v=6sMPvucWNRU)
- [Nick Chapsas: Integration Testing](https://www.youtube.com/watch?v=tj5ZCtvgXKY)
- [Milan Jovanovic: Health Checks](https://www.youtube.com/watch?v=p2faw9DCSsY)
- [Raw Coding: OpenTelemetry](https://www.youtube.com/watch?v=qiBlYdCvnSQ)

### Tools
- [Testcontainers](https://testcontainers.com/) — Real dependencies in tests
- [FluentAssertions](https://fluentassertions.com/) — Better test assertions
- [Swagger UI](https://swagger.io/tools/swagger-ui/) — API documentation
- [Prometheus](https://prometheus.io/) — Metrics collection

### Reference
- [eShop Reference Application](https://github.com/dotnet/eshop)
- [Clean Architecture Template](https://github.com/jasontaylordev/CleanArchitecture)

---

## Reflection Questions

1. Why is JWT stateless, and what are the implications?
2. What's the difference between authentication and authorization?
3. Why use WebApplicationFactory instead of mocking everything?
4. What should health checks verify vs what should they skip?

---

**Next:** [Module 04 — Entity Framework Core](../04-ef-core/part-1.md)
