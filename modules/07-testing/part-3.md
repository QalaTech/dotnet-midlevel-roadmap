# 07 — Testing Mastery: Unit, Integration, Contract

## Part 3 / 4 — Integration Testing

---

## Why This Matters

Unit tests verify pieces in isolation. But:
- Does DI wire everything correctly?
- Does EF Core generate the right SQL?
- Does the middleware pipeline work?
- Does serialization round-trip correctly?

Integration tests catch the bugs that unit tests miss — the ones where each piece works alone but fails together.

---

## Core Concept #11 — WebApplicationFactory

### Why This Matters

`WebApplicationFactory` boots your entire ASP.NET Core app in-memory. You get:
- Real DI container
- Real middleware pipeline
- Real routing
- Real serialization

But with test-friendly infrastructure (databases, caches, message brokers).

### What Goes Wrong Without This

**War story — The Serialization Bug:**
Unit tests mocked everything. All passed. In production, API returned `{"firstName": null}` because the property was `FirstName` and JSON serialization used camelCase. No integration test caught it.

**War story — The Middleware Gap:**
Authentication middleware was configured wrong. Unit tests mocked auth entirely. Integration tests would have caught `401 Unauthorized` on first request.

### Basic Setup

```csharp
// Custom factory for OrderFlow
public class OrderFlowWebAppFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real database
            var dbDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<OrderFlowDbContext>));
            if (dbDescriptor != null)
                services.Remove(dbDescriptor);

            // Add test database
            services.AddDbContext<OrderFlowDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb_" + Guid.NewGuid());
            });

            // Replace external services
            services.RemoveAll<IPaymentGateway>();
            services.AddSingleton<IPaymentGateway, FakePaymentGateway>();

            services.RemoveAll<IEmailService>();
            services.AddSingleton<IEmailService, FakeEmailService>();
        });

        builder.UseEnvironment("Testing");
    }
}

// Basic integration test
public class OrdersApiTests : IClassFixture<OrderFlowWebAppFactory>
{
    private readonly HttpClient _client;
    private readonly OrderFlowWebAppFactory _factory;

    public OrdersApiTests(OrderFlowWebAppFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Create_order_returns_201()
    {
        var response = await _client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = Guid.NewGuid(),
            Items = new[]
            {
                new { ProductId = Guid.NewGuid(), Quantity = 2, Price = 50.00m }
            }
        });

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task Get_nonexistent_order_returns_404()
    {
        var response = await _client.GetAsync($"/api/orders/{Guid.NewGuid()}");

        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

### Accessing Services in Tests

```csharp
public class OrdersApiTests : IClassFixture<OrderFlowWebAppFactory>
{
    private readonly OrderFlowWebAppFactory _factory;
    private readonly HttpClient _client;

    public OrdersApiTests(OrderFlowWebAppFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Create_order_persists_to_database()
    {
        var customerId = Guid.NewGuid();

        // Create order via API
        var response = await _client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = customerId,
            Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 1, Price = 100m } }
        });

        response.EnsureSuccessStatusCode();

        // Verify in database
        using var scope = _factory.Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();

        var order = await dbContext.Orders
            .FirstOrDefaultAsync(o => o.CustomerId == customerId);

        order.Should().NotBeNull();
        order!.Items.Should().HaveCount(1);
    }

    [Fact]
    public async Task Create_order_sends_confirmation_email()
    {
        var customerId = Guid.NewGuid();

        await _client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = customerId,
            Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 1, Price = 100m } }
        });

        // Check fake email service
        using var scope = _factory.Services.CreateScope();
        var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>()
            as FakeEmailService;

        emailService!.SentEmails.Should().Contain(e =>
            e.Subject.Contains("Order Confirmation"));
    }
}
```

---

## Core Concept #12 — Testcontainers

### Why This Matters

In-memory databases behave differently from real ones:
- SQLite doesn't support all SQL Server features
- In-memory doesn't have transactions like the real DB
- Query plans differ significantly

Testcontainers run real databases in Docker for true integration testing.

### What Goes Wrong Without This

**War story — The Index Bug:**
Query worked fine in SQLite tests. In production PostgreSQL, it timed out. The query plan was completely different. Real database testing would have caught it.

**War story — The Transaction Isolation:**
Test passed with in-memory database. Production showed race condition. In-memory doesn't implement real transaction isolation levels.

### Setting Up Testcontainers

```csharp
public class PostgresContainerFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .WithDatabase("orderflow_test")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}

// Factory using real Postgres
public class OrderFlowIntegrationFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .Build();

    private readonly RedisContainer _redis = new RedisBuilder()
        .WithImage("redis:7")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Replace database
            services.RemoveAll<DbContextOptions<OrderFlowDbContext>>();
            services.AddDbContext<OrderFlowDbContext>(options =>
            {
                options.UseNpgsql(_postgres.GetConnectionString());
            });

            // Replace Redis
            services.RemoveAll<IConnectionMultiplexer>();
            services.AddSingleton<IConnectionMultiplexer>(_ =>
                ConnectionMultiplexer.Connect(_redis.GetConnectionString()));
        });
    }

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
        await _redis.StartAsync();

        // Apply migrations
        using var scope = Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();
        await dbContext.Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await _postgres.DisposeAsync();
        await _redis.DisposeAsync();
    }
}

// Tests with real dependencies
public class OrdersIntegrationTests : IClassFixture<OrderFlowIntegrationFactory>
{
    private readonly HttpClient _client;
    private readonly OrderFlowIntegrationFactory _factory;

    public OrdersIntegrationTests(OrderFlowIntegrationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Order_persisted_and_retrieved()
    {
        // Create
        var createResponse = await _client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = Guid.NewGuid(),
            Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 2, Price = 100m } }
        });
        createResponse.EnsureSuccessStatusCode();

        var location = createResponse.Headers.Location!.ToString();

        // Retrieve
        var getResponse = await _client.GetAsync(location);
        getResponse.EnsureSuccessStatusCode();

        var order = await getResponse.Content.ReadFromJsonAsync<OrderResponse>();
        order!.Total.Should().Be(200m);
    }
}
```

### Collection Fixtures for Shared Containers

```csharp
// Shared container across all tests in collection
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<PostgresContainerFixture>
{
}

[Collection("Database")]
public class OrderRepositoryTests
{
    private readonly PostgresContainerFixture _fixture;

    public OrderRepositoryTests(PostgresContainerFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task Saves_and_retrieves_order()
    {
        var options = new DbContextOptionsBuilder<OrderFlowDbContext>()
            .UseNpgsql(_fixture.ConnectionString)
            .Options;

        await using var context = new OrderFlowDbContext(options);
        await context.Database.EnsureCreatedAsync();

        var order = new Order(Guid.NewGuid());
        order.AddItem(Guid.NewGuid(), 2, 50m);

        context.Orders.Add(order);
        await context.SaveChangesAsync();

        var retrieved = await context.Orders
            .Include(o => o.Items)
            .FirstAsync(o => o.Id == order.Id);

        retrieved.Items.Should().HaveCount(1);
        retrieved.Total.Should().Be(100m);
    }
}
```

---

## Core Concept #13 — Database Testing Patterns

### Why This Matters

Database tests have unique challenges:
- Test isolation (tests shouldn't affect each other)
- Data setup (creating valid test data)
- Schema management (migrations vs recreate)

### Test Isolation Strategies

```csharp
// Strategy 1: Transaction rollback per test
public class TransactionRollbackTests : IAsyncLifetime
{
    private readonly OrderFlowDbContext _context;
    private IDbContextTransaction _transaction = null!;

    public TransactionRollbackTests(OrderFlowDbContext context)
    {
        _context = context;
    }

    public async Task InitializeAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task DisposeAsync()
    {
        await _transaction.RollbackAsync();
        await _transaction.DisposeAsync();
    }

    [Fact]
    public async Task Test_that_modifies_data()
    {
        // Changes are rolled back after test
        var order = new Order(Guid.NewGuid());
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();

        // Verify
        (await _context.Orders.CountAsync()).Should().BeGreaterThan(0);
    }
}

// Strategy 2: Database per test
public class DatabasePerTestFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<OrderFlowDbContext>>();
            services.AddDbContext<OrderFlowDbContext>(options =>
            {
                // Unique database per test run
                var dbName = $"OrderFlow_Test_{Guid.NewGuid():N}";
                options.UseInMemoryDatabase(dbName);
            });
        });
    }
}

// Strategy 3: Respawn for fast cleanup
public class RespawnTests : IAsyncLifetime
{
    private readonly string _connectionString;
    private Respawner _respawner = null!;

    public RespawnTests(PostgresContainerFixture fixture)
    {
        _connectionString = fixture.ConnectionString;
    }

    public async Task InitializeAsync()
    {
        _respawner = await Respawner.CreateAsync(_connectionString, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            TablesToIgnore = new[] { "__EFMigrationsHistory" }
        });
    }

    public async Task DisposeAsync()
    {
        // Clean all data between tests
        await _respawner.ResetAsync(_connectionString);
    }

    [Fact]
    public async Task Test_with_clean_database()
    {
        // Database is clean for each test
    }
}
```

### Seeding Test Data

```csharp
public static class TestDataSeeder
{
    public static async Task<Customer> SeedCustomerAsync(
        OrderFlowDbContext context,
        string? email = null)
    {
        var customer = new Customer
        {
            Id = Guid.NewGuid(),
            Email = email ?? $"customer-{Guid.NewGuid():N}@test.com",
            Name = "Test Customer"
        };

        context.Customers.Add(customer);
        await context.SaveChangesAsync();
        return customer;
    }

    public static async Task<Product> SeedProductAsync(
        OrderFlowDbContext context,
        decimal price = 100m,
        int stock = 100)
    {
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = $"Test Product {Guid.NewGuid():N}",
            Price = price,
            StockQuantity = stock
        };

        context.Products.Add(product);
        await context.SaveChangesAsync();
        return product;
    }

    public static async Task<Order> SeedOrderAsync(
        OrderFlowDbContext context,
        Customer? customer = null,
        OrderStatus status = OrderStatus.Pending)
    {
        customer ??= await SeedCustomerAsync(context);
        var product = await SeedProductAsync(context);

        var order = new Order(customer.Id);
        order.AddItem(product.Id, 1, product.Price);

        if (status >= OrderStatus.Paid)
        {
            order.MarkAsPaid();
        }
        if (status >= OrderStatus.Shipped)
        {
            order.Ship();
        }

        context.Orders.Add(order);
        await context.SaveChangesAsync();
        return order;
    }
}

// Usage in tests
public class OrderQueriesTests : IClassFixture<OrderFlowIntegrationFactory>
{
    [Fact]
    public async Task Get_orders_by_status()
    {
        using var scope = _factory.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();

        // Seed test data
        await TestDataSeeder.SeedOrderAsync(context, status: OrderStatus.Pending);
        await TestDataSeeder.SeedOrderAsync(context, status: OrderStatus.Pending);
        await TestDataSeeder.SeedOrderAsync(context, status: OrderStatus.Shipped);

        // Test query
        var pendingOrders = await context.Orders
            .Where(o => o.Status == OrderStatus.Pending)
            .ToListAsync();

        pendingOrders.Should().HaveCount(2);
    }
}
```

---

## Core Concept #14 — Testing Authentication

### Why This Matters

Most APIs require authentication. Tests need to:
- Test protected endpoints
- Test authorization rules
- Simulate different users/roles

### Authentication in Tests

```csharp
// Custom authentication handler for tests
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public const string SchemeName = "Test";
    public const string TestUserId = "test-user-id";

    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder)
    {
    }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // Check for test auth header
        if (!Request.Headers.TryGetValue("X-Test-User", out var userIdValues))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        var userId = userIdValues.FirstOrDefault() ?? TestUserId;

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId),
            new Claim(ClaimTypes.Name, "Test User"),
            new Claim(ClaimTypes.Email, "test@example.com"),
        };

        // Check for roles header
        if (Request.Headers.TryGetValue("X-Test-Roles", out var rolesValue))
        {
            var roles = rolesValue.ToString().Split(',');
            claims = claims.Concat(
                roles.Select(r => new Claim(ClaimTypes.Role, r.Trim()))
            ).ToArray();
        }

        var identity = new ClaimsIdentity(claims, SchemeName);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, SchemeName);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}

// Configure in test factory
public class AuthenticatedApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.AddAuthentication(TestAuthHandler.SchemeName)
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                    TestAuthHandler.SchemeName, _ => { });
        });
    }
}

// Usage in tests
public class ProtectedEndpointTests : IClassFixture<AuthenticatedApiFactory>
{
    private readonly HttpClient _client;

    public ProtectedEndpointTests(AuthenticatedApiFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Unauthenticated_request_returns_401()
    {
        var response = await _client.GetAsync("/api/orders");

        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task Authenticated_request_succeeds()
    {
        _client.DefaultRequestHeaders.Add("X-Test-User", "user-123");

        var response = await _client.GetAsync("/api/orders");

        response.Should().BeSuccessful();
    }

    [Fact]
    public async Task Admin_endpoint_requires_admin_role()
    {
        // Regular user
        _client.DefaultRequestHeaders.Add("X-Test-User", "user-123");
        var response = await _client.DeleteAsync("/api/admin/orders/123");
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);

        // Admin user
        _client.DefaultRequestHeaders.Clear();
        _client.DefaultRequestHeaders.Add("X-Test-User", "admin-123");
        _client.DefaultRequestHeaders.Add("X-Test-Roles", "Admin");

        var adminResponse = await _client.DeleteAsync("/api/admin/orders/123");
        adminResponse.StatusCode.Should().NotBe(HttpStatusCode.Forbidden);
    }
}

// Extension method for authenticated requests
public static class HttpClientAuthExtensions
{
    public static HttpClient WithUser(this HttpClient client, string userId, params string[] roles)
    {
        client.DefaultRequestHeaders.Remove("X-Test-User");
        client.DefaultRequestHeaders.Remove("X-Test-Roles");

        client.DefaultRequestHeaders.Add("X-Test-User", userId);
        if (roles.Any())
        {
            client.DefaultRequestHeaders.Add("X-Test-Roles", string.Join(",", roles));
        }

        return client;
    }
}

// Cleaner usage
[Fact]
public async Task User_can_only_see_own_orders()
{
    var client = _factory.CreateClient().WithUser("user-123");

    var response = await client.GetAsync("/api/orders");

    var orders = await response.Content.ReadFromJsonAsync<List<OrderResponse>>();
    orders.Should().AllSatisfy(o => o.CustomerId.Should().Be("user-123"));
}
```

---

## Core Concept #15 — Testing Message Handlers

### Why This Matters

Message handlers run outside HTTP context. They need integration tests too:
- Consumer processes messages correctly
- Retry policies work
- Dead letter handling works

### Testing MassTransit Consumers

```csharp
// MassTransit test harness
public class OrderPlacedConsumerTests
{
    [Fact]
    public async Task Processes_order_placed_event()
    {
        await using var provider = new ServiceCollection()
            .AddMassTransitTestHarness(cfg =>
            {
                cfg.AddConsumer<OrderPlacedConsumer>();
            })
            .AddSingleton<IEmailService, FakeEmailService>()
            .BuildServiceProvider(true);

        var harness = provider.GetRequiredService<ITestHarness>();
        await harness.Start();

        // Publish event
        await harness.Bus.Publish(new OrderPlacedEvent
        {
            OrderId = Guid.NewGuid(),
            CustomerEmail = "test@example.com",
            Total = 150m
        });

        // Wait for consumer
        var consumerHarness = harness.GetConsumerHarness<OrderPlacedConsumer>();
        (await consumerHarness.Consumed.Any<OrderPlacedEvent>()).Should().BeTrue();

        // Verify side effects
        var emailService = provider.GetRequiredService<IEmailService>() as FakeEmailService;
        emailService!.SentEmails.Should().ContainSingle();
    }

    [Fact]
    public async Task Handles_duplicate_messages_idempotently()
    {
        await using var provider = new ServiceCollection()
            .AddMassTransitTestHarness(cfg =>
            {
                cfg.AddConsumer<OrderPlacedConsumer>();
            })
            .AddSingleton<IProcessedMessageRepository, FakeProcessedMessageRepository>()
            .AddSingleton<IEmailService, FakeEmailService>()
            .BuildServiceProvider(true);

        var harness = provider.GetRequiredService<ITestHarness>();
        await harness.Start();

        var eventId = Guid.NewGuid();
        var @event = new OrderPlacedEvent
        {
            EventId = eventId,
            OrderId = Guid.NewGuid(),
            CustomerEmail = "test@example.com"
        };

        // Publish twice
        await harness.Bus.Publish(@event);
        await harness.Bus.Publish(@event);

        // Wait for processing
        await Task.Delay(500);

        // Should only send one email
        var emailService = provider.GetRequiredService<IEmailService>() as FakeEmailService;
        emailService!.SentEmails.Should().HaveCount(1);
    }
}

// Full integration with RabbitMQ container
public class MessageIntegrationTests : IClassFixture<RabbitMqContainerFixture>
{
    private readonly RabbitMqContainerFixture _fixture;

    public MessageIntegrationTests(RabbitMqContainerFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task Message_flows_through_real_broker()
    {
        var services = new ServiceCollection();

        services.AddMassTransit(x =>
        {
            x.AddConsumer<TestConsumer>();

            x.UsingRabbitMq((context, cfg) =>
            {
                cfg.Host(_fixture.Host, _fixture.Port, "/", h =>
                {
                    h.Username(_fixture.Username);
                    h.Password(_fixture.Password);
                });

                cfg.ConfigureEndpoints(context);
            });
        });

        await using var provider = services.BuildServiceProvider(true);
        var bus = provider.GetRequiredService<IBus>();

        // Wait for bus to start
        await Task.Delay(1000);

        // Publish and verify
        await bus.Publish(new TestMessage { Value = "Hello" });

        // Wait for consumer
        await Task.Delay(1000);

        TestConsumer.ReceivedMessages.Should().Contain("Hello");
    }
}

public class RabbitMqContainerFixture : IAsyncLifetime
{
    private readonly RabbitMqContainer _container = new RabbitMqBuilder()
        .WithImage("rabbitmq:3-management")
        .Build();

    public string Host => _container.Hostname;
    public int Port => _container.GetMappedPublicPort(5672);
    public string Username => "guest";
    public string Password => "guest";

    public async Task InitializeAsync() => await _container.StartAsync();
    public async Task DisposeAsync() => await _container.DisposeAsync();
}
```

---

## Core Concept #16 — Performance and Flakiness

### Why This Matters

Integration tests are slower and more prone to flakiness. Without careful design:
- CI takes 30+ minutes
- Random failures erode trust
- Developers stop running tests locally

### Reducing Test Time

```csharp
// 1. Share containers across tests
[CollectionDefinition("Integration")]
public class IntegrationCollection :
    ICollectionFixture<PostgresContainerFixture>,
    ICollectionFixture<RedisContainerFixture>
{
}

// All tests in this collection share containers
[Collection("Integration")]
public class OrderTests { }

[Collection("Integration")]
public class CustomerTests { }

// 2. Parallel execution with isolated databases
public class ParallelTestFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container;
    private readonly string _schema;

    public ParallelTestFixture()
    {
        _schema = $"test_{Guid.NewGuid():N}";
    }

    public async Task InitializeAsync()
    {
        // Create isolated schema for this test class
        await using var connection = new NpgsqlConnection(_container.GetConnectionString());
        await connection.ExecuteAsync($"CREATE SCHEMA {_schema}");
    }

    public string ConnectionString =>
        $"{_container.GetConnectionString()};SearchPath={_schema}";
}

// 3. Skip slow tests locally
public class SlowTests
{
    [Fact]
    [Trait("Category", "Slow")]
    public async Task Large_data_import()
    {
        // Run this only in CI
    }
}

// Filter in CI: dotnet test --filter "Category!=Slow"
```

### Fixing Flaky Tests

```csharp
// 1. Avoid timing-based assertions
// ❌ BAD
await Task.Delay(1000);
result.Should().NotBeNull();

// ✅ GOOD: Use polling/waiting
var result = await WaitForAsync(
    () => _repository.GetByIdAsync(orderId),
    timeout: TimeSpan.FromSeconds(5));

public static async Task<T> WaitForAsync<T>(
    Func<Task<T?>> action,
    TimeSpan timeout,
    TimeSpan? interval = null) where T : class
{
    var deadline = DateTime.UtcNow + timeout;
    var pollInterval = interval ?? TimeSpan.FromMilliseconds(100);

    while (DateTime.UtcNow < deadline)
    {
        var result = await action();
        if (result != null)
            return result;

        await Task.Delay(pollInterval);
    }

    throw new TimeoutException("Condition not met within timeout");
}

// 2. Isolate external dependencies
// ❌ BAD: Real HTTP calls
var response = await _httpClient.GetAsync("https://api.external.com/data");

// ✅ GOOD: WireMock for HTTP stubbing
public class ExternalApiTests : IClassFixture<WireMockFixture>
{
    [Fact]
    public async Task Handles_external_api_error()
    {
        _wireMock.Given(Request.Create().WithPath("/data"))
            .RespondWith(Response.Create()
                .WithStatusCode(503)
                .WithBody("Service Unavailable"));

        var response = await _client.GetAsync("/api/external-data");

        response.StatusCode.Should().Be(HttpStatusCode.ServiceUnavailable);
    }
}

// 3. Use deterministic data
// ❌ BAD: Random data can cause non-deterministic results
var price = Random.Shared.Next(1, 1000);

// ✅ GOOD: Seeded random or fixed data
var price = 150m;  // Known value for assertions
```

---

## Hands-On: OrderFlow Integration Tests

### Task 1 — Set Up WebApplicationFactory

Create `OrderFlowTestFactory` with:
- In-memory database for fast tests
- Fake external services
- Test authentication

### Task 2 — Set Up Testcontainers

Create fixtures for:
- PostgreSQL
- Redis
- RabbitMQ

### Task 3 — Write API Integration Tests

Test the orders endpoint:
- Create order (success, validation failure)
- Get order (found, not found)
- List orders (with pagination)
- Update order (authorized, unauthorized)

### Task 4 — Test Data Isolation

Implement one of:
- Transaction rollback
- Respawn
- Schema per test

---

## Deliverables

1. **WebApplicationFactory** with replaceable services
2. **Testcontainers setup** for real database tests
3. **15 integration tests** covering key API flows
4. **Authentication tests** for protected endpoints

---

## Resources

### Must-Read
- [Microsoft: Integration Tests in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)
- [Testcontainers for .NET](https://dotnet.testcontainers.org/)
- [Respawn Documentation](https://github.com/jbogard/Respawn)

### Videos
- [Nick Chapsas: Integration Testing in ASP.NET Core](https://www.youtube.com/watch?v=7roqteWLw4s)
- [Raw Coding: Testing with Testcontainers](https://www.youtube.com/watch?v=8IRNC7qZBmk)
- [Milan Jovanovic: Integration Testing](https://www.youtube.com/watch?v=tj5ZCtvgXKY)

### Tools
- [WireMock.NET](https://github.com/WireMock-Net/WireMock.Net) — HTTP mocking
- [Respawn](https://github.com/jbogard/Respawn) — Database cleanup
- [Bogus](https://github.com/bchavez/Bogus) — Test data generation

---

## Reflection Questions

1. When should you use in-memory database vs Testcontainers?
2. How do you balance test isolation with test speed?
3. What makes an integration test flaky?
4. How do you test code that calls external APIs?

---

**Next:** [Part 4 — Contract & E2E Testing](./part-4.md)
