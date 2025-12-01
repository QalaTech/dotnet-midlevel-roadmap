# Part 4 — Resilience & Reliability Patterns

## Why This Matters

In distributed systems, failure isn't exceptional — it's normal.

> "Everything fails, all the time." — Werner Vogels, CTO of Amazon

Networks fail. Services crash. Databases timeout. The question isn't whether failures will happen, but whether your system handles them gracefully.

---

## What Goes Wrong Without This

### The Cascade Failure

```
3:00 AM: Payment service slows down (database issue)
3:01 AM: Order service waiting for payment (threads blocked)
3:02 AM: Order service thread pool exhausted
3:03 AM: API gateway queuing requests
3:04 AM: All requests timing out
3:05 AM: Every dependent service is now failing

One slow service → entire platform down

Without resilience patterns:
- No circuit breaker = keep hammering failing service
- No timeout = threads wait forever
- No bulkhead = one failure affects everything
```

### The Retry Storm

```
Initial failure: Payment service returns 500

Without backoff:
- Client 1 retries immediately
- Client 2 retries immediately
- ... 1000 clients retry immediately
- Payment service: Now handling 10x load while recovering
- Recovery time: Infinite (system never stabilizes)

With exponential backoff:
- Client 1 waits 1s, then 2s, then 4s
- Clients spread retries over time
- Payment service load stays manageable
- Recovery time: Minutes
```

### The Timeout That Wasn't

```csharp
// Anti-pattern: No timeout
var response = await _httpClient.GetAsync("/api/orders");

// What happens when service is slow?
// - Thread blocked for... how long?
// - Default HttpClient timeout: 100 seconds!
// - 100 blocked threads = thread pool exhaustion

// Production incident:
// "We had 47 threads all waiting for the same dead service"
```

---

## Resilience Patterns Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESILIENCE PATTERNS                          │
├─────────────────┬─────────────────┬─────────────────────────────┤
│     RETRY       │ CIRCUIT BREAKER │        TIMEOUT              │
│                 │                 │                             │
│ "Try again,     │ "Stop calling   │ "Don't wait forever"        │
│  maybe it'll    │  if it keeps    │                             │
│  work"          │  failing"       │                             │
├─────────────────┼─────────────────┼─────────────────────────────┤
│    BULKHEAD     │    FALLBACK     │        HEDGING              │
│                 │                 │                             │
│ "Isolate        │ "Return         │ "Try multiple"              │
│  failures"      │  something"     │                             │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

## .NET Resilience with Polly v8

### Basic Setup

```bash
dotnet add package Microsoft.Extensions.Http.Resilience
dotnet add package Polly.Extensions
```

### Configuring Resilience Pipelines

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add resilient HTTP client for Payment Service
builder.Services.AddHttpClient<IPaymentClient, PaymentClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["PaymentService:Url"]!);
})
.AddStandardResilienceHandler(options =>
{
    // Retry configuration
    options.Retry.MaxRetryAttempts = 3;
    options.Retry.Delay = TimeSpan.FromMilliseconds(500);
    options.Retry.BackoffType = DelayBackoffType.Exponential;
    options.Retry.UseJitter = true;

    // Circuit breaker
    options.CircuitBreaker.FailureRatio = 0.5;
    options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(10);
    options.CircuitBreaker.MinimumThroughput = 10;
    options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);

    // Timeout
    options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
    options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(10);
});
```

### Custom Resilience Pipeline

```csharp
// For more control, build custom pipelines
builder.Services.AddResiliencePipeline("inventory-pipeline", builder =>
{
    builder
        // Retry with exponential backoff
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(200),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = new PredicateBuilder()
                .Handle<HttpRequestException>()
                .Handle<TimeoutRejectedException>()
                .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode),
            OnRetry = args =>
            {
                Log.Warning("Retry attempt {Attempt} after {Delay}ms. Error: {Error}",
                    args.AttemptNumber,
                    args.RetryDelay.TotalMilliseconds,
                    args.Outcome.Exception?.Message);
                return default;
            }
        })
        // Circuit breaker
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(10),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(30),
            OnOpened = args =>
            {
                Log.Error("Circuit OPENED. Failures: {Failures}", args.Outcome.Exception?.Message);
                return default;
            },
            OnClosed = args =>
            {
                Log.Information("Circuit CLOSED. Service recovered.");
                return default;
            },
            OnHalfOpened = args =>
            {
                Log.Information("Circuit HALF-OPEN. Testing service...");
                return default;
            }
        })
        // Timeout
        .AddTimeout(new TimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(5),
            OnTimeout = args =>
            {
                Log.Warning("Request timed out after {Timeout}s", args.Timeout.TotalSeconds);
                return default;
            }
        });
});

// Use in a service
public class InventoryClient
{
    private readonly HttpClient _client;
    private readonly ResiliencePipeline _pipeline;

    public InventoryClient(
        HttpClient client,
        [FromKeyedServices("inventory-pipeline")] ResiliencePipeline pipeline)
    {
        _client = client;
        _pipeline = pipeline;
    }

    public async Task<StockLevel?> GetStockAsync(int productId)
    {
        return await _pipeline.ExecuteAsync(async ct =>
        {
            var response = await _client.GetAsync($"/api/stock/{productId}", ct);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<StockLevel>(ct);
        });
    }
}
```

---

## Pattern Deep Dives

### Retry with Exponential Backoff

**Why exponential backoff?** Constant retries hammer a recovering service. Exponential backoff gives it breathing room.

```csharp
// Retry delays: 200ms → 400ms → 800ms → 1600ms (with jitter)

.AddRetry(new RetryStrategyOptions
{
    MaxRetryAttempts = 4,
    Delay = TimeSpan.FromMilliseconds(200),
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true,  // Add randomness to prevent thundering herd
    ShouldHandle = new PredicateBuilder()
        .Handle<HttpRequestException>()
        .HandleResult<HttpResponseMessage>(r =>
            r.StatusCode == HttpStatusCode.ServiceUnavailable ||
            r.StatusCode == HttpStatusCode.TooManyRequests ||
            r.StatusCode == HttpStatusCode.GatewayTimeout)
})
```

**What to retry:**
- Network errors (`HttpRequestException`)
- 503 Service Unavailable
- 429 Too Many Requests
- 504 Gateway Timeout

**What NOT to retry:**
- 400 Bad Request (your data is wrong)
- 401/403 Unauthorized (auth issues)
- 404 Not Found (resource doesn't exist)
- 409 Conflict (business logic violation)

### Circuit Breaker

```
        CLOSED                    OPEN                  HALF-OPEN
          │                         │                       │
          │ Success                 │                       │
          ▼                         │                       │
       ┌─────┐    Failures      ┌─────┐    After       ┌─────┐
       │     │  > threshold     │     │  break time    │     │
       │ OK  │ ──────────────►  │OPEN │ ─────────────► │TEST │
       │     │                  │     │                │     │
       └─────┘                  └─────┘                └─────┘
          ▲                         │                    │  │
          │                         │ All requests      │  │
          │                         │ fail fast         │  │
          │                         │                   │  │
          │         Success         │                   │  │
          └─────────────────────────┴───────────────────┘  │
                                           │ Failure       │
                                           └───────────────┘
                                             Back to OPEN
```

```csharp
.AddCircuitBreaker(new CircuitBreakerStrategyOptions
{
    // Break when 50% of requests fail
    FailureRatio = 0.5,

    // ...in the last 10 seconds
    SamplingDuration = TimeSpan.FromSeconds(10),

    // ...with at least 10 requests
    MinimumThroughput = 10,

    // Stay open for 30 seconds
    BreakDuration = TimeSpan.FromSeconds(30)
})
```

### Bulkhead Isolation

Prevent one failing service from taking down the whole system.

```csharp
// Limit concurrent calls to external services
.AddConcurrencyLimiter(new ConcurrencyLimiterOptions
{
    PermitLimit = 10,  // Max 10 concurrent requests
    QueueLimit = 20    // Queue up to 20 more
})

// Or use separate HttpClient instances per service
builder.Services.AddHttpClient<IPaymentClient, PaymentClient>()
    .AddStandardResilienceHandler();

builder.Services.AddHttpClient<IInventoryClient, InventoryClient>()
    .AddStandardResilienceHandler();

// Each client has its own connection pool and resilience pipeline
// Payment slowdown won't affect Inventory calls
```

### Timeout Strategies

```csharp
// Two levels of timeout:
// 1. Per-attempt timeout (for individual retries)
// 2. Total timeout (for the entire operation)

.AddTimeout(new TimeoutStrategyOptions
{
    Timeout = TimeSpan.FromSeconds(3)  // Per attempt
})
.AddRetry(new RetryStrategyOptions
{
    MaxRetryAttempts = 3,
    Delay = TimeSpan.FromSeconds(1)
})
.AddTimeout(new TimeoutStrategyOptions
{
    Timeout = TimeSpan.FromSeconds(15)  // Total operation
})

// Attempt 1: 3s timeout
// Wait: 1s
// Attempt 2: 3s timeout
// Wait: 2s
// Attempt 3: 3s timeout
// Total: 3 + 1 + 3 + 2 + 3 = 12s (within 15s total)
```

### Fallback Strategies

When all else fails, return something useful.

```csharp
public class ProductCatalogClient
{
    private readonly HttpClient _client;
    private readonly ResiliencePipeline<ProductInfo?> _pipeline;
    private readonly IMemoryCache _cache;

    public async Task<ProductInfo?> GetProductAsync(int productId)
    {
        var context = ResilienceContextPool.Shared.Get();
        context.Properties.Set(new ResiliencePropertyKey<int>("ProductId"), productId);

        try
        {
            return await _pipeline.ExecuteAsync(async ctx =>
            {
                // Try primary source
                var response = await _client.GetAsync($"/products/{productId}");

                if (response.IsSuccessStatusCode)
                {
                    var product = await response.Content.ReadFromJsonAsync<ProductInfo>();

                    // Cache successful responses
                    _cache.Set($"product:{productId}", product, TimeSpan.FromMinutes(5));
                    return product;
                }

                // Primary failed, try cache
                if (_cache.TryGetValue($"product:{productId}", out ProductInfo? cached))
                {
                    Log.Warning("Using cached product data for {ProductId}", productId);
                    return cached;
                }

                // No cache, return minimal fallback
                Log.Error("No product data available for {ProductId}", productId);
                return new ProductInfo
                {
                    Id = productId,
                    Name = "Product information temporarily unavailable",
                    Price = 0,
                    Available = false
                };
            }, context);
        }
        finally
        {
            ResilienceContextPool.Shared.Return(context);
        }
    }
}
```

---

## Monitoring Resilience

### Exposing Resilience Metrics

```csharp
// Add telemetry to resilience pipelines
builder.Services.AddResiliencePipeline("inventory-pipeline", (builder, context) =>
{
    builder
        .AddRetry(new RetryStrategyOptions { /* ... */ })
        .ConfigureTelemetry(context.ServiceProvider.GetRequiredService<ILoggerFactory>());
});

// Custom metrics with OpenTelemetry
public class ResilienceMetrics
{
    private readonly Counter<long> _retryAttempts;
    private readonly Counter<long> _circuitBreakerStateChanges;
    private readonly Counter<long> _fallbackExecutions;
    private readonly Histogram<double> _requestDuration;

    public ResilienceMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("OrderFlow.Resilience");

        _retryAttempts = meter.CreateCounter<long>(
            "resilience.retry.attempts",
            description: "Number of retry attempts");

        _circuitBreakerStateChanges = meter.CreateCounter<long>(
            "resilience.circuitbreaker.state_changes",
            description: "Circuit breaker state transitions");

        _fallbackExecutions = meter.CreateCounter<long>(
            "resilience.fallback.executions",
            description: "Number of fallback executions");

        _requestDuration = meter.CreateHistogram<double>(
            "resilience.request.duration",
            unit: "ms",
            description: "Request duration including retries");
    }

    public void RecordRetryAttempt(string service, int attempt)
    {
        _retryAttempts.Add(1,
            new KeyValuePair<string, object?>("service", service),
            new KeyValuePair<string, object?>("attempt", attempt));
    }

    public void RecordCircuitBreakerStateChange(string service, string fromState, string toState)
    {
        _circuitBreakerStateChanges.Add(1,
            new KeyValuePair<string, object?>("service", service),
            new KeyValuePair<string, object?>("from_state", fromState),
            new KeyValuePair<string, object?>("to_state", toState));
    }
}
```

### Grafana Dashboard Queries

```promql
# Retry rate by service
sum(rate(resilience_retry_attempts_total[5m])) by (service)

# Circuit breaker state changes
sum(increase(resilience_circuitbreaker_state_changes_total{to_state="open"}[1h])) by (service)

# Fallback usage
sum(rate(resilience_fallback_executions_total[5m])) by (service)

# Request success rate including retries
sum(rate(http_client_requests_total{status="success"}[5m])) by (service)
/
sum(rate(http_client_requests_total[5m])) by (service)
```

---

## Chaos Engineering

### What Is Chaos Engineering?

> "Chaos Engineering is the discipline of experimenting on a system to build confidence in the system's capability to withstand turbulent conditions in production."

### Toxiproxy Setup

```yaml
# docker-compose.toxiproxy.yml
version: '3.8'
services:
  toxiproxy:
    image: ghcr.io/shopify/toxiproxy:latest
    ports:
      - "8474:8474"  # API port
      - "18080:18080"  # Proxy to payment service
      - "18081:18081"  # Proxy to inventory service

  # Configure proxies on startup
  toxiproxy-config:
    image: curlimages/curl
    depends_on:
      - toxiproxy
    entrypoint: /bin/sh
    command: |
      -c "
        sleep 2
        curl -X POST http://toxiproxy:8474/proxies -d '{
          \"name\": \"payment\",
          \"listen\": \"0.0.0.0:18080\",
          \"upstream\": \"payment-service:8080\"
        }'
        curl -X POST http://toxiproxy:8474/proxies -d '{
          \"name\": \"inventory\",
          \"listen\": \"0.0.0.0:18081\",
          \"upstream\": \"inventory-service:8080\"
        }'
      "
```

### Injecting Failures

```csharp
// Tests/Chaos/ChaosTests.cs
public class ChaosTests : IClassFixture<ChaosWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly ToxiproxyClient _toxiproxy;

    [Fact]
    public async Task Order_Succeeds_With_Payment_Latency()
    {
        // Add 2 second latency to payment service
        await _toxiproxy.AddToxicAsync("payment", new LatencyToxic
        {
            Name = "payment-latency",
            Latency = 2000,
            Jitter = 500
        });

        try
        {
            var response = await _client.PostAsJsonAsync("/api/orders", new CreateOrderRequest
            {
                CustomerId = 1,
                Lines = [new OrderLine { ProductId = 1, Quantity = 1 }]
            });

            // Should succeed (resilience handles latency)
            Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        }
        finally
        {
            await _toxiproxy.RemoveToxicAsync("payment", "payment-latency");
        }
    }

    [Fact]
    public async Task Order_Succeeds_With_Inventory_Down()
    {
        // Cut connection to inventory service
        await _toxiproxy.AddToxicAsync("inventory", new ResetPeerToxic
        {
            Name = "inventory-down"
        });

        try
        {
            var response = await _client.PostAsJsonAsync("/api/orders", new CreateOrderRequest
            {
                CustomerId = 1,
                Lines = [new OrderLine { ProductId = 1, Quantity = 1 }]
            });

            // Should use fallback (cached inventory)
            Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        }
        finally
        {
            await _toxiproxy.RemoveToxicAsync("inventory", "inventory-down");
        }
    }

    [Fact]
    public async Task Circuit_Opens_After_Multiple_Failures()
    {
        // Make inventory return 500 errors
        await _toxiproxy.AddToxicAsync("inventory", new SlicerToxic
        {
            Name = "inventory-errors"
        });

        try
        {
            // Make enough requests to trigger circuit breaker
            for (int i = 0; i < 15; i++)
            {
                await _client.GetAsync("/api/inventory/1");
            }

            // Circuit should be open now - requests fail fast
            var sw = Stopwatch.StartNew();
            var response = await _client.GetAsync("/api/inventory/1");
            sw.Stop();

            // Should fail fast (< 100ms) not timeout (5s)
            Assert.True(sw.ElapsedMilliseconds < 100, "Should fail fast when circuit is open");
        }
        finally
        {
            await _toxiproxy.RemoveToxicAsync("inventory", "inventory-errors");
        }
    }
}
```

### CI/CD Chaos Integration

```yaml
# .github/workflows/ci.yml
jobs:
  resilience-tests:
    runs-on: ubuntu-latest
    services:
      toxiproxy:
        image: ghcr.io/shopify/toxiproxy:latest
        ports:
          - 8474:8474

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Start Dependencies
        run: docker compose up -d

      - name: Configure Toxiproxy
        run: |
          curl -X POST http://localhost:8474/proxies -d '{
            "name": "payment",
            "listen": "0.0.0.0:18080",
            "upstream": "localhost:8080"
          }'

      - name: Run Chaos Tests
        run: dotnet test --filter "Category=Chaos"
        env:
          PAYMENT_SERVICE_URL: http://localhost:18080
```

---

## Graceful Degradation

### Implementing Feature Toggles for Resilience

```csharp
public class OrderService
{
    private readonly IPaymentClient _paymentClient;
    private readonly IFeatureManager _featureManager;
    private readonly ILogger<OrderService> _logger;

    public async Task<OrderResult> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = await CreateOrderInternalAsync(request);

        // Primary path: Real-time payment processing
        if (await _featureManager.IsEnabledAsync("RealTimePayments"))
        {
            try
            {
                await _paymentClient.ProcessPaymentAsync(order);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Real-time payment failed, queueing for retry");

                // Graceful degradation: Queue for later processing
                await QueuePaymentForRetryAsync(order);

                return new OrderResult
                {
                    OrderId = order.Id,
                    Status = "PaymentPending",
                    Message = "Order created. Payment will be processed shortly."
                };
            }
        }
        else
        {
            // Feature disabled: Always queue payments
            await QueuePaymentForRetryAsync(order);
        }

        return new OrderResult
        {
            OrderId = order.Id,
            Status = "Confirmed"
        };
    }
}
```

### Health Checks for Dependencies

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddUrlGroup(
        new Uri(builder.Configuration["PaymentService:HealthUrl"]!),
        name: "payment-service",
        failureStatus: HealthStatus.Degraded)
    .AddUrlGroup(
        new Uri(builder.Configuration["InventoryService:HealthUrl"]!),
        name: "inventory-service",
        failureStatus: HealthStatus.Degraded)
    .AddCheck<CircuitBreakerHealthCheck>("circuit-breakers");

// Custom health check for circuit breaker states
public class CircuitBreakerHealthCheck : IHealthCheck
{
    private readonly ResiliencePipelineProvider<string> _pipelines;

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        var openCircuits = new List<string>();

        // Check each circuit breaker
        // Note: This requires tracking circuit breaker states

        if (openCircuits.Any())
        {
            return Task.FromResult(HealthCheckResult.Degraded(
                $"Open circuits: {string.Join(", ", openCircuits)}"));
        }

        return Task.FromResult(HealthCheckResult.Healthy());
    }
}
```

---

## Hands-On Exercise: Implement Resilience for OrderFlow

### Step 1: Add Resilience to HTTP Clients

Configure resilience for all external service calls:
- Payment service: Critical, needs careful retry/circuit breaker
- Inventory service: Can use cached fallback
- Notification service: Fire-and-forget acceptable

### Step 2: Implement Chaos Tests

Create tests that:
1. Verify orders succeed with 2s payment latency
2. Verify fallback works when inventory is down
3. Verify circuit breaker opens after failures

### Step 3: Add Resilience Monitoring

- Log all retry attempts
- Track circuit breaker state changes
- Alert when fallbacks are being used frequently

### Step 4: Document Resilience Strategy

```markdown
# OrderFlow Resilience Strategy

## External Dependencies

### Payment Service
- **Criticality**: High (orders can't complete without payment)
- **Timeout**: 10s per attempt, 30s total
- **Retry**: 3 attempts with exponential backoff
- **Circuit Breaker**: 50% failure rate over 10s
- **Fallback**: Queue for async processing

### Inventory Service
- **Criticality**: Medium (can use cached data)
- **Timeout**: 5s per attempt
- **Retry**: 2 attempts
- **Circuit Breaker**: 50% failure rate over 10s
- **Fallback**: Return cached inventory levels

### Notification Service
- **Criticality**: Low (best effort)
- **Timeout**: 3s
- **Retry**: 1 attempt
- **Circuit Breaker**: None
- **Fallback**: Log and continue
```

---

## Deliverables

1. **Resilience Configuration**: Polly pipelines for all HTTP clients
2. **Chaos Test Suite**: Tests proving resilience under failure
3. **Monitoring Dashboard**: Grafana dashboard for resilience metrics
4. **Documentation**: Resilience strategy document

---

## Reflection Questions

1. How do you decide timeout values for different services?
2. When would you NOT use a circuit breaker?
3. How do you test resilience without affecting production?
4. What's the cost of too much resilience?

---

## Resources

### Polly
- [Polly Documentation](https://www.pollydocs.org/)
- [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/)
- [Dylan Reisenberger - Polly Patterns](https://github.com/App-vNext/Polly/wiki)

### Chaos Engineering
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Toxiproxy](https://github.com/Shopify/toxiproxy)
- [Netflix Chaos Monkey](https://netflix.github.io/chaosmonkey/)

### Books
- [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/) by Michael Nygard
- [Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/) - Chapter 22: Cascading Failures

---

## Module Summary

You've completed the Architecture Patterns & Distributed Systems module. You can now:

- Choose and implement appropriate architecture patterns
- Model complex domains using DDD techniques
- Make informed decisions about service boundaries
- Implement resilience patterns for production systems
- Test system behavior under failure conditions

**Remember**: Good architecture isn't about following patterns — it's about making deliberate trade-offs that serve your team and users.

---

**Next Module:** [Module 11 — DevOps & Deployment](../11-devops/README.md)
