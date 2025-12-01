# 08 — Security, Authentication & Identity

## Part 3 / 4 — Authorization

---

## Why This Matters

Authorization answers: "What are you allowed to do?"

Authentication says who you are. Authorization decides what you can access. Getting authorization wrong means:
- Users access other users' data
- Regular users perform admin actions
- Deleted/suspended users retain access
- Audit trails are meaningless

---

## Core Concept #12 — Policy-Based Authorization

### Why This Matters

ASP.NET Core's policy-based authorization is powerful but often misunderstood. Policies express business rules, not just role checks.

### What Goes Wrong Without This

**War story — The Role Explosion:**
Team used `[Authorize(Roles = "Admin,Manager,Support,...")]` everywhere. 50 roles, hundreds of attributes. Adding a new role meant updating 200 files. Nobody knew what roles could do what.

**War story — The Missing Check:**
Developer forgot `[Authorize]` on new endpoint. Any authenticated user could delete any order. Caught 6 months later in security audit.

### Defining Policies

```csharp
// Program.cs — Policy definitions
builder.Services.AddAuthorization(options =>
{
    // Default policy — all authenticated users
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // Fallback — apply to all endpoints without explicit policy
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // Role-based policies
    options.AddPolicy("Admin", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("Support", policy =>
        policy.RequireRole("Admin", "Support"));

    // Claim-based policies
    options.AddPolicy("VerifiedEmail", policy =>
        policy.RequireClaim("email_verified", "true"));

    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"));

    // Custom requirement policies
    options.AddPolicy("CanManageOrders", policy =>
        policy.AddRequirements(new ManageOrdersRequirement()));

    options.AddPolicy("OrderOwner", policy =>
        policy.AddRequirements(new ResourceOwnerRequirement<Order>()));

    // Combined policies
    options.AddPolicy("AdminOrOrderOwner", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.AddRequirements(new AdminOrOwnerRequirement());
    });

    // Scope-based (for OAuth)
    options.AddPolicy("orders:read", policy =>
        policy.RequireClaim("scope", "orders:read", "orders:*"));

    options.AddPolicy("orders:write", policy =>
        policy.RequireClaim("scope", "orders:write", "orders:*"));
});
```

### Applying Policies

```csharp
// Minimal API endpoints
app.MapGet("/orders", GetOrders)
    .RequireAuthorization("orders:read");

app.MapPost("/orders", CreateOrder)
    .RequireAuthorization("orders:write");

app.MapDelete("/orders/{id}", DeleteOrder)
    .RequireAuthorization("Admin");

// Controller-based
[ApiController]
[Route("api/[controller]")]
[Authorize]  // Default policy for all actions
public class OrdersController : ControllerBase
{
    [HttpGet]
    [Authorize(Policy = "orders:read")]
    public async Task<IActionResult> GetOrders() { }

    [HttpGet("{id}")]
    [Authorize(Policy = "OrderOwner")]
    public async Task<IActionResult> GetOrder(Guid id) { }

    [HttpPost]
    [Authorize(Policy = "orders:write")]
    public async Task<IActionResult> CreateOrder() { }

    [HttpDelete("{id}")]
    [Authorize(Policy = "Admin")]
    public async Task<IActionResult> DeleteOrder(Guid id) { }

    [HttpPost("{id}/refund")]
    [Authorize(Policy = "Support")]
    public async Task<IActionResult> RefundOrder(Guid id) { }

    [AllowAnonymous]  // Override default
    [HttpGet("public-catalog")]
    public async Task<IActionResult> GetPublicCatalog() { }
}
```

---

## Core Concept #13 — Custom Authorization Handlers

### Why This Matters

Simple role checks aren't enough. Real authorization often requires:
- Checking resource ownership
- Querying database for permissions
- Time-based access
- Multi-factor conditions

### Building Custom Handlers

```csharp
// Requirement — what needs to be satisfied
public class ResourceOwnerRequirement : IAuthorizationRequirement
{
}

// Handler — how to evaluate the requirement
public class ResourceOwnerHandler : AuthorizationHandler<ResourceOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ResourceOwnerRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);

        if (userId is null)
        {
            return Task.CompletedTask;  // Not authenticated
        }

        if (resource.CustomerId.ToString() == userId)
        {
            context.Succeed(requirement);
        }

        // Don't call Fail() — let other handlers try
        return Task.CompletedTask;
    }
}

// Register handler
builder.Services.AddSingleton<IAuthorizationHandler, ResourceOwnerHandler>();
```

### Admin-or-Owner Pattern

```csharp
public class AdminOrOwnerRequirement : IAuthorizationRequirement
{
}

public class AdminOrOwnerHandler : AuthorizationHandler<AdminOrOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        AdminOrOwnerRequirement requirement,
        Order resource)
    {
        // Admin can access anything
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Owner can access their own resources
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (resource.CustomerId.ToString() == userId)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

### Handler with Database Access

```csharp
public class TeamMemberRequirement : IAuthorizationRequirement
{
    public string Permission { get; }

    public TeamMemberRequirement(string permission)
    {
        Permission = permission;
    }
}

public class TeamMemberHandler : AuthorizationHandler<TeamMemberRequirement, Project>
{
    private readonly ITeamRepository _teamRepository;
    private readonly ILogger<TeamMemberHandler> _logger;

    public TeamMemberHandler(
        ITeamRepository teamRepository,
        ILogger<TeamMemberHandler> logger)
    {
        _teamRepository = teamRepository;
        _logger = logger;
    }

    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        TeamMemberRequirement requirement,
        Project resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (userId is null) return;

        var membership = await _teamRepository.GetMembershipAsync(
            Guid.Parse(userId),
            resource.TeamId);

        if (membership is null)
        {
            _logger.LogWarning(
                "User {UserId} denied access to project {ProjectId} — not a team member",
                userId, resource.Id);
            return;
        }

        if (membership.Permissions.Contains(requirement.Permission))
        {
            context.Succeed(requirement);
            _logger.LogInformation(
                "User {UserId} granted {Permission} on project {ProjectId}",
                userId, requirement.Permission, resource.Id);
        }
        else
        {
            _logger.LogWarning(
                "User {UserId} denied {Permission} on project {ProjectId} — insufficient permissions",
                userId, requirement.Permission, resource.Id);
        }
    }
}
```

---

## Core Concept #14 — Resource-Based Authorization

### Why This Matters

Most real authorization is resource-based: "Can this user edit THIS order?" not "Can this user edit orders in general?"

### Using IAuthorizationService

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;
    private readonly IOrderRepository _orderRepository;

    public OrdersController(
        IAuthorizationService authorizationService,
        IOrderRepository orderRepository)
    {
        _authorizationService = authorizationService;
        _orderRepository = orderRepository;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(Guid id)
    {
        var order = await _orderRepository.GetByIdAsync(id);
        if (order is null)
            return NotFound();

        var authResult = await _authorizationService.AuthorizeAsync(
            User, order, "OrderOwner");

        if (!authResult.Succeeded)
        {
            return Forbid();
        }

        return Ok(order);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateOrder(Guid id, UpdateOrderRequest request)
    {
        var order = await _orderRepository.GetByIdAsync(id);
        if (order is null)
            return NotFound();

        // Check edit permission
        var authResult = await _authorizationService.AuthorizeAsync(
            User, order, new OperationAuthorizationRequirement { Name = "Edit" });

        if (!authResult.Succeeded)
        {
            return Forbid();
        }

        // Update order...
        return Ok(order);
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteOrder(Guid id)
    {
        var order = await _orderRepository.GetByIdAsync(id);
        if (order is null)
            return NotFound();

        // Only admins or owners can delete
        var authResult = await _authorizationService.AuthorizeAsync(
            User, order, "AdminOrOrderOwner");

        if (!authResult.Succeeded)
        {
            _logger.LogWarning(
                "User {UserId} denied delete on order {OrderId}",
                User.FindFirstValue(ClaimTypes.NameIdentifier),
                id);
            return Forbid();
        }

        await _orderRepository.DeleteAsync(id);
        return NoContent();
    }
}
```

### Operation-Based Requirements

```csharp
public static class Operations
{
    public static OperationAuthorizationRequirement Create =
        new() { Name = nameof(Create) };
    public static OperationAuthorizationRequirement Read =
        new() { Name = nameof(Read) };
    public static OperationAuthorizationRequirement Update =
        new() { Name = nameof(Update) };
    public static OperationAuthorizationRequirement Delete =
        new() { Name = nameof(Delete) };
}

public class OrderOperationsHandler :
    AuthorizationHandler<OperationAuthorizationRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        var isOwner = resource.CustomerId.ToString() == userId;
        var isAdmin = context.User.IsInRole("Admin");
        var isSupport = context.User.IsInRole("Support");

        var permitted = requirement.Name switch
        {
            nameof(Operations.Read) => isOwner || isAdmin || isSupport,
            nameof(Operations.Update) => isOwner || isAdmin,
            nameof(Operations.Delete) => isAdmin,
            nameof(Operations.Create) => true,  // Any authenticated user
            _ => false
        };

        if (permitted)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

---

## Core Concept #15 — Multi-Tenant Authorization

### Why This Matters

Multi-tenant systems must ensure:
- Tenant A cannot see Tenant B's data
- Users can only access their tenant
- Cross-tenant queries are impossible

### What Goes Wrong Without This

**War story — The Tenant Leak:**
Search endpoint didn't filter by tenant. User searched "invoice" and saw invoices from 50 other companies. GDPR violation, massive breach notification required.

### Tenant Isolation

```csharp
// Tenant context
public interface ITenantContext
{
    Guid TenantId { get; }
    string TenantName { get; }
}

public class TenantContext : ITenantContext
{
    public Guid TenantId { get; set; }
    public string TenantName { get; set; } = string.Empty;
}

// Middleware to set tenant from claims
public class TenantMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context, ITenantContext tenantContext)
    {
        if (context.User.Identity?.IsAuthenticated == true)
        {
            var tenantClaim = context.User.FindFirstValue("tenant_id");
            if (tenantClaim != null)
            {
                ((TenantContext)tenantContext).TenantId = Guid.Parse(tenantClaim);
                ((TenantContext)tenantContext).TenantName =
                    context.User.FindFirstValue("tenant_name") ?? "";
            }
        }

        await _next(context);
    }
}

// Global query filter for tenant isolation
public class OrderFlowDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    public OrderFlowDbContext(
        DbContextOptions<OrderFlowDbContext> options,
        ITenantContext tenantContext)
        : base(options)
    {
        _tenantContext = tenantContext;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Every tenant-scoped entity gets a filter
        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenantContext.TenantId);

        modelBuilder.Entity<Customer>()
            .HasQueryFilter(c => c.TenantId == _tenantContext.TenantId);

        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantContext.TenantId);
    }

    // Automatically set TenantId on save
    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.TenantId = _tenantContext.TenantId;
            }
        }

        return base.SaveChangesAsync(cancellationToken);
    }
}

// Authorization handler for tenant access
public class TenantAccessHandler : AuthorizationHandler<TenantAccessRequirement, ITenantEntity>
{
    private readonly ITenantContext _tenantContext;

    public TenantAccessHandler(ITenantContext tenantContext)
    {
        _tenantContext = tenantContext;
    }

    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        TenantAccessRequirement requirement,
        ITenantEntity resource)
    {
        if (resource.TenantId == _tenantContext.TenantId)
        {
            context.Succeed(requirement);
        }
        else
        {
            // Log suspicious cross-tenant access attempt
            var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
            // _logger.LogWarning("Cross-tenant access attempt by {UserId}", userId);
        }

        return Task.CompletedTask;
    }
}
```

---

## Core Concept #16 — Authorization Logging and Audit

### Why This Matters

Authorization decisions must be auditable:
- Who accessed what?
- Who was denied access?
- When did permissions change?

### Logging Authorization Events

```csharp
// Custom authorization handler with logging
public class LoggingAuthorizationHandler<TRequirement, TResource> :
    AuthorizationHandler<TRequirement, TResource>
    where TRequirement : IAuthorizationRequirement
{
    private readonly ILogger _logger;
    private readonly AuthorizationHandler<TRequirement, TResource> _inner;

    public LoggingAuthorizationHandler(
        ILogger<LoggingAuthorizationHandler<TRequirement, TResource>> logger,
        AuthorizationHandler<TRequirement, TResource> inner)
    {
        _logger = logger;
        _inner = inner;
    }

    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        TRequirement requirement,
        TResource resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        var resourceType = typeof(TResource).Name;
        var requirementType = typeof(TRequirement).Name;

        var stopwatch = Stopwatch.StartNew();

        await _inner.HandleRequirementAsync(context, requirement, resource);

        stopwatch.Stop();

        var granted = context.HasSucceeded;

        _logger.LogInformation(
            "Authorization {Result}: User={UserId} Requirement={Requirement} " +
            "Resource={ResourceType} Duration={Duration}ms",
            granted ? "Granted" : "Denied",
            userId,
            requirementType,
            resourceType,
            stopwatch.ElapsedMilliseconds);
    }
}

// Audit log for sensitive operations
public class AuditLogger
{
    private readonly OrderFlowDbContext _context;

    public async Task LogAccessAsync(
        string userId,
        string action,
        string resourceType,
        string resourceId,
        bool granted,
        string? reason = null)
    {
        _context.AuditLogs.Add(new AuditLog
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Action = action,
            ResourceType = resourceType,
            ResourceId = resourceId,
            Granted = granted,
            Reason = reason,
            Timestamp = DateTime.UtcNow,
            IpAddress = GetClientIp(),
            UserAgent = GetUserAgent()
        });

        await _context.SaveChangesAsync();
    }
}

public class AuditLog
{
    public Guid Id { get; set; }
    public string UserId { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty;
    public string ResourceType { get; set; } = string.Empty;
    public string ResourceId { get; set; } = string.Empty;
    public bool Granted { get; set; }
    public string? Reason { get; set; }
    public DateTime Timestamp { get; set; }
    public string? IpAddress { get; set; }
    public string? UserAgent { get; set; }
}
```

### Tracking Permission Changes

```csharp
// Log when user roles/permissions change
public class RoleChangeService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly AuditLogger _auditLogger;

    public async Task AddToRoleAsync(string userId, string role, string changedBy)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user is null) throw new NotFoundException("User not found");

        var result = await _userManager.AddToRoleAsync(user, role);

        if (result.Succeeded)
        {
            await _auditLogger.LogAccessAsync(
                changedBy,
                "RoleAdded",
                "User",
                userId,
                granted: true,
                reason: $"Added role: {role}");
        }
    }

    public async Task RemoveFromRoleAsync(string userId, string role, string changedBy)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user is null) throw new NotFoundException("User not found");

        var result = await _userManager.RemoveFromRoleAsync(user, role);

        if (result.Succeeded)
        {
            await _auditLogger.LogAccessAsync(
                changedBy,
                "RoleRemoved",
                "User",
                userId,
                granted: true,
                reason: $"Removed role: {role}");
        }
    }
}
```

---

## Core Concept #17 — Testing Authorization

### Why This Matters

Authorization bugs are security bugs. They must be tested:
- Positive tests (allowed access works)
- Negative tests (denied access fails)
- Boundary tests (edge cases)

### Integration Tests for Authorization

```csharp
public class OrderAuthorizationTests : IClassFixture<OrderFlowWebAppFactory>
{
    private readonly HttpClient _client;
    private readonly OrderFlowWebAppFactory _factory;

    [Fact]
    public async Task Owner_can_view_own_order()
    {
        // Arrange
        var userId = Guid.NewGuid().ToString();
        var order = await SeedOrderForUser(userId);

        var client = _factory.CreateClient()
            .WithUser(userId);

        // Act
        var response = await client.GetAsync($"/api/orders/{order.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task User_cannot_view_other_users_order()
    {
        // Arrange
        var ownerUserId = Guid.NewGuid().ToString();
        var otherUserId = Guid.NewGuid().ToString();
        var order = await SeedOrderForUser(ownerUserId);

        var client = _factory.CreateClient()
            .WithUser(otherUserId);

        // Act
        var response = await client.GetAsync($"/api/orders/{order.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    [Fact]
    public async Task Admin_can_view_any_order()
    {
        // Arrange
        var ownerUserId = Guid.NewGuid().ToString();
        var adminUserId = Guid.NewGuid().ToString();
        var order = await SeedOrderForUser(ownerUserId);

        var client = _factory.CreateClient()
            .WithUser(adminUserId, "Admin");

        // Act
        var response = await client.GetAsync($"/api/orders/{order.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task Unauthenticated_user_gets_401()
    {
        var client = _factory.CreateClient();
        // No auth header

        var response = await client.GetAsync("/api/orders");

        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task User_without_scope_gets_403()
    {
        var client = _factory.CreateClient()
            .WithUser("user-123", scopes: new[] { "profile:read" });  // No orders scope

        var response = await client.GetAsync("/api/orders");

        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    [Theory]
    [InlineData("Customer", false)]
    [InlineData("Support", false)]
    [InlineData("Admin", true)]
    public async Task Only_admin_can_delete_order(string role, bool shouldSucceed)
    {
        var order = await SeedOrderForUser(Guid.NewGuid().ToString());
        var client = _factory.CreateClient()
            .WithUser(Guid.NewGuid().ToString(), role);

        var response = await client.DeleteAsync($"/api/orders/{order.Id}");

        if (shouldSucceed)
        {
            response.StatusCode.Should().Be(HttpStatusCode.NoContent);
        }
        else
        {
            response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
        }
    }
}
```

### Unit Tests for Handlers

```csharp
public class OrderOwnerHandlerTests
{
    [Fact]
    public async Task Owner_is_authorized()
    {
        var userId = Guid.NewGuid();
        var order = new Order(userId);
        var user = CreateClaimsPrincipal(userId.ToString());
        var context = new AuthorizationHandlerContext(
            new[] { new ResourceOwnerRequirement() },
            user,
            order);

        var handler = new ResourceOwnerHandler();

        await handler.HandleAsync(context);

        context.HasSucceeded.Should().BeTrue();
    }

    [Fact]
    public async Task Non_owner_is_not_authorized()
    {
        var ownerId = Guid.NewGuid();
        var otherUserId = Guid.NewGuid();
        var order = new Order(ownerId);
        var user = CreateClaimsPrincipal(otherUserId.ToString());
        var context = new AuthorizationHandlerContext(
            new[] { new ResourceOwnerRequirement() },
            user,
            order);

        var handler = new ResourceOwnerHandler();

        await handler.HandleAsync(context);

        context.HasSucceeded.Should().BeFalse();
    }

    private static ClaimsPrincipal CreateClaimsPrincipal(string userId)
    {
        var claims = new[] { new Claim(ClaimTypes.NameIdentifier, userId) };
        var identity = new ClaimsIdentity(claims, "Test");
        return new ClaimsPrincipal(identity);
    }
}
```

---

## Hands-On: OrderFlow Authorization

### Task 1 — Define Policies

Create policies for:
- Admin (full access)
- Support (read + refund)
- Customer (own orders only)
- API client (scope-based)

### Task 2 — Resource-Based Authorization

Implement:
- Order ownership check
- Admin-or-owner pattern
- Tenant isolation

### Task 3 — Audit Logging

Build:
- Authorization event logging
- Denied access alerts
- Permission change tracking

### Task 4 — Authorization Tests

Write tests for:
- All policy combinations
- Cross-tenant access attempts
- Scope validation

---

## Deliverables

1. **Policy definitions** with documentation
2. **Custom authorization handlers** for ownership and operations
3. **Multi-tenant isolation** with query filters
4. **Audit logging** for authorization events
5. **Test suite** covering all authorization scenarios

---

## Resources

### Must-Read
- [ASP.NET Core Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/introduction)
- [Policy-based Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies)
- [Resource-based Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased)

### Videos
- [Nick Chapsas: Authorization Deep Dive](https://www.youtube.com/watch?v=5XA2KHv3o_c)
- [NDC: ASP.NET Core Authorization](https://www.youtube.com/watch?v=RBMO_hruKaI)
- [Raw Coding: Policy Authorization](https://www.youtube.com/watch?v=eRxLfUCgO5A)

### Books
- [OAuth 2.0 in Action](https://www.manning.com/books/oauth-2-in-action) — Authorization deep dive
- [Identity and Data Security for Web Development](https://www.oreilly.com/library/view/identity-and-data/9781491937006/)

---

## Reflection Questions

1. When should you use roles vs claims vs policies?
2. How do you handle authorization for batch operations?
3. What's the difference between 401 and 403?
4. How do you test for authorization bypass vulnerabilities?

---

**Next:** [Part 4 — API Security Operations](./part-4.md)
