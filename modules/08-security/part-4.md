# 08 — Security, Authentication & Identity

## Part 4 / 4 — API Security Operations

---

## Why This Matters

Security isn't just code — it's operations:
- Rate limiting protects against abuse
- Secrets management prevents leaks
- Monitoring detects attacks
- Incident response limits damage

A secure API with exposed secrets or no rate limiting will still be compromised.

---

## Core Concept #18 — Rate Limiting

### Why This Matters

Without rate limiting:
- Brute force attacks succeed
- DDoS brings down your API
- Scrapers steal your data
- Costs spiral from abuse

### What Goes Wrong Without This

**War story — The Credential Stuffing:**
Login endpoint had no rate limiting. Attacker tried 10 million username/password combinations overnight. 50,000 accounts compromised.

**War story — The Scraping Attack:**
Public API had no rate limiting. Competitor scraped entire product catalog in 2 hours. Pricing data leaked to competitors.

### ASP.NET Core Rate Limiting

```csharp
// Program.cs — Configure rate limiting
builder.Services.AddRateLimiter(options =>
{
    // Global rate limit
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        var userId = context.User?.Identity?.Name;

        if (userId is not null)
        {
            // Authenticated users: higher limits per user
            return RateLimitPartition.GetTokenBucketLimiter(userId, _ =>
                new TokenBucketRateLimiterOptions
                {
                    TokenLimit = 100,
                    TokensPerPeriod = 50,
                    ReplenishmentPeriod = TimeSpan.FromSeconds(10),
                    QueueLimit = 10,
                    QueueProcessingOrder = QueueProcessingOrder.OldestFirst
                });
        }

        // Anonymous users: stricter limits per IP
        var ip = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return RateLimitPartition.GetFixedWindowLimiter(ip, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 20,
                Window = TimeSpan.FromMinutes(1)
            });
    });

    // Custom response for rate limited requests
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.ContentType = "application/problem+json";

        var retryAfter = context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retry)
            ? retry.TotalSeconds
            : 60;

        context.HttpContext.Response.Headers.RetryAfter = retryAfter.ToString();

        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 429,
            Title = "Too Many Requests",
            Detail = $"Rate limit exceeded. Try again in {retryAfter} seconds.",
            Extensions = { ["retryAfter"] = retryAfter }
        }, cancellationToken);

        // Log for monitoring
        var logger = context.HttpContext.RequestServices
            .GetRequiredService<ILogger<Program>>();
        logger.LogWarning(
            "Rate limit exceeded for {Identity}",
            context.HttpContext.User?.Identity?.Name
                ?? context.HttpContext.Connection.RemoteIpAddress?.ToString());
    };
});

// Named policies for different endpoints
builder.Services.AddRateLimiter(options =>
{
    // Strict limit for authentication
    options.AddFixedWindowLimiter("auth", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(1);
    });

    // Relaxed limit for read operations
    options.AddSlidingWindowLimiter("read", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6;
    });

    // Moderate limit for writes
    options.AddTokenBucketLimiter("write", opt =>
    {
        opt.TokenLimit = 20;
        opt.TokensPerPeriod = 10;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(30);
    });

    // Very strict for expensive operations
    options.AddConcurrencyLimiter("export", opt =>
    {
        opt.PermitLimit = 2;  // Max 2 concurrent exports per user
        opt.QueueLimit = 5;
    });
});

var app = builder.Build();
app.UseRateLimiter();

// Apply to endpoints
app.MapPost("/auth/login", Login)
    .RequireRateLimiting("auth");

app.MapGet("/orders", GetOrders)
    .RequireRateLimiting("read");

app.MapPost("/orders", CreateOrder)
    .RequireRateLimiting("write");

app.MapGet("/orders/export", ExportOrders)
    .RequireRateLimiting("export");
```

### Distributed Rate Limiting with Redis

```csharp
// For multiple API instances, use Redis
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

// Custom distributed rate limiter
public class RedisRateLimiter
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisRateLimiter> _logger;

    public async Task<RateLimitResult> CheckRateLimitAsync(
        string key,
        int limit,
        TimeSpan window)
    {
        var db = _redis.GetDatabase();
        var now = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        var windowStart = now - (long)window.TotalSeconds;

        var redisKey = $"ratelimit:{key}";

        // Use sorted set with timestamps as scores
        var transaction = db.CreateTransaction();

        // Remove old entries
        _ = transaction.SortedSetRemoveRangeByScoreAsync(
            redisKey, 0, windowStart);

        // Count current entries
        var countTask = transaction.SortedSetLengthAsync(redisKey);

        // Add current request
        _ = transaction.SortedSetAddAsync(redisKey, now.ToString(), now);

        // Set expiry
        _ = transaction.KeyExpireAsync(redisKey, window);

        await transaction.ExecuteAsync();

        var count = await countTask;

        if (count >= limit)
        {
            _logger.LogWarning("Rate limit exceeded for key {Key}: {Count}/{Limit}",
                key, count, limit);

            return new RateLimitResult
            {
                IsAllowed = false,
                CurrentCount = (int)count,
                Limit = limit,
                RetryAfter = window
            };
        }

        return new RateLimitResult
        {
            IsAllowed = true,
            CurrentCount = (int)count + 1,
            Limit = limit,
            Remaining = limit - (int)count - 1
        };
    }
}
```

---

## Core Concept #19 — Secrets Management

### Why This Matters

Secrets in code = secrets exposed:
- Git history is forever
- Logs capture environment variables
- Error messages leak connection strings
- Developers copy secrets to personal machines

### What Goes Wrong Without This

**War story — The GitHub Leak:**
Developer committed AWS credentials to public repo. Bot scraped them in 30 seconds. $50,000 in cryptocurrency mining charges before anyone noticed.

**War story — The Log Exposure:**
Connection string logged in error message. Log aggregator indexed it. Security researcher found it in Shodan. Database dumped.

### Azure Key Vault Integration

```csharp
// Program.cs — Load secrets from Key Vault
var builder = WebApplication.CreateBuilder(args);

// Add Key Vault in production
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUrl = builder.Configuration["KeyVault:Url"];
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUrl!),
        new DefaultAzureCredential());
}

// Access secrets through configuration
var connectionString = builder.Configuration["Database:ConnectionString"];
var apiKey = builder.Configuration["ExternalApi:ApiKey"];

// For local development, use user secrets
// dotnet user-secrets set "Database:ConnectionString" "..."
```

### AWS Secrets Manager Integration

```csharp
// Program.cs — AWS Secrets Manager
builder.Configuration.AddSecretsManager(configurator: options =>
{
    options.SecretFilter = entry => entry.Name.StartsWith("orderflow/");
    options.KeyGenerator = (entry, key) => key.Replace("orderflow/", "").Replace("/", ":");
});

// Or load specific secrets
public class SecretsService
{
    private readonly IAmazonSecretsManager _secretsManager;

    public async Task<string> GetSecretAsync(string secretName)
    {
        var request = new GetSecretValueRequest
        {
            SecretId = secretName,
            VersionStage = "AWSCURRENT"
        };

        var response = await _secretsManager.GetSecretValueAsync(request);
        return response.SecretString;
    }
}
```

### Secret Rotation

```csharp
// Automatic secret rotation handler
public class SecretRotationService : BackgroundService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<SecretRotationService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Check for updated secrets
                await RefreshSecretsAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error refreshing secrets");
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }

    private async Task RefreshSecretsAsync(CancellationToken cancellationToken)
    {
        // Reload configuration
        if (_configuration is IConfigurationRoot configRoot)
        {
            configRoot.Reload();
            _logger.LogInformation("Configuration refreshed");
        }
    }
}

// Connection string rotation without downtime
public class RotatingConnectionFactory
{
    private readonly IConfiguration _configuration;
    private string _currentConnectionString;
    private DateTime _lastRefresh;

    public string GetConnectionString()
    {
        // Refresh every 5 minutes
        if (DateTime.UtcNow - _lastRefresh > TimeSpan.FromMinutes(5))
        {
            _currentConnectionString = _configuration["Database:ConnectionString"]!;
            _lastRefresh = DateTime.UtcNow;
        }

        return _currentConnectionString;
    }
}
```

### Secrets in CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get secrets from Key Vault
        uses: azure/get-keyvault-secrets@v1
        with:
          keyvault: "orderflow-prod-kv"
          secrets: 'DatabaseConnectionString, ApiKey, JwtSecret'
        id: keyvault

      - name: Deploy
        run: |
          # Secrets available as environment variables
          # ${{ steps.keyvault.outputs.DatabaseConnectionString }}
          az webapp config appsettings set \
            --name orderflow-api \
            --settings "ConnectionStrings__Database=${{ steps.keyvault.outputs.DatabaseConnectionString }}"
```

---

## Core Concept #20 — Security Monitoring

### Why This Matters

You can't protect against what you can't see:
- Failed login attempts indicate attacks
- Unusual traffic patterns signal abuse
- Authorization failures reveal probing
- Error spikes suggest exploitation

### Security Event Logging

```csharp
// Security event types
public enum SecurityEventType
{
    AuthenticationSuccess,
    AuthenticationFailure,
    AuthorizationFailure,
    RateLimitExceeded,
    SuspiciousActivity,
    TokenRefresh,
    PasswordChange,
    MfaEnabled,
    MfaDisabled,
    AccountLockout,
    SessionInvalidated
}

public class SecurityEventLogger
{
    private readonly ILogger<SecurityEventLogger> _logger;
    private readonly IMetrics _metrics;

    public void Log(SecurityEvent evt)
    {
        // Structured logging for SIEM
        _logger.LogInformation(
            "SecurityEvent: {EventType} User={UserId} IP={IpAddress} " +
            "Resource={Resource} Success={Success} Details={Details}",
            evt.EventType,
            evt.UserId,
            evt.IpAddress,
            evt.Resource,
            evt.Success,
            evt.Details);

        // Metrics for dashboards
        _metrics.Increment($"security.events.{evt.EventType.ToString().ToLower()}",
            tags: new Dictionary<string, string>
            {
                ["success"] = evt.Success.ToString(),
                ["event_type"] = evt.EventType.ToString()
            });
    }
}

public class SecurityEvent
{
    public SecurityEventType EventType { get; init; }
    public string? UserId { get; init; }
    public string? IpAddress { get; init; }
    public string? Resource { get; init; }
    public bool Success { get; init; }
    public string? Details { get; init; }
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
}
```

### Alerting Rules

```yaml
# prometheus-rules.yml
groups:
  - name: security-alerts
    rules:
      # Brute force detection
      - alert: HighAuthenticationFailureRate
        expr: |
          sum(rate(security_events_total{event_type="AuthenticationFailure"}[5m])) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High authentication failure rate detected"
          description: "More than 10 auth failures per second for 2 minutes"

      # Rate limit abuse
      - alert: RateLimitExceeded
        expr: |
          sum(rate(security_events_total{event_type="RateLimitExceeded"}[5m])) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High rate of rate-limit violations"

      # Suspicious authorization failures
      - alert: AuthorizationFailureSpike
        expr: |
          sum(rate(security_events_total{event_type="AuthorizationFailure"}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Authorization failures exceed 10% of requests"

      # Account lockouts
      - alert: AccountLockoutSpike
        expr: |
          sum(increase(security_events_total{event_type="AccountLockout"}[1h])) > 50
        labels:
          severity: critical
        annotations:
          summary: "High number of account lockouts"
```

### Security Dashboard

```csharp
// Endpoint for security metrics dashboard
app.MapGet("/admin/security/metrics", async (
    OrderFlowDbContext db,
    TimeProvider time) =>
{
    var now = time.GetUtcNow();
    var hourAgo = now.AddHours(-1);
    var dayAgo = now.AddDays(-1);

    var metrics = new SecurityMetrics
    {
        LastHour = new SecurityPeriodMetrics
        {
            AuthenticationSuccesses = await db.SecurityEvents
                .CountAsync(e => e.EventType == SecurityEventType.AuthenticationSuccess
                    && e.Timestamp > hourAgo),
            AuthenticationFailures = await db.SecurityEvents
                .CountAsync(e => e.EventType == SecurityEventType.AuthenticationFailure
                    && e.Timestamp > hourAgo),
            AuthorizationFailures = await db.SecurityEvents
                .CountAsync(e => e.EventType == SecurityEventType.AuthorizationFailure
                    && e.Timestamp > hourAgo),
            RateLimitViolations = await db.SecurityEvents
                .CountAsync(e => e.EventType == SecurityEventType.RateLimitExceeded
                    && e.Timestamp > hourAgo),
            UniqueIPs = await db.SecurityEvents
                .Where(e => e.Timestamp > hourAgo)
                .Select(e => e.IpAddress)
                .Distinct()
                .CountAsync()
        },
        TopFailedIPs = await db.SecurityEvents
            .Where(e => e.EventType == SecurityEventType.AuthenticationFailure
                && e.Timestamp > dayAgo)
            .GroupBy(e => e.IpAddress)
            .OrderByDescending(g => g.Count())
            .Take(10)
            .Select(g => new { IP = g.Key, Count = g.Count() })
            .ToListAsync(),
        RecentLockouts = await db.SecurityEvents
            .Where(e => e.EventType == SecurityEventType.AccountLockout
                && e.Timestamp > dayAgo)
            .OrderByDescending(e => e.Timestamp)
            .Take(20)
            .ToListAsync()
    };

    return Results.Ok(metrics);
}).RequireAuthorization("Admin");
```

---

## Core Concept #21 — Incident Response

### Why This Matters

Security incidents happen. The response determines the damage:
- Fast detection limits exposure
- Clear procedures prevent panic
- Communication maintains trust
- Post-mortems prevent recurrence

### Incident Response Runbook

```markdown
# Security Incident Response Runbook

## Severity Levels

### Critical (P1)
- Active data breach
- Compromised credentials in use
- System under active attack
- Response: Immediate, all hands

### High (P2)
- Suspected breach
- Exposed credentials discovered
- Unusual access patterns
- Response: Within 1 hour

### Medium (P3)
- Failed attack attempts
- Policy violations
- Suspicious activity
- Response: Within 4 hours

## Response Procedures

### 1. Credential Leak Detected

**Indicators:**
- Secret found in git history
- Credential in logs
- Alert from secret scanning

**Steps:**
1. IMMEDIATELY rotate the exposed credential
2. Check for unauthorized access in logs
3. Identify scope of exposure
4. Notify affected parties
5. Update secret storage
6. Post-mortem within 24 hours

**Commands:**
```bash
# Rotate database password
az keyvault secret set --vault-name orderflow-kv \
  --name DatabasePassword --value "$(openssl rand -base64 32)"

# Force restart to pick up new secret
az webapp restart --name orderflow-api

# Check for unauthorized access
az monitor activity-log list --start-time 2024-01-01 \
  --query "[?authorization.action=='Microsoft.Sql/servers/databases/read']"
```

### 2. Brute Force Attack Detected

**Indicators:**
- Alert: HighAuthenticationFailureRate
- Multiple failed logins from single IP
- Account lockout spike

**Steps:**
1. Identify attacking IPs
2. Block at WAF/firewall level
3. Check for compromised accounts
4. Force password reset if needed
5. Review rate limiting configuration

**Commands:**
```bash
# Block IP at Azure WAF
az network application-gateway waf-policy custom-rule create \
  --policy-name orderflow-waf-policy \
  --name BlockAttackerIP \
  --priority 1 \
  --rule-type MatchRule \
  --action Block \
  --match-conditions "RemoteAddr" "IPMatch" "1.2.3.4"

# Query failed logins
SELECT ip_address, COUNT(*) as attempts
FROM security_events
WHERE event_type = 'AuthenticationFailure'
  AND timestamp > NOW() - INTERVAL '1 hour'
GROUP BY ip_address
ORDER BY attempts DESC;
```

### 3. Unauthorized Data Access

**Indicators:**
- Authorization failures for sensitive resources
- Cross-tenant access attempts
- Unusual query patterns

**Steps:**
1. Identify affected user/session
2. Invalidate all sessions for user
3. Review access logs for scope
4. Notify data protection officer if PII involved
5. Preserve evidence for investigation

**Commands:**
```csharp
// Invalidate all sessions for user
await _sessionService.InvalidateAllSessionsAsync(userId);

// Block user temporarily
await _userManager.SetLockoutEnabledAsync(user, true);
await _userManager.SetLockoutEndDateAsync(user, DateTimeOffset.MaxValue);
```
```

### Automated Incident Response

```csharp
// Automatic response to security events
public class SecurityIncidentResponder : BackgroundService
{
    private readonly IServiceProvider _services;
    private readonly ILogger<SecurityIncidentResponder> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CheckForIncidentsAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task CheckForIncidentsAsync(CancellationToken cancellationToken)
    {
        using var scope = _services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();
        var alertService = scope.ServiceProvider.GetRequiredService<IAlertService>();

        var recentFailures = await db.SecurityEvents
            .Where(e => e.EventType == SecurityEventType.AuthenticationFailure)
            .Where(e => e.Timestamp > DateTime.UtcNow.AddMinutes(-5))
            .GroupBy(e => e.IpAddress)
            .Where(g => g.Count() > 10)
            .Select(g => new { IP = g.Key, Count = g.Count() })
            .ToListAsync(cancellationToken);

        foreach (var failure in recentFailures)
        {
            _logger.LogWarning(
                "Potential brute force from {IP}: {Count} failures in 5 minutes",
                failure.IP, failure.Count);

            // Auto-block if threshold exceeded
            if (failure.Count > 50)
            {
                await BlockIpAsync(failure.IP!);
                await alertService.SendAlertAsync(new Alert
                {
                    Severity = AlertSeverity.Critical,
                    Title = "IP Auto-Blocked",
                    Message = $"IP {failure.IP} blocked after {failure.Count} auth failures"
                });
            }
        }
    }
}
```

---

## Core Concept #22 — Security Testing

### Why This Matters

Regular security testing finds vulnerabilities before attackers do:
- Penetration testing simulates real attacks
- Vulnerability scanning finds known issues
- Security audits review architecture
- Bug bounties leverage external expertise

### OWASP ZAP Integration

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Monday
  workflow_dispatch:

jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: |
          docker-compose up -d
          sleep 30  # Wait for startup

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: 'http://localhost:5000'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'

      - name: ZAP Full Scan
        uses: zaproxy/action-full-scan@v0.10.0
        with:
          target: 'http://localhost:5000'
          rules_file_name: '.zap/rules.tsv'

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: report_html.html
```

### Dependency Vulnerability Scanning

```yaml
# .github/workflows/dependency-scan.yml
name: Dependency Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'  # Daily

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Check for vulnerabilities
        run: |
          dotnet list package --vulnerable --include-transitive \
            --format json > vulnerabilities.json

          # Fail if high severity vulnerabilities found
          if jq -e '.projects[].frameworks[].topLevelPackages[]
            .vulnerabilities[]? | select(.severity == "High" or .severity == "Critical")' \
            vulnerabilities.json; then
            echo "::error::High/Critical vulnerabilities found!"
            exit 1
          fi

      - name: Upload vulnerability report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: vulnerability-report
          path: vulnerabilities.json
```

---

## Hands-On: OrderFlow Security Operations

### Task 1 — Rate Limiting

Implement rate limiting for:
- Authentication endpoints (strict)
- Read endpoints (relaxed)
- Write endpoints (moderate)
- Export endpoints (concurrent limit)

### Task 2 — Secrets Management

Set up:
- Azure Key Vault or AWS Secrets Manager
- Local development with user secrets
- Secret rotation procedure
- CI/CD secret injection

### Task 3 — Security Monitoring

Build:
- Security event logging
- Prometheus metrics
- Alerting rules
- Security dashboard

### Task 4 — Incident Response

Create:
- Incident response runbook
- Automated response for brute force
- Alert escalation procedure
- Post-mortem template

---

## Deliverables

1. **Rate limiting** configuration with multiple policies
2. **Secrets management** with Key Vault integration
3. **Security monitoring** dashboard and alerts
4. **Incident response** runbook and automation
5. **Security scanning** in CI/CD pipeline

---

## Resources

### Must-Read
- [ASP.NET Core Rate Limiting](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)

### Videos
- [Nick Chapsas: Rate Limiting in .NET](https://www.youtube.com/watch?v=GQAgh_z1rHo)
- [Azure Key Vault Best Practices](https://www.youtube.com/watch?v=w8L8tLT8WnI)
- [OWASP ZAP Tutorial](https://www.youtube.com/watch?v=CxS3Z4dS05o)

### Tools
- [OWASP ZAP](https://www.zaproxy.org/) — Security scanner
- [Burp Suite](https://portswigger.net/burp) — Penetration testing
- [Snyk](https://snyk.io/) — Dependency scanning
- [Grafana](https://grafana.com/) — Security dashboards

---

## Reflection Questions

1. How do you balance rate limiting with user experience?
2. What's your secret rotation strategy?
3. How quickly can you respond to a security incident?
4. What security metrics do you track?

---

## Module Summary

You've learned:
1. **Security Foundations** — OWASP Top 10, threat modeling, secure coding
2. **Authentication** — JWT, OAuth2/OIDC, MFA, API keys
3. **Authorization** — Policies, handlers, resource-based, multi-tenant
4. **Operations** — Rate limiting, secrets, monitoring, incident response

Security is not a feature — it's a continuous practice. Build it into every decision.

---

**Module Complete!** Continue to [Module 09 — Performance](../09-performance/README.md)
