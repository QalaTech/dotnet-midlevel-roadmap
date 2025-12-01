# 07 — Testing Mastery: Unit, Integration, Contract

## Part 4 / 4 — Contract & E2E Testing

---

## Why This Matters

In distributed systems, services depend on each other. When Service A changes its API:
- Service B might break
- But B's tests still pass (they mock A)
- Bug discovered in production

Contract tests catch breaking changes before deployment. E2E tests verify the complete system works.

---

## Core Concept #17 — Consumer-Driven Contracts

### Why This Matters

Traditional testing tests what you build. Contract testing tests what others expect from you.

```
Without Contracts:
Service A (Provider) → Changes response format
Service B (Consumer) → Tests mock old format → Pass ✓
Production → B calls A → Crash!

With Contracts:
Service B → Defines expected contract
Service A → Runs against B's contract → Fail ✗
Deploy blocked → Fix before production
```

### What Goes Wrong Without This

**War story — The Breaking Change:**
Team renamed `firstName` to `first_name` in API response. Their tests passed. Frontend team's tests passed (mocked the old format). Deployed on Friday. Entire checkout broken. Rollback took 2 hours.

**War story — The Silent Addition:**
API added required field. Provider tests passed. Consumer was sending old format. 500 errors on every request. No one noticed until monitoring alerted.

### Pact — Consumer-Driven Contracts

```csharp
// 1. Consumer defines expected interactions
public class OrderServiceConsumerTests
{
    private readonly IPactBuilderV4 _pactBuilder;

    public OrderServiceConsumerTests()
    {
        var config = new PactConfig
        {
            PactDir = Path.Join("..", "..", "..", "pacts"),
            LogLevel = PactLogLevel.Information
        };

        _pactBuilder = Pact.V4("OrdersWebApp", "OrdersApi", config).WithHttpInteractions();
    }

    [Fact]
    public async Task Get_order_returns_order_details()
    {
        var orderId = Guid.Parse("11111111-1111-1111-1111-111111111111");

        // Define expected interaction
        _pactBuilder
            .UponReceiving("A request to get an order")
            .Given("Order exists", new Dictionary<string, string>
            {
                { "orderId", orderId.ToString() }
            })
            .WithRequest(HttpMethod.Get, $"/api/orders/{orderId}")
            .WithHeader("Accept", "application/json")
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithHeader("Content-Type", "application/json")
            .WithJsonBody(new
            {
                id = Match.Type(orderId),
                customerId = Match.Type(Guid.NewGuid()),
                total = Match.Decimal(150.00m),
                status = Match.Regex("Pending|Paid|Shipped", "Pending"),
                items = Match.MinType(new
                {
                    productId = Match.Type(Guid.NewGuid()),
                    quantity = Match.Integer(2),
                    price = Match.Decimal(75.00m)
                }, 1)
            });

        // Execute consumer code against mock
        await _pactBuilder.VerifyAsync(async ctx =>
        {
            var client = new HttpClient { BaseAddress = ctx.MockServerUri };
            var orderService = new OrderServiceClient(client);

            var order = await orderService.GetOrderAsync(orderId);

            order.Should().NotBeNull();
            order.Id.Should().Be(orderId);
            order.Items.Should().NotBeEmpty();
        });

        // Pact file generated at pacts/OrdersWebApp-OrdersApi.json
    }

    [Fact]
    public async Task Create_order_returns_created()
    {
        _pactBuilder
            .UponReceiving("A request to create an order")
            .WithRequest(HttpMethod.Post, "/api/orders")
            .WithHeader("Content-Type", "application/json")
            .WithJsonBody(new
            {
                customerId = Match.Type(Guid.NewGuid()),
                items = Match.MinType(new
                {
                    productId = Match.Type(Guid.NewGuid()),
                    quantity = Match.Integer(1),
                    price = Match.Decimal(100.00m)
                }, 1)
            })
            .WillRespond()
            .WithStatus(HttpStatusCode.Created)
            .WithHeader("Location", Match.Regex(@"/api/orders/[a-f0-9\-]+", "/api/orders/123"));

        await _pactBuilder.VerifyAsync(async ctx =>
        {
            var client = new HttpClient { BaseAddress = ctx.MockServerUri };
            var orderService = new OrderServiceClient(client);

            var result = await orderService.CreateOrderAsync(new CreateOrderRequest
            {
                CustomerId = Guid.NewGuid(),
                Items = new[] { new OrderItem(Guid.NewGuid(), 1, 100m) }
            });

            result.Should().NotBeNull();
        });
    }
}
```

### Provider Verification

```csharp
// 2. Provider verifies against consumer contracts
public class OrdersApiProviderTests : IClassFixture<OrderFlowWebAppFactory>
{
    private readonly OrderFlowWebAppFactory _factory;

    public OrdersApiProviderTests(OrderFlowWebAppFactory factory)
    {
        _factory = factory;
    }

    [Fact]
    public void Verify_pact_with_consumers()
    {
        var config = new PactVerifierConfig
        {
            LogLevel = PactLogLevel.Information
        };

        using var server = _factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Set up provider states
                services.AddSingleton<IProviderStateMiddleware, ProviderStateMiddleware>();
            });
        }).CreateDefaultClient();

        var verifier = new PactVerifier("OrdersApi", config);

        verifier
            .WithHttpEndpoint(_factory.Server.BaseAddress)
            .WithPactBrokerSource(new Uri("https://pact-broker.example.com"), options =>
            {
                options.ConsumerVersionSelectors(new ConsumerVersionSelector
                {
                    MainBranch = true
                });
                options.PublishResults(Environment.GetEnvironmentVariable("CI") == "true");
            })
            .WithProviderStateUrl(new Uri(_factory.Server.BaseAddress, "/provider-states"))
            .Verify();
    }
}

// Provider state setup
public class ProviderStateMiddleware
{
    private readonly RequestDelegate _next;
    private readonly OrderFlowDbContext _context;

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.StartsWithSegments("/provider-states"))
        {
            await HandleProviderState(context);
            return;
        }

        await _next(context);
    }

    private async Task HandleProviderState(HttpContext context)
    {
        var state = await context.Request.ReadFromJsonAsync<ProviderState>();

        switch (state?.State)
        {
            case "Order exists":
                var orderId = Guid.Parse(state.Params["orderId"]);
                await SeedOrderAsync(orderId);
                break;

            case "No orders exist":
                await _context.Orders.ExecuteDeleteAsync();
                break;
        }

        context.Response.StatusCode = 200;
    }

    private async Task SeedOrderAsync(Guid orderId)
    {
        var order = new Order(Guid.NewGuid()) { Id = orderId };
        order.AddItem(Guid.NewGuid(), 2, 75m);
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
    }
}
```

### Pact Broker Workflow

```yaml
# .github/workflows/pact.yml
name: Pact Contract Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  consumer-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run consumer contract tests
        run: dotnet test OrderFlow.Consumer.Tests

      - name: Publish pacts to broker
        run: |
          pact-broker publish ./pacts \
            --consumer-app-version ${{ github.sha }} \
            --branch ${{ github.ref_name }} \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --broker-token ${{ secrets.PACT_BROKER_TOKEN }}

  provider-verification:
    runs-on: ubuntu-latest
    needs: consumer-tests
    steps:
      - uses: actions/checkout@v4

      - name: Verify provider against pacts
        run: dotnet test OrderFlow.Provider.Tests
        env:
          CI: true
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  can-i-deploy:
    runs-on: ubuntu-latest
    needs: provider-verification
    steps:
      - name: Check if safe to deploy
        run: |
          pact-broker can-i-deploy \
            --pacticipant OrdersApi \
            --version ${{ github.sha }} \
            --to-environment production \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --broker-token ${{ secrets.PACT_BROKER_TOKEN }}
```

---

## Core Concept #18 — OpenAPI Contract Testing

### Why This Matters

OpenAPI schemas are contracts too. They define:
- Required fields
- Data types
- Allowed values

Schema validation catches drift between docs and implementation.

### Schema Validation

```csharp
// Generate and validate OpenAPI schema
public class OpenApiContractTests : IClassFixture<OrderFlowWebAppFactory>
{
    private readonly HttpClient _client;

    public OpenApiContractTests(OrderFlowWebAppFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task OpenAPI_schema_is_valid()
    {
        var response = await _client.GetAsync("/swagger/v1/swagger.json");
        response.EnsureSuccessStatusCode();

        var schema = await response.Content.ReadAsStringAsync();

        // Parse and validate schema
        var reader = new OpenApiStringReader();
        var result = reader.Read(schema, out var diagnostic);

        diagnostic.Errors.Should().BeEmpty();
        result.Should().NotBeNull();
    }

    [Fact]
    public async Task Create_order_matches_schema()
    {
        var response = await _client.GetAsync("/swagger/v1/swagger.json");
        var schemaJson = await response.Content.ReadAsStringAsync();
        var schema = JObject.Parse(schemaJson);

        // Get the CreateOrder request schema
        var createOrderSchema = schema
            .SelectToken("$.paths./api/orders.post.requestBody.content.application/json.schema");

        // Validate actual request body
        var request = new
        {
            customerId = Guid.NewGuid(),
            items = new[] { new { productId = Guid.NewGuid(), quantity = 1, price = 100m } }
        };

        var requestJson = JObject.FromObject(request);

        // Use JSON Schema validation
        var validationResult = requestJson.IsValid(JSchema.Parse(createOrderSchema!.ToString()));
        validationResult.Should().BeTrue();
    }
}

// Spectral linting in CI
// .spectral.yaml
extends: ["spectral:oas"]
rules:
  operation-operationId: error
  operation-description: warn
  operation-tags: error
  paths-kebab-case: error
  no-$ref-siblings: error
```

### Schema Diff Detection

```csharp
// Detect breaking changes between schema versions
public class SchemaBreakingChangeTests
{
    [Fact]
    public void Detect_breaking_changes()
    {
        var oldSchema = LoadSchema("v1.0.0/swagger.json");
        var newSchema = LoadSchema("current/swagger.json");

        var differ = new OpenApiDiff();
        var changes = differ.Compare(oldSchema, newSchema);

        var breakingChanges = changes.Where(c => c.IsBreaking).ToList();

        breakingChanges.Should().BeEmpty(
            $"Breaking changes detected: {string.Join(", ", breakingChanges.Select(c => c.Description))}");
    }

    // Breaking changes:
    // - Removed endpoint
    // - Removed required field from response
    // - Added required field to request
    // - Changed field type
    // - Removed enum value

    // Non-breaking:
    // - Added endpoint
    // - Added optional field to request
    // - Added field to response
    // - Added enum value
}
```

---

## Core Concept #19 — E2E Testing Strategy

### Why This Matters

E2E tests verify the complete system. They catch:
- Integration issues between services
- Configuration problems
- Infrastructure issues

But E2E tests are expensive:
- Slow (real services, real databases)
- Flaky (timing, external dependencies)
- Maintenance burden

### When to Use E2E Tests

```
Use E2E for:
✓ Critical user journeys (checkout, registration)
✓ Happy path verification
✓ Smoke tests after deployment

Don't use E2E for:
✗ Edge cases (use unit/integration)
✗ Error handling (use unit tests)
✗ Every scenario (too expensive)
```

### E2E Test Structure

```csharp
// E2E tests using Playwright for API testing
public class OrderFlowE2ETests : IAsyncLifetime
{
    private readonly IPlaywright _playwright;
    private IAPIRequestContext _apiContext = null!;

    public OrderFlowE2ETests()
    {
        _playwright = Playwright.CreateAsync().Result;
    }

    public async Task InitializeAsync()
    {
        _apiContext = await _playwright.APIRequest.NewContextAsync(new()
        {
            BaseURL = "https://staging.orderflow.example.com",
            ExtraHTTPHeaders = new Dictionary<string, string>
            {
                ["Accept"] = "application/json",
                ["Authorization"] = $"Bearer {GetTestToken()}"
            }
        });
    }

    public async Task DisposeAsync()
    {
        await _apiContext.DisposeAsync();
        _playwright.Dispose();
    }

    [Fact]
    public async Task Complete_order_flow()
    {
        // 1. Create order
        var createResponse = await _apiContext.PostAsync("/api/orders", new()
        {
            DataObject = new
            {
                customerId = Guid.NewGuid(),
                items = new[]
                {
                    new { productId = Guid.Parse("product-1"), quantity = 2, price = 100m }
                }
            }
        });
        createResponse.Ok.Should().BeTrue();

        var orderLocation = createResponse.Headers["Location"];
        var orderId = orderLocation.Split('/').Last();

        // 2. Get order - should be Pending
        var getResponse = await _apiContext.GetAsync($"/api/orders/{orderId}");
        var order = await getResponse.JsonAsync();
        order!.GetProperty("status").GetString().Should().Be("Pending");

        // 3. Pay order
        var payResponse = await _apiContext.PostAsync($"/api/orders/{orderId}/pay", new()
        {
            DataObject = new
            {
                paymentMethod = "CreditCard",
                cardToken = "tok_test_visa"
            }
        });
        payResponse.Ok.Should().BeTrue();

        // 4. Verify order is Paid
        var paidOrder = await (await _apiContext.GetAsync($"/api/orders/{orderId}")).JsonAsync();
        paidOrder!.GetProperty("status").GetString().Should().Be("Paid");

        // 5. Ship order
        var shipResponse = await _apiContext.PostAsync($"/api/orders/{orderId}/ship");
        shipResponse.Ok.Should().BeTrue();

        // 6. Verify order is Shipped
        var shippedOrder = await (await _apiContext.GetAsync($"/api/orders/{orderId}")).JsonAsync();
        shippedOrder!.GetProperty("status").GetString().Should().Be("Shipped");
    }

    [Fact]
    public async Task Smoke_test_critical_endpoints()
    {
        // Just verify endpoints respond
        var endpoints = new[]
        {
            "/api/health",
            "/api/orders",
            "/api/products",
            "/api/customers/me"
        };

        foreach (var endpoint in endpoints)
        {
            var response = await _apiContext.GetAsync(endpoint);
            response.Status.Should().BeLessThan(500,
                $"Endpoint {endpoint} returned server error");
        }
    }
}
```

### Smoke Tests Post-Deployment

```csharp
// Run after every deployment
public class ProductionSmokeTests
{
    private readonly HttpClient _client;

    public ProductionSmokeTests()
    {
        _client = new HttpClient
        {
            BaseAddress = new Uri(Environment.GetEnvironmentVariable("API_URL")
                ?? "https://api.orderflow.example.com")
        };
    }

    [Fact]
    public async Task Health_check_passes()
    {
        var response = await _client.GetAsync("/health");
        response.EnsureSuccessStatusCode();

        var health = await response.Content.ReadFromJsonAsync<HealthCheckResult>();
        health!.Status.Should().Be("Healthy");
    }

    [Fact]
    public async Task Database_connection_healthy()
    {
        var response = await _client.GetAsync("/health/db");
        response.EnsureSuccessStatusCode();
    }

    [Fact]
    public async Task Can_list_products()
    {
        var response = await _client.GetAsync("/api/products?limit=1");
        response.EnsureSuccessStatusCode();

        var products = await response.Content.ReadFromJsonAsync<List<Product>>();
        products.Should().NotBeNull();
    }

    [Fact]
    public async Task API_version_matches_expected()
    {
        var response = await _client.GetAsync("/api/version");
        var version = await response.Content.ReadAsStringAsync();

        var expectedVersion = Environment.GetEnvironmentVariable("EXPECTED_VERSION");
        if (!string.IsNullOrEmpty(expectedVersion))
        {
            version.Should().Contain(expectedVersion);
        }
    }
}
```

---

## Core Concept #20 — CI/CD Integration

### Why This Matters

Tests are only useful if they run automatically. A good CI pipeline:
- Runs tests on every PR
- Blocks merges on failures
- Provides fast feedback
- Produces artifacts

### Comprehensive Test Pipeline

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore

      - name: Run unit tests
        run: |
          dotnet test tests/OrderFlow.Domain.Tests \
            --no-build \
            --logger "trx;LogFileName=unit-results.trx" \
            --collect:"XPlat Code Coverage"

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: unit-test-results
          path: '**/TestResults/**/*.trx'

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: '**/TestResults/**/coverage.cobertura.xml'
          fail_ci_if_error: true

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: orderflow_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Run integration tests
        run: |
          dotnet test tests/OrderFlow.Integration.Tests \
            --logger "trx;LogFileName=integration-results.trx"
        env:
          ConnectionStrings__Database: "Host=localhost;Database=orderflow_test;Username=postgres;Password=test"
          ConnectionStrings__Redis: "localhost:6379"

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: integration-test-results
          path: '**/TestResults/**/*.trx'

  contract-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests]
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Run consumer tests
        run: dotnet test tests/OrderFlow.Consumer.Tests

      - name: Publish pacts
        if: github.event_name == 'push'
        run: |
          pact-broker publish ./pacts \
            --consumer-app-version ${{ github.sha }} \
            --branch ${{ github.ref_name }} \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --broker-token ${{ secrets.PACT_BROKER_TOKEN }}

      - name: Verify provider
        run: dotnet test tests/OrderFlow.Provider.Tests
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Install Playwright
        run: pwsh tests/OrderFlow.E2E.Tests/bin/Debug/net8.0/playwright.ps1 install

      - name: Run E2E tests against staging
        run: dotnet test tests/OrderFlow.E2E.Tests
        env:
          API_URL: ${{ secrets.STAGING_API_URL }}
          TEST_TOKEN: ${{ secrets.STAGING_TEST_TOKEN }}

  deploy-gate:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, contract-tests]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Check if safe to deploy
        run: |
          pact-broker can-i-deploy \
            --pacticipant OrdersApi \
            --version ${{ github.sha }} \
            --to-environment production \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --broker-token ${{ secrets.PACT_BROKER_TOKEN }}
```

### Test Result Reporting

```csharp
// Custom test reporter for better CI output
public class CITestReporter : ITestOutputHelper
{
    public void WriteLine(string message)
    {
        // Format for GitHub Actions annotations
        if (message.Contains("FAIL"))
        {
            Console.WriteLine($"::error::{message}");
        }
        else if (message.Contains("WARN"))
        {
            Console.WriteLine($"::warning::{message}");
        }
        else
        {
            Console.WriteLine(message);
        }
    }

    public void WriteLine(string format, params object[] args)
    {
        WriteLine(string.Format(format, args));
    }
}

// Generate test summary for PR comments
public static class TestSummaryGenerator
{
    public static string GenerateMarkdown(TestResults results)
    {
        var sb = new StringBuilder();

        sb.AppendLine("## Test Results");
        sb.AppendLine();
        sb.AppendLine($"| Category | Passed | Failed | Skipped |");
        sb.AppendLine($"|----------|--------|--------|---------|");
        sb.AppendLine($"| Unit | {results.UnitPassed} | {results.UnitFailed} | {results.UnitSkipped} |");
        sb.AppendLine($"| Integration | {results.IntegrationPassed} | {results.IntegrationFailed} | {results.IntegrationSkipped} |");
        sb.AppendLine($"| Contract | {results.ContractPassed} | {results.ContractFailed} | {results.ContractSkipped} |");

        if (results.FailedTests.Any())
        {
            sb.AppendLine();
            sb.AppendLine("### Failed Tests");
            foreach (var test in results.FailedTests)
            {
                sb.AppendLine($"- ❌ `{test.Name}`: {test.Error}");
            }
        }

        sb.AppendLine();
        sb.AppendLine($"**Coverage**: {results.CoveragePercent:F1}%");

        return sb.ToString();
    }
}
```

---

## Core Concept #21 — Test Maintenance

### Why This Matters

Test suites rot over time:
- Tests become slow
- Tests become flaky
- Tests become irrelevant
- Coverage gaps appear

Regular maintenance keeps tests valuable.

### Test Health Metrics

```csharp
// Track test health over time
public class TestHealthDashboard
{
    public record TestMetrics(
        int TotalTests,
        int PassingTests,
        int FlakyTests,
        TimeSpan AverageRunTime,
        double CoveragePercent,
        int TestsAddedThisWeek,
        int TestsRemovedThisWeek
    );

    // Flaky test detection
    public static async Task<List<string>> DetectFlakyTestsAsync(
        ITestRunRepository repository,
        int runsToAnalyze = 100)
    {
        var recentRuns = await repository.GetRecentRunsAsync(runsToAnalyze);

        var testResults = recentRuns
            .SelectMany(r => r.Tests)
            .GroupBy(t => t.Name)
            .Select(g => new
            {
                Name = g.Key,
                Runs = g.Count(),
                Passes = g.Count(t => t.Passed),
                Failures = g.Count(t => !t.Passed)
            })
            .Where(t => t.Runs >= 10)  // Enough data
            .Where(t => t.Failures > 0 && t.Passes > 0)  // Not consistent
            .OrderByDescending(t => (double)t.Failures / t.Runs)
            .ToList();

        return testResults
            .Where(t => (double)t.Failures / t.Runs > 0.05)  // >5% failure rate
            .Select(t => t.Name)
            .ToList();
    }
}
```

### Dealing with Flaky Tests

```csharp
// 1. Quarantine flaky tests
[Fact]
[Trait("Flaky", "true")]
[Trait("Issue", "JIRA-1234")]
public async Task Flaky_test_under_investigation()
{
    // Skip in regular runs, run in dedicated flaky test job
}

// 2. Add retry logic for known timing issues
public class RetryAttribute : BeforeAfterTestAttribute
{
    private readonly int _maxRetries;

    public RetryAttribute(int maxRetries = 3)
    {
        _maxRetries = maxRetries;
    }

    public override void Before(MethodInfo methodUnderTest)
    {
        // Set up retry context
    }
}

// 3. Add better waiting mechanisms
public static class TestWaiter
{
    public static async Task WaitForConditionAsync(
        Func<Task<bool>> condition,
        TimeSpan timeout,
        string description)
    {
        var deadline = DateTime.UtcNow + timeout;

        while (DateTime.UtcNow < deadline)
        {
            if (await condition())
                return;

            await Task.Delay(100);
        }

        throw new TimeoutException($"Condition not met: {description}");
    }
}

// Usage
await TestWaiter.WaitForConditionAsync(
    async () => (await _repo.GetByIdAsync(id)) != null,
    TimeSpan.FromSeconds(5),
    "Order should be persisted");
```

---

## Hands-On: OrderFlow Contract Tests

### Task 1 — Consumer Contract Tests

Write Pact consumer tests for:
- Get order by ID
- Create order
- List orders with pagination

### Task 2 — Provider Verification

Set up provider verification with:
- Provider states
- Pact broker integration
- CI pipeline

### Task 3 — OpenAPI Validation

Implement:
- Schema generation test
- Request/response validation
- Breaking change detection

### Task 4 — E2E Smoke Tests

Create smoke tests for:
- Health endpoints
- Authentication flow
- Critical user journey

---

## Deliverables

1. **Pact consumer tests** for 3+ interactions
2. **Provider verification** with states
3. **OpenAPI contract tests** in CI
4. **E2E smoke test suite** for staging
5. **GitHub Actions workflow** for all test types

---

## Resources

### Must-Read
- [Pact Documentation](https://docs.pact.io/)
- [Consumer-Driven Contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)
- [Testing in Production](https://copyconstruct.medium.com/testing-in-production-the-safe-way-18ca102d0ef1)

### Videos
- [Pact: Contract Testing Made Easy](https://www.youtube.com/watch?v=h-79QmIV824)
- [Nick Chapsas: Contract Testing with Pact](https://www.youtube.com/watch?v=qp8R1X9fhmo)
- [Martin Fowler: Testing Strategies](https://www.youtube.com/watch?v=HhwElTL-mdI)

### Tools
- [Pact.NET](https://github.com/pact-foundation/pact-net)
- [Spectral](https://stoplight.io/open-source/spectral) — OpenAPI linting
- [Playwright](https://playwright.dev/) — E2E testing
- [Pact Broker](https://docs.pact.io/pact_broker) — Contract storage

---

## Reflection Questions

1. When should you use contract testing vs integration testing?
2. How do you handle breaking API changes?
3. What's the right number of E2E tests?
4. How do you maintain test suites long-term?

---

## Module Summary

You've learned:
1. **Testing Foundations** — Pyramid, what to test, test doubles
2. **Unit Testing** — Domain entities, handlers, async code
3. **Integration Testing** — WebApplicationFactory, Testcontainers, authentication
4. **Contract & E2E** — Pact, OpenAPI, smoke tests, CI integration

A well-tested codebase is:
- Confident to deploy
- Safe to refactor
- Easy to understand
- Documented by tests

---

**Module Complete!** Continue to [Module 08 — Security](../08-security/README.md)
