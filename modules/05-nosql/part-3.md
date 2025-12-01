# 05 — NoSQL & Distributed Data

## Part 3 / 3 — Redis Caching & Distributed Primitives

---

## Why This Matters

Redis is often the first distributed system developers touch. It's deceptively simple — but misusing it causes:
- Cache stampedes that take down your database
- Stale data shown to users
- Race conditions in distributed locks
- Memory exhaustion in production

Understanding Redis patterns is essential for building reliable, performant applications.

---

## Core Concept #10 — Caching Fundamentals

### Why This Matters

Caching improves performance dramatically — a cache hit is 100x faster than a database query. But cache invalidation is one of the two hard problems in computer science.

### What Goes Wrong Without This

**War story #1 — The Stale Price Problem:**
E-commerce site caches product prices for 1 hour. Marketing runs a flash sale. Customers see old prices. Some pay more than advertised. Legal gets involved. Cost: $50,000 in refunds.

**War story #2 — The Cache Stampede:**
Cache expires. 1000 concurrent requests hit the database for the same data. Database CPU hits 100%. Site goes down for 10 minutes.

### Caching Patterns

```csharp
// Cache-Aside (most common pattern)
public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repository;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(15);

    public async Task<Product?> GetByIdAsync(string id)
    {
        var cacheKey = $"product:{id}";

        // Try cache first
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<Product>(cached);
        }

        // Cache miss — load from database
        var product = await _repository.GetByIdAsync(id);
        if (product == null)
        {
            return null;
        }

        // Store in cache
        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(product),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _cacheDuration
            });

        return product;
    }

    // Invalidate on update
    public async Task UpdateAsync(Product product)
    {
        await _repository.UpdateAsync(product);
        await _cache.RemoveAsync($"product:{product.Id}");
    }
}
```

### Preventing Cache Stampede

```csharp
public class StampedeProtectedCache
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;
    private readonly SemaphoreSlim _localLock = new(1, 1);

    public async Task<T?> GetOrSetAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan expiry)
    {
        // Try cache first
        var cached = await _db.StringGetAsync(key);
        if (!cached.IsNullOrEmpty)
        {
            return JsonSerializer.Deserialize<T>(cached!);
        }

        // Use distributed lock to prevent stampede
        var lockKey = $"{key}:lock";
        var lockValue = Guid.NewGuid().ToString();
        var lockExpiry = TimeSpan.FromSeconds(30);

        // Try to acquire lock
        var acquired = await _db.StringSetAsync(
            lockKey,
            lockValue,
            lockExpiry,
            When.NotExists);

        if (acquired)
        {
            try
            {
                // Double-check cache (another process might have filled it)
                cached = await _db.StringGetAsync(key);
                if (!cached.IsNullOrEmpty)
                {
                    return JsonSerializer.Deserialize<T>(cached!);
                }

                // Load from source
                var value = await factory();

                // Store in cache
                await _db.StringSetAsync(
                    key,
                    JsonSerializer.Serialize(value),
                    expiry);

                return value;
            }
            finally
            {
                // Release lock (only if we still own it)
                var script = @"
                    if redis.call('get', KEYS[1]) == ARGV[1] then
                        return redis.call('del', KEYS[1])
                    else
                        return 0
                    end";

                await _db.ScriptEvaluateAsync(
                    script,
                    new RedisKey[] { lockKey },
                    new RedisValue[] { lockValue });
            }
        }
        else
        {
            // Someone else is loading — wait and retry
            await Task.Delay(100);
            return await GetOrSetAsync(key, factory, expiry);
        }
    }
}
```

### Cache Warming and Refresh-Ahead

```csharp
public class CacheWarmingService : BackgroundService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IProductRepository _repository;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Refresh popular products before they expire
                var popularProductIds = await GetPopularProductIdsAsync();

                foreach (var id in popularProductIds)
                {
                    var cacheKey = $"product:{id}";
                    var ttl = await _redis.GetDatabase().KeyTimeToLiveAsync(cacheKey);

                    // Refresh if expiring within 5 minutes
                    if (ttl.HasValue && ttl.Value < TimeSpan.FromMinutes(5))
                    {
                        var product = await _repository.GetByIdAsync(id);
                        if (product != null)
                        {
                            await _redis.GetDatabase().StringSetAsync(
                                cacheKey,
                                JsonSerializer.Serialize(product),
                                TimeSpan.FromMinutes(15));
                        }
                    }
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Cache warming failed");
            }

            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}
```

---

## Core Concept #11 — Redis Data Structures

### Why This Matters

Redis isn't just a key-value store. Its data structures solve specific problems efficiently:

| Structure | Use Case | Example |
|-----------|----------|---------|
| String | Caching, counters | Page cache, rate limiting |
| Hash | Object storage | User sessions |
| List | Queues, logs | Task queue, recent activity |
| Set | Unique items, tags | Online users, feature flags |
| Sorted Set | Rankings, time-series | Leaderboards, scheduled tasks |
| Stream | Event streaming | Activity feed, logging |

### Practical Examples

```csharp
public class RedisExamples
{
    private readonly IDatabase _db;

    // STRING: Simple caching with atomic operations
    public async Task<long> IncrementViewCountAsync(string productId)
    {
        return await _db.StringIncrementAsync($"views:{productId}");
    }

    // HASH: Store objects efficiently
    public async Task SetSessionAsync(string sessionId, UserSession session)
    {
        var entries = new HashEntry[]
        {
            new("userId", session.UserId),
            new("email", session.Email),
            new("roles", string.Join(",", session.Roles)),
            new("createdAt", session.CreatedAt.ToString("O"))
        };

        await _db.HashSetAsync($"session:{sessionId}", entries);
        await _db.KeyExpireAsync($"session:{sessionId}", TimeSpan.FromHours(24));
    }

    public async Task<UserSession?> GetSessionAsync(string sessionId)
    {
        var entries = await _db.HashGetAllAsync($"session:{sessionId}");
        if (entries.Length == 0) return null;

        var dict = entries.ToDictionary(
            e => e.Name.ToString(),
            e => e.Value.ToString());

        return new UserSession
        {
            UserId = dict["userId"],
            Email = dict["email"],
            Roles = dict["roles"].Split(','),
            CreatedAt = DateTime.Parse(dict["createdAt"])
        };
    }

    // LIST: Recent activity feed
    public async Task AddToActivityFeedAsync(string userId, string activity)
    {
        var key = $"feed:{userId}";

        // Push to front, keep only last 100
        await _db.ListLeftPushAsync(key, activity);
        await _db.ListTrimAsync(key, 0, 99);
    }

    public async Task<string[]> GetRecentActivityAsync(string userId, int count = 10)
    {
        var items = await _db.ListRangeAsync($"feed:{userId}", 0, count - 1);
        return items.Select(i => i.ToString()).ToArray();
    }

    // SET: Online users tracking
    public async Task UserCameOnlineAsync(string userId)
    {
        await _db.SetAddAsync("online:users", userId);
    }

    public async Task UserWentOfflineAsync(string userId)
    {
        await _db.SetRemoveAsync("online:users", userId);
    }

    public async Task<long> GetOnlineUserCountAsync()
    {
        return await _db.SetLengthAsync("online:users");
    }

    public async Task<bool> IsUserOnlineAsync(string userId)
    {
        return await _db.SetContainsAsync("online:users", userId);
    }

    // SORTED SET: Leaderboard
    public async Task UpdateScoreAsync(string userId, double score)
    {
        await _db.SortedSetAddAsync("leaderboard", userId, score);
    }

    public async Task<LeaderboardEntry[]> GetTopPlayersAsync(int count = 10)
    {
        var entries = await _db.SortedSetRangeByRankWithScoresAsync(
            "leaderboard",
            0,
            count - 1,
            Order.Descending);

        return entries.Select((e, i) => new LeaderboardEntry
        {
            Rank = i + 1,
            UserId = e.Element.ToString(),
            Score = e.Score
        }).ToArray();
    }

    public async Task<long?> GetUserRankAsync(string userId)
    {
        return await _db.SortedSetRankAsync("leaderboard", userId, Order.Descending);
    }
}
```

---

## Core Concept #12 — Distributed Locks

### Why This Matters

In distributed systems, you often need to ensure only one process performs an action:
- Processing a payment (don't double-charge)
- Sending notifications (don't spam users)
- Running scheduled jobs (don't run twice)

### What Goes Wrong Without This

**Scenario:** Two servers try to process the same payment simultaneously. Both check "is payment processed?" — both see "no". Both charge the customer. Customer is charged twice.

### Implementing Distributed Locks

```csharp
public class RedisDistributedLock : IAsyncDisposable
{
    private readonly IDatabase _db;
    private readonly string _key;
    private readonly string _value;
    private readonly TimeSpan _expiry;
    private bool _released;

    private RedisDistributedLock(
        IDatabase db,
        string key,
        string value,
        TimeSpan expiry)
    {
        _db = db;
        _key = key;
        _value = value;
        _expiry = expiry;
    }

    public static async Task<RedisDistributedLock?> TryAcquireAsync(
        IDatabase db,
        string resource,
        TimeSpan expiry,
        TimeSpan? timeout = null)
    {
        var key = $"lock:{resource}";
        var value = Guid.NewGuid().ToString();
        var deadline = DateTime.UtcNow + (timeout ?? TimeSpan.FromSeconds(10));

        while (DateTime.UtcNow < deadline)
        {
            var acquired = await db.StringSetAsync(
                key,
                value,
                expiry,
                When.NotExists);

            if (acquired)
            {
                return new RedisDistributedLock(db, key, value, expiry);
            }

            // Wait a bit before retrying
            await Task.Delay(50);
        }

        return null; // Failed to acquire within timeout
    }

    public async Task<bool> ExtendAsync(TimeSpan additionalTime)
    {
        // Only extend if we still own the lock
        var script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('pexpire', KEYS[1], ARGV[2])
            else
                return 0
            end";

        var result = await _db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { _key },
            new RedisValue[] { _value, (long)additionalTime.TotalMilliseconds });

        return (int)result == 1;
    }

    public async ValueTask DisposeAsync()
    {
        if (_released) return;

        // Only release if we still own the lock (atomic check-and-delete)
        var script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end";

        await _db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { _key },
            new RedisValue[] { _value });

        _released = true;
    }
}

// Usage
public class PaymentService
{
    public async Task<PaymentResult> ProcessPaymentAsync(string orderId, decimal amount)
    {
        await using var lockHandle = await RedisDistributedLock.TryAcquireAsync(
            _db,
            $"payment:{orderId}",
            expiry: TimeSpan.FromMinutes(5),
            timeout: TimeSpan.FromSeconds(30));

        if (lockHandle == null)
        {
            return PaymentResult.Conflict("Payment already being processed");
        }

        // Check if already processed (idempotency)
        var existing = await _paymentRepository.GetByOrderIdAsync(orderId);
        if (existing != null)
        {
            return PaymentResult.AlreadyProcessed(existing);
        }

        // Process payment
        var result = await _paymentGateway.ChargeAsync(orderId, amount);

        // Store result
        await _paymentRepository.SaveAsync(new Payment
        {
            OrderId = orderId,
            Amount = amount,
            Status = result.Success ? "Completed" : "Failed",
            ProcessedAt = DateTime.UtcNow
        });

        return result;
    }
}
```

### Rate Limiting with Redis

```csharp
public class RedisRateLimiter
{
    private readonly IDatabase _db;

    // Sliding window rate limiter
    public async Task<RateLimitResult> CheckRateLimitAsync(
        string key,
        int maxRequests,
        TimeSpan window)
    {
        var now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var windowStart = now - (long)window.TotalMilliseconds;
        var redisKey = $"ratelimit:{key}";

        // Lua script for atomic rate limiting
        var script = @"
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window_start = tonumber(ARGV[2])
            local max_requests = tonumber(ARGV[3])
            local window_ms = tonumber(ARGV[4])

            -- Remove old entries
            redis.call('zremrangebyscore', key, '-inf', window_start)

            -- Count current requests
            local current = redis.call('zcard', key)

            if current < max_requests then
                -- Add this request
                redis.call('zadd', key, now, now .. ':' .. math.random())
                redis.call('pexpire', key, window_ms)
                return {1, max_requests - current - 1, 0}
            else
                -- Get oldest entry to calculate retry time
                local oldest = redis.call('zrange', key, 0, 0, 'WITHSCORES')
                local retry_after = oldest[2] + window_ms - now
                return {0, 0, retry_after}
            end";

        var result = (RedisResult[])await _db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { redisKey },
            new RedisValue[] { now, windowStart, maxRequests, (long)window.TotalMilliseconds });

        return new RateLimitResult
        {
            Allowed = (int)result[0] == 1,
            Remaining = (int)result[1],
            RetryAfterMs = (long)result[2]
        };
    }
}

// ASP.NET Core middleware
public class RateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly RedisRateLimiter _limiter;

    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = context.User.Identity?.Name ??
                       context.Connection.RemoteIpAddress?.ToString() ??
                       "anonymous";

        var result = await _limiter.CheckRateLimitAsync(
            clientId,
            maxRequests: 100,
            window: TimeSpan.FromMinutes(1));

        context.Response.Headers["X-RateLimit-Remaining"] = result.Remaining.ToString();

        if (!result.Allowed)
        {
            context.Response.Headers["Retry-After"] =
                (result.RetryAfterMs / 1000).ToString();
            context.Response.StatusCode = 429;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Rate limit exceeded",
                retryAfterSeconds = result.RetryAfterMs / 1000
            });
            return;
        }

        await _next(context);
    }
}
```

---

## Core Concept #13 — Pub/Sub and Event Streaming

### Why This Matters

Redis can distribute events across services for:
- Real-time notifications
- Cache invalidation
- Distributed event handling

### Redis Pub/Sub

```csharp
public class RedisPubSub
{
    private readonly IConnectionMultiplexer _redis;

    // Publisher
    public async Task PublishAsync<T>(string channel, T message)
    {
        var subscriber = _redis.GetSubscriber();
        var json = JsonSerializer.Serialize(message);
        await subscriber.PublishAsync(channel, json);
    }

    // Subscriber
    public async Task SubscribeAsync<T>(
        string channel,
        Func<T, Task> handler,
        CancellationToken ct)
    {
        var subscriber = _redis.GetSubscriber();

        await subscriber.SubscribeAsync(channel, async (_, message) =>
        {
            try
            {
                var data = JsonSerializer.Deserialize<T>(message!);
                await handler(data!);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error handling message on {Channel}", channel);
            }
        });

        // Keep subscription alive
        await Task.Delay(Timeout.Infinite, ct);
    }
}

// Cache invalidation via Pub/Sub
public class DistributedCacheInvalidator
{
    private readonly RedisPubSub _pubsub;
    private readonly IMemoryCache _localCache;

    public async Task InvalidateAsync(string cacheKey)
    {
        // Remove from local cache
        _localCache.Remove(cacheKey);

        // Broadcast to all instances
        await _pubsub.PublishAsync("cache:invalidate", new
        {
            Key = cacheKey,
            Timestamp = DateTime.UtcNow
        });
    }

    public async Task StartListeningAsync(CancellationToken ct)
    {
        await _pubsub.SubscribeAsync<CacheInvalidation>(
            "cache:invalidate",
            invalidation =>
            {
                _localCache.Remove(invalidation.Key);
                return Task.CompletedTask;
            },
            ct);
    }
}
```

### Redis Streams (Better for Reliability)

```csharp
public class RedisStreamService
{
    private readonly IDatabase _db;

    // Add to stream
    public async Task<string> PublishEventAsync(
        string stream,
        Dictionary<string, string> eventData)
    {
        var entries = eventData.Select(kv =>
            new NameValueEntry(kv.Key, kv.Value)).ToArray();

        var id = await _db.StreamAddAsync(stream, entries);
        return id.ToString();
    }

    // Consume with consumer group (at-least-once delivery)
    public async Task ConsumeAsync(
        string stream,
        string group,
        string consumer,
        Func<StreamEntry, Task> handler,
        CancellationToken ct)
    {
        // Create consumer group if it doesn't exist
        try
        {
            await _db.StreamCreateConsumerGroupAsync(stream, group, "0");
        }
        catch (RedisServerException ex) when (ex.Message.Contains("BUSYGROUP"))
        {
            // Group already exists
        }

        while (!ct.IsCancellationRequested)
        {
            // Read new messages
            var entries = await _db.StreamReadGroupAsync(
                stream,
                group,
                consumer,
                ">",  // Only undelivered messages
                count: 10);

            foreach (var entry in entries)
            {
                try
                {
                    await handler(entry);

                    // Acknowledge successful processing
                    await _db.StreamAcknowledgeAsync(stream, group, entry.Id);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex,
                        "Error processing stream entry {EntryId}",
                        entry.Id);
                    // Message will be re-delivered to another consumer
                }
            }

            if (entries.Length == 0)
            {
                await Task.Delay(100, ct);
            }
        }
    }

    // Claim pending messages (dead letter handling)
    public async Task ClaimPendingMessagesAsync(
        string stream,
        string group,
        string consumer,
        TimeSpan minIdleTime)
    {
        var pending = await _db.StreamPendingMessagesAsync(
            stream,
            group,
            10,
            consumerName: RedisValue.Null);

        foreach (var msg in pending.Where(m => m.IdleTimeInMilliseconds > minIdleTime.TotalMilliseconds))
        {
            // Claim the message for this consumer
            var claimed = await _db.StreamClaimAsync(
                stream,
                group,
                consumer,
                (long)minIdleTime.TotalMilliseconds,
                new[] { msg.MessageId });

            _logger.LogWarning(
                "Claimed abandoned message {MessageId} from {Stream}",
                msg.MessageId, stream);
        }
    }
}
```

---

## Hands-On: OrderFlow Redis Integration

### Task 1 — Implement Product Cache

Build caching for products with:
- Cache-aside pattern
- Stampede protection
- Invalidation on updates

### Task 2 — Shopping Cart in Redis

Implement cart storage using Redis hashes with:
- TTL for abandoned carts
- Atomic add/remove operations
- Cart merging when user logs in

### Task 3 — Rate Limiting Middleware

Create rate limiting for the API with:
- Per-user limits
- Sliding window algorithm
- Response headers (X-RateLimit-Remaining)

### Task 4 — Distributed Lock for Checkout

Implement checkout locking to prevent:
- Double orders
- Race conditions on inventory

### Task 5 — Cache Metrics

Add observability:
- Hit/miss ratio
- Latency percentiles
- Memory usage alerts

---

## Deliverables

1. **Product caching** with stampede protection
2. **Shopping cart** using Redis data structures
3. **Rate limiting** middleware with sliding window
4. **Distributed locks** for checkout process
5. **Metrics dashboard** (Grafana/Application Insights)

---

## Resources

### Must-Read
- [StackExchange.Redis Documentation](https://stackexchange.github.io/StackExchange.Redis/)
- [Redis Data Types](https://redis.io/docs/data-types/)
- [Redis Best Practices](https://redis.io/docs/management/optimization/)

### Videos
- [Redis University: RU101 Introduction](https://university.redis.com/courses/ru101/)
- [Nick Chapsas: Redis in .NET](https://www.youtube.com/watch?v=5FR4XR7uNBI)
- [Hussein Nasser: Redis Explained](https://www.youtube.com/watch?v=G1rOthIU-uo)

### Tools
- [RedisInsight](https://redis.com/redis-enterprise/redis-insight/) — Visual debugging
- [redis-cli](https://redis.io/docs/ui/cli/) — Command-line interface
- [Testcontainers.Redis](https://www.nuget.org/packages/Testcontainers.Redis)

### Libraries
- [StackExchange.Redis](https://www.nuget.org/packages/StackExchange.Redis)
- [Microsoft.Extensions.Caching.StackExchangeRedis](https://www.nuget.org/packages/Microsoft.Extensions.Caching.StackExchangeRedis)
- [Polly](https://www.nuget.org/packages/Polly) — For retry policies

---

## Reflection Questions

1. When should you cache at the application level vs Redis vs both?
2. How do you handle Redis being unavailable?
3. What's the difference between Pub/Sub and Streams?
4. How do you prevent distributed lock deadlocks?

---

## Module Summary

You've learned the complete NoSQL toolkit:

1. **Foundations** — When SQL vs NoSQL, CAP trade-offs
2. **MongoDB** — Document modeling, indexes, aggregations
3. **Redis** — Caching, locks, rate limiting, streaming

**Key Takeaways:**
- Choose the right database for each use case
- Document databases need different modeling than SQL
- Cache invalidation is harder than caching
- Distributed locks require careful implementation
- Always have fallback strategies when Redis is down

---

**Next Module:** [06 — Messaging and Async Patterns](../06-messaging/README.md)
