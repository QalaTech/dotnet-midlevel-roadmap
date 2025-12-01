# 08 — Security, Authentication & Identity

## Part 1 / 4 — Security Foundations

---

## Why This Matters

Security isn't a feature — it's a quality of all features. Every endpoint, every input, every database query is an attack surface.

The OWASP Top 10 hasn't changed much in years because the same mistakes keep happening:
1. Injection (SQL, command, LDAP)
2. Broken authentication
3. Sensitive data exposure
4. XML external entities (XXE)
5. Broken access control
6. Security misconfiguration
7. Cross-site scripting (XSS)
8. Insecure deserialization
9. Using components with known vulnerabilities
10. Insufficient logging and monitoring

---

## Core Concept #1 — OWASP Top 10 for .NET

### Why This Matters

Knowing the Top 10 isn't enough. You need to know how each manifests in .NET and how to prevent it.

### What Goes Wrong Without This

**War story — The SQL Injection:**
```csharp
// ❌ Vulnerable code
var query = $"SELECT * FROM Users WHERE Email = '{email}'";
```
Attacker enters: `' OR '1'='1' --`
Result: Returns all users. Attacker gains admin access.

**War story — The Exposed Secrets:**
Team committed `appsettings.Development.json` with production connection strings. Attacker found it in GitHub search. Database dumped overnight.

### OWASP in .NET — Prevention

```csharp
// 1. INJECTION — Use parameterized queries
// ❌ VULNERABLE
var sql = $"SELECT * FROM Products WHERE Name LIKE '%{searchTerm}%'";

// ✅ SAFE — Parameterized with EF Core
var products = await _context.Products
    .Where(p => EF.Functions.Like(p.Name, $"%{searchTerm}%"))
    .ToListAsync();

// ✅ SAFE — Parameterized with Dapper
var products = await connection.QueryAsync<Product>(
    "SELECT * FROM Products WHERE Name LIKE @Term",
    new { Term = $"%{searchTerm}%" });
```

```csharp
// 2. BROKEN AUTHENTICATION — Secure password handling
// ❌ VULNERABLE — Plain text or weak hashing
var passwordHash = MD5.HashData(Encoding.UTF8.GetBytes(password));

// ✅ SAFE — Use Identity or proper hashing
services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Password.RequiredLength = 12;
    options.Password.RequireDigit = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
});
```

```csharp
// 3. SENSITIVE DATA EXPOSURE — Encrypt in transit and at rest
// ❌ VULNERABLE — Logging sensitive data
_logger.LogInformation("User {Email} logged in with password {Password}", email, password);

// ✅ SAFE — Never log sensitive data
_logger.LogInformation("User {Email} logged in", email);

// Configure HTTPS
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443;
});

// HSTS
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});
```

```csharp
// 4. BROKEN ACCESS CONTROL — Always verify ownership
// ❌ VULNERABLE — No ownership check
app.MapGet("/orders/{id}", async (Guid id, OrderFlowDbContext db) =>
{
    var order = await db.Orders.FindAsync(id);
    return order;  // Any user can access any order!
});

// ✅ SAFE — Verify ownership
app.MapGet("/orders/{id}", async (
    Guid id,
    ClaimsPrincipal user,
    OrderFlowDbContext db) =>
{
    var userId = user.FindFirstValue(ClaimTypes.NameIdentifier);
    var order = await db.Orders
        .Where(o => o.Id == id && o.CustomerId == Guid.Parse(userId!))
        .FirstOrDefaultAsync();

    return order is null ? Results.NotFound() : Results.Ok(order);
}).RequireAuthorization();
```

```csharp
// 5. SECURITY MISCONFIGURATION — Secure defaults
// ❌ VULNERABLE — Detailed errors in production
app.UseDeveloperExceptionPage();  // NEVER in production

// ✅ SAFE — Environment-appropriate errors
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

// Disable unnecessary features
builder.Services.AddControllers(options =>
{
    options.SuppressAsyncSuffixInActionNames = true;
})
.AddJsonOptions(options =>
{
    // Don't expose internal property names
    options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
});
```

```csharp
// 6. XSS — Encode output
// ❌ VULNERABLE — Raw HTML output
return Content($"<div>Welcome, {userName}</div>", "text/html");

// ✅ SAFE — HTML encode
return Content($"<div>Welcome, {HtmlEncoder.Default.Encode(userName)}</div>", "text/html");

// Or use Razor which auto-encodes
// @Model.UserName is automatically encoded

// Content Security Policy
app.Use(async (context, next) =>
{
    context.Response.Headers.Append(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
    await next();
});
```

```csharp
// 7. INSECURE DESERIALIZATION — Be careful with untrusted data
// ❌ VULNERABLE — Deserializing untrusted types
var settings = new JsonSerializerSettings { TypeNameHandling = TypeNameHandling.All };
var obj = JsonConvert.DeserializeObject(untrustedJson, settings);

// ✅ SAFE — Explicit types, no type handling
var obj = JsonSerializer.Deserialize<MyKnownType>(json);

// Or if you must handle polymorphism
var options = new JsonSerializerOptions
{
    TypeInfoResolver = new DefaultJsonTypeInfoResolver()
};
```

```csharp
// 8. VULNERABLE COMPONENTS — Keep dependencies updated
// Check for vulnerabilities regularly
// dotnet list package --vulnerable

// Use Dependabot or similar
// .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

```csharp
// 9. INSUFFICIENT LOGGING — Log security events
public class SecurityAuditLogger
{
    private readonly ILogger<SecurityAuditLogger> _logger;

    public void LogAuthenticationSuccess(string userId, string ipAddress)
    {
        _logger.LogInformation(
            "Authentication successful. UserId: {UserId}, IP: {IP}, Time: {Time}",
            userId, ipAddress, DateTime.UtcNow);
    }

    public void LogAuthenticationFailure(string attemptedUser, string ipAddress, string reason)
    {
        _logger.LogWarning(
            "Authentication failed. AttemptedUser: {User}, IP: {IP}, Reason: {Reason}",
            attemptedUser, ipAddress, reason);
    }

    public void LogAuthorizationFailure(string userId, string resource, string action)
    {
        _logger.LogWarning(
            "Authorization denied. UserId: {UserId}, Resource: {Resource}, Action: {Action}",
            userId, resource, action);
    }
}
```

---

## Core Concept #2 — Threat Modeling with STRIDE

### Why This Matters

You can't protect against threats you haven't identified. Threat modeling makes you think like an attacker before building.

### STRIDE Categories

| Threat | Property Violated | Example |
|--------|------------------|---------|
| **S**poofing | Authentication | Attacker impersonates another user |
| **T**ampering | Integrity | Attacker modifies order total |
| **R**epudiation | Non-repudiation | User denies placing order |
| **I**nformation Disclosure | Confidentiality | Customer data leaked |
| **D**enial of Service | Availability | API flooded with requests |
| **E**levation of Privilege | Authorization | User gains admin access |

### Threat Model Example — Order Placement

```
Data Flow Diagram:

[Browser] ---(1)---> [API Gateway] ---(2)---> [Order Service]
                                                    |
                                              (3)   v
                                            [Database]
                                                    |
                                              (4)   v
                                            [Payment Service]
                                                    |
                                              (5)   v
                                            [Notification Service]
```

```markdown
## Threat Model: Order Placement

### Assets
- Customer PII (name, address, email)
- Payment information (card tokens)
- Order data (items, totals)
- Authentication tokens

### Trust Boundaries
1. Browser → API Gateway (untrusted to trusted)
2. API Gateway → Internal Services (trusted)
3. Services → Database (trusted)
4. Services → External Payment Provider (trusted to external)

### STRIDE Analysis

#### (1) Browser to API Gateway

| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | User impersonation | JWT validation, MFA for sensitive ops |
| Tampering | Modified request | HTTPS, request signing |
| Repudiation | Deny order placement | Audit logging with correlation ID |
| Info Disclosure | Token theft | Secure cookies, short token lifetime |
| DoS | Request flooding | Rate limiting, CAPTCHA |
| Elevation | Access other users' data | Authorization checks |

#### (2) API Gateway to Order Service

| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | Rogue service | mTLS, service mesh |
| Tampering | Modified messages | Message signing |

#### (4) Order Service to Payment Service

| Threat | Risk | Mitigation |
|--------|------|------------|
| Info Disclosure | Card data exposure | Never store card data, use tokens |
| Repudiation | Disputed charges | Payment receipts, audit log |
```

### Documenting Mitigations

```csharp
// Mitigation: Rate limiting for DoS protection
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User?.Identity?.Name
                ?? context.Connection.RemoteIpAddress?.ToString()
                ?? "anonymous",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            });
    });
});

// Mitigation: Audit logging for repudiation
public class PlaceOrderHandler
{
    private readonly IAuditLogger _auditLogger;

    public async Task<Result<Guid>> Handle(PlaceOrderCommand command)
    {
        var orderId = await CreateOrderAsync(command);

        await _auditLogger.LogAsync(new AuditEvent
        {
            EventType = "OrderPlaced",
            UserId = command.UserId,
            ResourceId = orderId.ToString(),
            Timestamp = DateTime.UtcNow,
            CorrelationId = command.CorrelationId,
            Details = new { ItemCount = command.Items.Count, Total = command.Total }
        });

        return Result.Success(orderId);
    }
}
```

---

## Core Concept #3 — Secure Coding Practices

### Why This Matters

Security bugs are coding bugs. Secure coding practices prevent vulnerabilities at the source.

### Input Validation

```csharp
// Always validate on the server, never trust client validation
public class CreateOrderRequest
{
    [Required]
    public Guid CustomerId { get; init; }

    [Required]
    [MinLength(1)]
    [MaxLength(100)]
    public List<OrderItemRequest> Items { get; init; } = new();
}

public class OrderItemRequest
{
    [Required]
    public Guid ProductId { get; init; }

    [Range(1, 1000)]
    public int Quantity { get; init; }
}

// Use FluentValidation for complex rules
public class CreateOrderValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty()
            .WithMessage("Customer ID is required");

        RuleFor(x => x.Items)
            .NotEmpty()
            .WithMessage("Order must have at least one item");

        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.Quantity)
                .GreaterThan(0)
                .LessThanOrEqualTo(1000)
                .WithMessage("Quantity must be between 1 and 1000");
        });
    }
}
```

### Output Encoding

```csharp
// Encode based on context
public static class SecurityEncoders
{
    // For HTML content
    public static string HtmlEncode(string input) =>
        HtmlEncoder.Default.Encode(input);

    // For JavaScript strings
    public static string JsEncode(string input) =>
        JavaScriptEncoder.Default.Encode(input);

    // For URLs
    public static string UrlEncode(string input) =>
        UrlEncoder.Default.Encode(input);
}

// In API responses, JSON serialization handles encoding
// But be careful with raw content
app.MapGet("/user-content/{id}", async (Guid id, UserContentService service) =>
{
    var content = await service.GetAsync(id);

    // ❌ WRONG — Raw HTML
    return Results.Content(content.Html, "text/html");

    // ✅ SAFE — Return as JSON (auto-encoded)
    return Results.Ok(new { Content = content.Html });

    // ✅ SAFE — Sanitize if you must return HTML
    return Results.Content(
        HtmlSanitizer.Sanitize(content.Html),
        "text/html");
});
```

### Secure Error Handling

```csharp
// Never expose internal details
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    private readonly IWebHostEnvironment _env;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var correlationId = Activity.Current?.Id ?? context.TraceIdentifier;

        // Log full details internally
        _logger.LogError(exception,
            "Unhandled exception. CorrelationId: {CorrelationId}",
            correlationId);

        // Return safe response
        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An error occurred",
            Detail = _env.IsDevelopment()
                ? exception.Message  // Only in dev
                : "Please contact support with this ID",
            Extensions = { ["correlationId"] = correlationId }
        };

        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

---

## Core Concept #4 — Static Analysis and Security Scanning

### Why This Matters

Automated tools catch issues humans miss. They run on every commit, not just during code review.

### Security Analyzers

```xml
<!-- Enable security analyzers in .csproj -->
<PropertyGroup>
    <AnalysisLevel>latest</AnalysisLevel>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisMode>AllEnabledByDefault</AnalysisMode>
</PropertyGroup>

<ItemGroup>
    <!-- Security-specific analyzers -->
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="8.0.0" />
    <PackageReference Include="SecurityCodeScan.VS2019" Version="5.6.7" />
    <PackageReference Include="Roslynator.Analyzers" Version="4.12.0" />
</ItemGroup>

<!-- .editorconfig for security rules -->
[*.cs]
# Security rules
dotnet_diagnostic.CA2100.severity = error  # SQL injection
dotnet_diagnostic.CA2300.severity = error  # Insecure deserializer
dotnet_diagnostic.CA3001.severity = error  # SQL injection
dotnet_diagnostic.CA3002.severity = error  # XSS
dotnet_diagnostic.CA3003.severity = error  # Path traversal
dotnet_diagnostic.CA3004.severity = error  # Information disclosure
dotnet_diagnostic.CA3005.severity = error  # LDAP injection
dotnet_diagnostic.CA3006.severity = error  # Process command injection
dotnet_diagnostic.CA3007.severity = error  # Open redirect
dotnet_diagnostic.CA5350.severity = error  # Weak crypto
dotnet_diagnostic.CA5351.severity = error  # Broken crypto
```

### Vulnerability Scanning in CI

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 0 * * *'  # Daily

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Check for vulnerable packages
        run: |
          dotnet list package --vulnerable --include-transitive 2>&1 | tee vulnerability-report.txt
          if grep -q "has the following vulnerable packages" vulnerability-report.txt; then
            echo "::error::Vulnerable packages found!"
            exit 1
          fi

      - name: Upload vulnerability report
        uses: actions/upload-artifact@v4
        with:
          name: vulnerability-report
          path: vulnerability-report.txt

  sast-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Security Code Scan
        run: |
          dotnet tool install --global security-scan
          security-scan --export=sarif --output=security-results.sarif .

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: security-results.sarif

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### DevSkim Integration

```json
// .vscode/settings.json
{
    "devskim.enableManualReviewRules": true,
    "devskim.guidanceBaseURL": "https://github.com/microsoft/DevSkim/wiki/",
    "devskim.rules": {
        "DS126858": "Error",   // Hardcoded password
        "DS104456": "Error",   // Hardcoded connection string
        "DS137138": "Error",   // Hardcoded cryptographic key
        "DS114352": "Warning"  // Weak random number generator
    }
}
```

---

## Core Concept #5 — Secure Defaults

### Why This Matters

The default configuration should be secure. Developers shouldn't have to remember to enable security.

### Security Headers

```csharp
// Program.cs — Configure security middleware
var builder = WebApplication.CreateBuilder(args);

// Force HTTPS
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443;
});

// HSTS
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});

var app = builder.Build();

// Apply security headers
app.UseSecurityHeaders();  // Custom middleware below

app.UseHttpsRedirection();
app.UseHsts();

// Custom security headers middleware
public static class SecurityHeadersMiddleware
{
    public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder app)
    {
        return app.Use(async (context, next) =>
        {
            var headers = context.Response.Headers;

            // Prevent clickjacking
            headers.Append("X-Frame-Options", "DENY");

            // Prevent MIME type sniffing
            headers.Append("X-Content-Type-Options", "nosniff");

            // Enable XSS filter
            headers.Append("X-XSS-Protection", "1; mode=block");

            // Referrer policy
            headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");

            // Content Security Policy
            headers.Append("Content-Security-Policy",
                "default-src 'self'; " +
                "script-src 'self'; " +
                "style-src 'self' 'unsafe-inline'; " +
                "img-src 'self' data: https:; " +
                "font-src 'self'; " +
                "connect-src 'self'; " +
                "frame-ancestors 'none';");

            // Permissions Policy
            headers.Append("Permissions-Policy",
                "accelerometer=(), camera=(), geolocation=(), gyroscope=(), " +
                "magnetometer=(), microphone=(), payment=(), usb=()");

            await next();
        });
    }
}
```

### Secure Cookie Configuration

```csharp
// Cookie authentication with secure defaults
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.Cookie.Name = "OrderFlow.Auth";
        options.Cookie.HttpOnly = true;      // Prevent XSS access
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;  // HTTPS only
        options.Cookie.SameSite = SameSiteMode.Strict;  // CSRF protection
        options.ExpireTimeSpan = TimeSpan.FromHours(1);
        options.SlidingExpiration = true;

        options.Events = new CookieAuthenticationEvents
        {
            OnRedirectToLogin = context =>
            {
                // Return 401 instead of redirect for APIs
                if (context.Request.Path.StartsWithSegments("/api"))
                {
                    context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                    return Task.CompletedTask;
                }
                context.Response.Redirect(context.RedirectUri);
                return Task.CompletedTask;
            }
        };
    });
```

### CORS Configuration

```csharp
// Secure CORS — never use AllowAnyOrigin with credentials
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins(
            "https://app.orderflow.com",
            "https://admin.orderflow.com")
            .AllowCredentials()
            .WithMethods("GET", "POST", "PUT", "DELETE")
            .WithHeaders("Authorization", "Content-Type");
    });

    // ❌ NEVER DO THIS
    // policy.AllowAnyOrigin().AllowCredentials();  // Security vulnerability!
});
```

---

## Hands-On: OrderFlow Security Foundations

### Task 1 — OWASP Checklist

Review OrderFlow for OWASP Top 10:
- SQL injection in any endpoint
- Sensitive data in logs
- Missing authorization checks
- Insecure dependencies

### Task 2 — Threat Model

Create threat model for order placement:
- Draw data flow diagram
- Identify trust boundaries
- Apply STRIDE
- Document mitigations

### Task 3 — Security Analyzers

Configure and run:
- .NET analyzers with security rules
- Vulnerability scanning
- Secret detection

### Task 4 — Security Headers

Implement:
- HSTS
- CSP
- X-Frame-Options
- Secure cookies

---

## Deliverables

1. **OWASP checklist** with findings and fixes
2. **Threat model document** for order placement
3. **CI security pipeline** with SAST and dependency scanning
4. **Security headers** verified with SecurityHeaders.com

---

## Resources

### Must-Read
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Microsoft Security Documentation](https://learn.microsoft.com/en-us/aspnet/core/security/)

### Videos
- [Scott Helme: Security Headers](https://www.youtube.com/watch?v=4J5-5j8QxYA)
- [OWASP: Threat Modeling](https://www.youtube.com/watch?v=GuhIefIGeuA)
- [Nick Chapsas: Security in .NET](https://www.youtube.com/watch?v=JpZsNY7Yc4s)

### Tools
- [SecurityHeaders.com](https://securityheaders.com/) — Header analyzer
- [OWASP ZAP](https://www.zaproxy.org/) — Security scanner
- [GitLeaks](https://github.com/gitleaks/gitleaks) — Secret detection
- [DevSkim](https://github.com/microsoft/DevSkim) — IDE security linting

---

## Reflection Questions

1. What's the difference between authentication and authorization?
2. Why should you never log sensitive data?
3. How does threat modeling help find security issues?
4. Why is "secure by default" important?

---

**Next:** [Part 2 — Authentication](./part-2.md)
