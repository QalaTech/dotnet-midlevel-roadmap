# Part 2 — Scaling APIs Horizontally

## Why This Matters

Your app works great with 100 users. What happens with 10,000?

Vertical scaling (bigger servers) has limits:
- Hardware maxes out
- Single point of failure
- Expensive at scale
- Can't scale instantly

Horizontal scaling (more servers) is how modern systems grow:
- Add capacity on demand
- Survive server failures
- Cost-effective at scale
- But requires **stateless design**

---

## What Goes Wrong Without This

### The Session State Catastrophe

```
Setup: 3 API instances behind a load balancer
Problem: Session stored in memory

Request 1 → Instance A: Login successful, session created
Request 2 → Instance B: "Please log in" (no session!)
Request 3 → Instance A: "Welcome back"
Request 4 → Instance C: "Please log in"

User: "This app is broken!"
```

### The In-Memory Cache That Couldn't

```
Cache Strategy: Store hot data in MemoryCache

Instance A cache: { "product:123": {...} }
Instance B cache: { }  (empty, different process)
Instance C cache: { "product:123": {...stale...} }

Result: Inconsistent responses, confused users
Fix attempt: "Just invalidate the cache"
Reality: Which instance? You can't broadcast.
```

### The File Upload Fiasco

```
Day 1: User uploads file to Instance A
Day 2: User requests file, hits Instance B
Result: 404 Not Found

"But I just uploaded it!"

The file was in local storage on Instance A.
Instance B has no idea it exists.
```

---

## Stateless Design Principles

### What "Stateless" Actually Means

Stateless doesn't mean "no state." It means:

> Any instance can handle any request without prior knowledge of that client.

State still exists — it's just externalized:

| State Type | Bad (In-Process) | Good (Externalized) |
|------------|------------------|---------------------|
| Session | `HttpContext.Session` (in-memory) | Redis, database |
| Cache | `MemoryCache` | Redis, distributed cache |
| Files | Local disk | Blob storage (S3, Azure Blob) |
| Locks | `lock` keyword | Redis distributed locks |
| Rate limits | In-memory counters | Redis counters |

### Audit Your Application for State

```csharp
// Checklist: Search your codebase for these patterns

// 1. Static mutable state
public static class OrderCache  // RED FLAG
{
    private static Dictionary<int, Order> _cache = new();  // Process-local!
}

// 2. In-memory session
services.AddSession();  // Where is session stored?

// 3. Local file operations
File.WriteAllText("temp/order.json", data);  // Not shared!

// 4. In-memory rate limiting
private static int _requestCount = 0;  // Process-local!

// 5. SignalR without backplane
services.AddSignalR();  // Needs Redis for scale!
```

### Refactoring to Stateless

```csharp
// Before: In-memory cache (doesn't scale)
public class OrderService
{
    private static readonly ConcurrentDictionary<int, Order> _cache = new();

    public async Task<Order?> GetOrderAsync(int id)
    {
        if (_cache.TryGetValue(id, out var cached))
            return cached;

        var order = await _repository.GetOrderAsync(id);
        if (order != null)
            _cache[id] = order;
        return order;
    }
}

// After: Distributed cache (scales horizontally)
public class OrderService
{
    private readonly IDistributedCache _cache;
    private readonly IOrderRepository _repository;
    private readonly JsonSerializerOptions _jsonOptions;

    public OrderService(
        IDistributedCache cache,
        IOrderRepository repository)
    {
        _cache = cache;
        _repository = repository;
        _jsonOptions = new JsonSerializerOptions();
    }

    public async Task<Order?> GetOrderAsync(int id)
    {
        var cacheKey = $"order:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
            return JsonSerializer.Deserialize<Order>(cached, _jsonOptions);

        var order = await _repository.GetOrderAsync(id);

        if (order != null)
        {
            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(order, _jsonOptions),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
                });
        }

        return order;
    }
}

// Registration
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = configuration.GetConnectionString("Redis");
    options.InstanceName = "OrderFlow:";
});
```

---

## Database Connection Pooling

### Why Pooling Matters

Each database connection:
- Takes time to establish (TCP handshake, authentication)
- Consumes server resources (memory, process slots)
- Has limits (PostgreSQL default: 100 connections)

With 5 API instances at 20 connections each = 100 connections = pool exhausted.

### EF Core Connection Pooling

```csharp
// appsettings.json - Connection string pooling
{
    "ConnectionStrings": {
        "Default": "Host=localhost;Database=orderflow;Username=app;Password=secret;Maximum Pool Size=30;Connection Idle Lifetime=300;Connection Pruning Interval=10"
    }
}

// Program.cs - DbContext pooling
services.AddDbContextPool<OrderDbContext>(options =>
{
    options.UseNpgsql(connectionString);
}, poolSize: 128);  // Pool of pre-configured DbContext instances
```

### Connection Pool Monitoring

```csharp
public class ConnectionPoolHealthCheck : IHealthCheck
{
    private readonly OrderDbContext _context;
    private readonly ILogger<ConnectionPoolHealthCheck> _logger;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        try
        {
            // Test connection
            await _context.Database.CanConnectAsync(ct);

            // Get pool statistics (Npgsql specific)
            var dataSource = _context.Database.GetDbConnection() as NpgsqlConnection;
            if (dataSource?.DataSource is NpgsqlDataSource ds)
            {
                var stats = ds.Statistics;
                var data = new Dictionary<string, object>
                {
                    ["idle"] = stats.Idle,
                    ["busy"] = stats.Busy,
                    ["total"] = stats.Total,
                    ["max"] = stats.Max
                };

                // Warn if pool is > 80% utilized
                var utilization = (double)stats.Busy / stats.Max;
                if (utilization > 0.8)
                {
                    return HealthCheckResult.Degraded(
                        $"Connection pool at {utilization:P0} capacity",
                        data: data);
                }

                return HealthCheckResult.Healthy("Connection pool healthy", data);
            }

            return HealthCheckResult.Healthy();
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database connection failed", ex);
        }
    }
}
```

### Connection Pool Anti-Patterns

```csharp
// Anti-pattern: Forgetting to dispose connections
public async Task<Order> GetOrderBad(int id)
{
    var context = new OrderDbContext(_options);  // Never disposed!
    return await context.Orders.FindAsync(id);
}

// Anti-pattern: Long-running transactions holding connections
public async Task ProcessOrders()
{
    using var transaction = await _context.Database.BeginTransactionAsync();

    foreach (var order in await _context.Orders.ToListAsync())
    {
        await ProcessOrderAsync(order);  // 30 seconds each
        await _context.SaveChangesAsync();
    }

    await transaction.CommitAsync();
    // Connection held for entire batch duration!
}

// Better: Batch with shorter transactions
public async Task ProcessOrdersOptimized()
{
    var orderIds = await _context.Orders
        .Select(o => o.Id)
        .ToListAsync();

    foreach (var batch in orderIds.Chunk(100))
    {
        using var transaction = await _context.Database.BeginTransactionAsync();

        foreach (var id in batch)
        {
            var order = await _context.Orders.FindAsync(id);
            await ProcessOrderAsync(order!);
        }

        await _context.SaveChangesAsync();
        await transaction.CommitAsync();
        // Connection released after each batch
    }
}
```

---

## Load Testing with k6

### Why k6?

- Written in Go (fast, efficient)
- Scripts in JavaScript (familiar)
- Great metrics and thresholds
- Free and open source
- CI/CD friendly

### Installing k6

```bash
# macOS
brew install k6

# Windows
choco install k6

# Linux
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6
```

### Basic Load Test Script

```javascript
// tests/load/orders-api.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const orderCreationTime = new Trend('order_creation_time');

// Test configuration
export const options = {
    stages: [
        { duration: '1m', target: 50 },   // Ramp up to 50 users
        { duration: '3m', target: 50 },   // Stay at 50 users
        { duration: '1m', target: 100 },  // Ramp up to 100 users
        { duration: '3m', target: 100 },  // Stay at 100 users
        { duration: '1m', target: 0 },    // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500', 'p(99)<1000'],  // 95% under 500ms
        errors: ['rate<0.01'],                           // Error rate under 1%
        order_creation_time: ['p(95)<800'],              // Order creation under 800ms
    },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:5000';

// Reusable authentication
function getAuthHeaders() {
    // In real tests, implement proper auth token fetching
    return {
        'Authorization': `Bearer ${__ENV.API_TOKEN}`,
        'Content-Type': 'application/json',
    };
}

// Test scenarios
export default function () {
    // Scenario: Create an order
    const orderPayload = JSON.stringify({
        customerId: Math.floor(Math.random() * 1000) + 1,
        lines: [
            {
                productId: Math.floor(Math.random() * 100) + 1,
                quantity: Math.floor(Math.random() * 5) + 1,
            },
        ],
    });

    const createStart = Date.now();
    const createResponse = http.post(
        `${BASE_URL}/api/orders`,
        orderPayload,
        { headers: getAuthHeaders() }
    );
    orderCreationTime.add(Date.now() - createStart);

    const createSuccess = check(createResponse, {
        'order created': (r) => r.status === 201,
        'has order id': (r) => r.json('id') !== undefined,
    });
    errorRate.add(!createSuccess);

    if (createSuccess) {
        const orderId = createResponse.json('id');

        // Scenario: Get the order
        sleep(1);
        const getResponse = http.get(
            `${BASE_URL}/api/orders/${orderId}`,
            { headers: getAuthHeaders() }
        );

        check(getResponse, {
            'order retrieved': (r) => r.status === 200,
            'correct order id': (r) => r.json('id') === orderId,
        });
    }

    sleep(1); // Think time between iterations
}

// Setup: runs once before test
export function setup() {
    console.log(`Testing against: ${BASE_URL}`);

    // Health check
    const healthResponse = http.get(`${BASE_URL}/health`);
    if (healthResponse.status !== 200) {
        throw new Error('API is not healthy');
    }

    return { startTime: Date.now() };
}

// Teardown: runs once after test
export function teardown(data) {
    const duration = (Date.now() - data.startTime) / 1000;
    console.log(`Test completed in ${duration}s`);
}
```

### Running Load Tests

```bash
# Basic run
k6 run tests/load/orders-api.js

# With environment variables
k6 run -e BASE_URL=https://api.staging.example.com -e API_TOKEN=xxx tests/load/orders-api.js

# Output to JSON for CI/CD
k6 run --out json=results.json tests/load/orders-api.js

# HTML report (using k6 extension)
k6 run --out web-dashboard tests/load/orders-api.js
```

### Load Test Results Analysis

```
          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/

  execution: local
     script: tests/load/orders-api.js
     output: -

  scenarios: (100.00%) 1 scenario, 100 max VUs, 10m0s max duration
           default: Up to 100 looping VUs for 9m0s

running (9m00s), 000/100 VUs, 12847 complete iterations
default ✓ [======================================] 000/100 VUs  9m0s

     ✓ order created
     ✓ has order id
     ✓ order retrieved
     ✓ correct order id

     checks.........................: 100.00% ✓ 51388      ✗ 0
     data_received..................: 15 MB   28 kB/s
     data_sent......................: 4.2 MB  7.8 kB/s
     errors.........................: 0.00%   ✓ 0          ✗ 12847
   ✓ http_req_duration..............: avg=89ms  min=12ms  med=67ms  max=892ms p(90)=198ms p(95)=287ms
     http_reqs......................: 25694   47.58/s
     iteration_duration.............: avg=2.1s  min=1.02s med=2.0s  max=4.9s  p(90)=2.4s  p(95)=2.8s
   ✓ order_creation_time............: avg=134ms min=23ms  med=98ms  max=892ms p(90)=298ms p(95)=412ms
     vus............................: 1       min=1        max=100
     vus_max........................: 100     min=100      max=100
```

---

## Autoscaling Strategies

### Kubernetes Horizontal Pod Autoscaler (HPA)

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orderflow-api-hpa
  namespace: orderflow
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orderflow-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    # Scale on CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # Scale on memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Scale on custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 1000
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

### KEDA (Kubernetes Event-Driven Autoscaling)

```yaml
# k8s/keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: orderflow-worker-scaler
  namespace: orderflow
spec:
  scaleTargetRef:
    name: orderflow-worker
  minReplicaCount: 1
  maxReplicaCount: 20
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
    # Scale based on RabbitMQ queue length
    - type: rabbitmq
      metadata:
        host: amqp://rabbitmq.orderflow.svc.cluster.local
        queueName: order-processing
        queueLength: "50"  # Scale when > 50 messages
    # Scale based on PostgreSQL query
    - type: postgresql
      metadata:
        connectionString: "postgresql://..."
        query: "SELECT COUNT(*) FROM orders WHERE status = 'pending'"
        targetQueryValue: "100"
```

### Azure Container Apps Scaling

```bicep
// infra/containerapp.bicep
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'orderflow-api'
  location: location
  properties: {
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
      }
    }
    template: {
      containers: [
        {
          name: 'api'
          image: 'orderflow.azurecr.io/api:latest'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
      scale: {
        minReplicas: 2
        maxReplicas: 10
        rules: [
          {
            name: 'http-scaling'
            http: {
              metadata: {
                concurrentRequests: '100'
              }
            }
          }
          {
            name: 'cpu-scaling'
            custom: {
              type: 'cpu'
              metadata: {
                type: 'Utilization'
                value: '70'
              }
            }
          }
        ]
      }
    }
  }
}
```

---

## Load Balancing Patterns

### Health Check Endpoints

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddNpgSql(connectionString, name: "database")
    .AddRedis(redisConnection, name: "cache")
    .AddRabbitMQ(rabbitConnection, name: "messaging");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Liveness: Is the process alive?
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // No checks, just confirms process is running
});

// Readiness: Can it handle traffic?
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

### Graceful Shutdown

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Configure graceful shutdown
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

var app = builder.Build();

// Handle shutdown
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();

lifetime.ApplicationStopping.Register(() =>
{
    // Stop accepting new requests
    // Wait for in-flight requests to complete
    app.Logger.LogInformation("Application is stopping...");
});

lifetime.ApplicationStopped.Register(() =>
{
    app.Logger.LogInformation("Application has stopped");
});

app.Run();
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow-api
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: api
          lifecycle:
            preStop:
              exec:
                # Wait for load balancer to remove pod
                command: ["sleep", "5"]
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

## Hands-On Exercise: Scale OrderFlow

### Step 1: Identify State in Your Application

Create a checklist:

```markdown
# OrderFlow State Audit

## In-Memory State Found
- [ ] MemoryCache usage in ProductService
- [ ] Static dictionary in RateLimiter
- [ ] Session state in checkout flow

## External State (Already OK)
- [x] Database (PostgreSQL)
- [x] Message queue (RabbitMQ)

## Needs Migration
- [ ] MemoryCache → Redis
- [ ] Local file uploads → Azure Blob Storage
- [ ] In-memory rate limiting → Redis
```

### Step 2: Run Load Tests

```bash
# Baseline: Single instance
k6 run tests/load/orders-api.js

# Record baseline metrics:
# - p95 latency: ___ms
# - Max RPS: ___
# - Error rate: ___%

# Scale to 3 instances and re-test
kubectl scale deployment orderflow-api --replicas=3
k6 run tests/load/orders-api.js

# Compare metrics
```

### Step 3: Implement Autoscaling

```yaml
# Deploy HPA
kubectl apply -f k8s/hpa.yaml

# Monitor scaling during load test
watch kubectl get hpa orderflow-api-hpa

# In another terminal
k6 run --vus 200 --duration 10m tests/load/orders-api.js
```

### Step 4: Document Your Scaling Architecture

```markdown
# OrderFlow Scaling Architecture

## Current Capacity
- Minimum instances: 2
- Maximum instances: 10
- Requests per instance: ~500 RPS

## Scaling Triggers
- CPU > 70%: Scale up
- CPU < 30% for 5 min: Scale down
- Queue depth > 100: Scale workers

## Bottlenecks Identified
1. Database connections (max 100)
   - Mitigation: Connection pooling, read replicas
2. Redis memory
   - Mitigation: Eviction policy, cluster mode
```

---

## Common Scaling Anti-Patterns

### Anti-Pattern: Sticky Sessions

```csharp
// Anti-pattern: Relying on session affinity
// Load balancer config: "sticky sessions = true"

// This breaks when:
// - Instance crashes (session lost)
// - Instance scales down (session lost)
// - Uneven load distribution

// Better: Externalize session state
services.AddStackExchangeRedisCache(options => {
    options.Configuration = "redis:6379";
});

services.AddSession(options => {
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});
```

### Anti-Pattern: Assuming Instance Count

```csharp
// Anti-pattern: Scheduling based on instance count
public class ScheduledJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            // Runs on EVERY instance = duplicates!
            await ProcessPendingOrdersAsync();
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}

// Better: Use distributed locking or dedicated scheduler
public class ScheduledJob : BackgroundService
{
    private readonly IDistributedLock _lock;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var handle = await _lock.TryAcquireAsync("order-processing");
            if (handle != null)
            {
                await ProcessPendingOrdersAsync();
            }
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

---

## Deliverables

1. **State Audit**: Document all in-memory state and migration plan
2. **Load Test Suite**: k6 scripts for key endpoints with thresholds
3. **Autoscaling Config**: HPA or equivalent configuration
4. **Architecture Diagram**: Show scaling topology and bottlenecks

---

## Reflection Questions

1. What's the difference between horizontal and vertical scaling?
2. When would sticky sessions be acceptable?
3. How do you test autoscaling without affecting production?
4. What's the cost trade-off of scaling out vs scaling up?

---

## Resources

### Load Testing
- [k6 Documentation](https://k6.io/docs/)
- [k6 Best Practices](https://k6.io/docs/testing-guides/api-load-testing/)
- [Grafana k6 YouTube](https://www.youtube.com/c/k6test)

### Kubernetes Scaling
- [HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [KEDA Documentation](https://keda.sh/docs/)
- [Kubernetes Patterns](https://www.redhat.com/en/resources/oreilly-kubernetes-patterns-e-book)

### Azure
- [Container Apps Scaling](https://learn.microsoft.com/en-us/azure/container-apps/scale-app)
- [Azure Load Testing](https://learn.microsoft.com/en-us/azure/load-testing/)

---

**Next:** [Part 3 — Observability Deep Dive](./part-3.md)
