# 08 — Security, Authentication & Identity

## Part 2 / 4 — Authentication

---

## Why This Matters

Authentication answers: "Who are you?"

Get it wrong and:
- Attackers impersonate users
- Sessions get hijacked
- Tokens get stolen
- Credentials get leaked

Modern authentication is complex: JWTs, OAuth2, OIDC, refresh tokens, MFA. Understanding the pieces prevents costly mistakes.

---

## Core Concept #6 — JWT Authentication

### Why This Matters

JWTs (JSON Web Tokens) are the standard for API authentication. But they're easy to misuse:
- Not validating signatures
- Not checking expiration
- Storing sensitive data in payload
- Using weak signing keys

### What Goes Wrong Without This

**War story — The Unvalidated JWT:**
Team validated JWT structure but not signature. Attacker modified payload to change `role: "user"` to `role: "admin"`. Full admin access to production.

**War story — The Leaked Secret:**
JWT signing key was `"secret"`. Attacker brute-forced it in minutes. Created valid admin tokens at will.

### JWT Structure

```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4ifQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

```csharp
// JWT payload — NEVER include secrets
{
  "sub": "user-123",           // Subject (user ID)
  "name": "John Doe",          // Display name
  "email": "john@example.com", // Email
  "roles": ["Customer"],       // Roles
  "iat": 1516239022,           // Issued at
  "exp": 1516242622,           // Expiration
  "iss": "https://auth.orderflow.com",  // Issuer
  "aud": "orderflow-api"       // Audience
}

// ❌ NEVER include in JWT:
// - Passwords
// - Credit card numbers
// - Social security numbers
// - Any sensitive PII
```

### Configuring JWT Authentication

```csharp
// Program.cs — JWT Bearer Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Jwt:Authority"];
        options.Audience = builder.Configuration["Jwt:Audience"];

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],

            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],

            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(5),  // Allow 5 min clock drift

            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),

            // For asymmetric keys (RS256) — preferred for production
            // IssuerSigningKey is retrieved from Authority's JWKS endpoint
        };

        // Logging and custom validation
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                var logger = context.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();

                logger.LogWarning(
                    "Authentication failed: {Error}. Token: {Token}",
                    context.Exception.Message,
                    context.Request.Headers.Authorization.ToString().Substring(0, 20) + "...");

                return Task.CompletedTask;
            },

            OnTokenValidated = context =>
            {
                var logger = context.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();

                var userId = context.Principal?.FindFirstValue(ClaimTypes.NameIdentifier);
                logger.LogInformation("Token validated for user {UserId}", userId);

                return Task.CompletedTask;
            },

            OnChallenge = context =>
            {
                // Customize 401 response
                context.HandleResponse();
                context.Response.StatusCode = 401;
                context.Response.ContentType = "application/json";

                var result = JsonSerializer.Serialize(new
                {
                    error = "unauthorized",
                    message = "Invalid or missing token"
                });

                return context.Response.WriteAsync(result);
            }
        };
    });

// Apply to all endpoints by default
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

### Generating JWTs (for testing or simple scenarios)

```csharp
public class JwtTokenService
{
    private readonly IConfiguration _config;

    public string GenerateToken(User user)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_config["Jwt:Key"]!));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(ClaimTypes.Name, user.Name),
            new Claim(ClaimTypes.Role, user.Role),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };

        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),  // Short lifetime
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Core Concept #7 — OAuth2 and OpenID Connect

### Why This Matters

OAuth2 handles authorization ("what can you access"). OIDC adds authentication ("who are you") on top.

Don't build your own identity system. Use established providers:
- Azure AD / Entra ID
- Auth0
- Keycloak
- Okta

### OAuth2 Flows

```
Authorization Code Flow (with PKCE) — For SPAs and mobile apps
┌──────────┐                              ┌──────────┐
│  Client  │                              │  Auth    │
│  (SPA)   │                              │  Server  │
└────┬─────┘                              └────┬─────┘
     │  1. Redirect to /authorize              │
     │  + code_challenge (PKCE)                │
     ├────────────────────────────────────────►│
     │                                         │
     │  2. User logs in                        │
     │                                         │
     │  3. Redirect back with auth code        │
     │◄────────────────────────────────────────┤
     │                                         │
     │  4. Exchange code for tokens            │
     │  + code_verifier (PKCE)                 │
     ├────────────────────────────────────────►│
     │                                         │
     │  5. Return access_token, id_token,      │
     │     refresh_token                       │
     │◄────────────────────────────────────────┤


Client Credentials Flow — For service-to-service
┌──────────┐                              ┌──────────┐
│  Service │                              │  Auth    │
│    A     │                              │  Server  │
└────┬─────┘                              └────┬─────┘
     │  1. POST /token                         │
     │  client_id + client_secret              │
     ├────────────────────────────────────────►│
     │                                         │
     │  2. Return access_token                 │
     │◄────────────────────────────────────────┤
```

### Configuring OIDC with Azure AD

```csharp
// Program.cs — Azure AD / Entra ID
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

// appsettings.json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "your-tenant-id",
    "ClientId": "your-api-client-id",
    "Audience": "api://your-api-client-id",
    "Scopes": "Orders.Read Orders.Write"
  }
}
```

### Configuring OIDC with Auth0

```csharp
// Program.cs — Auth0
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = $"https://{builder.Configuration["Auth0:Domain"]}/";
        options.Audience = builder.Configuration["Auth0:Audience"];

        options.TokenValidationParameters = new TokenValidationParameters
        {
            NameClaimType = ClaimTypes.NameIdentifier
        };
    });

// appsettings.json
{
  "Auth0": {
    "Domain": "your-tenant.auth0.com",
    "Audience": "https://api.orderflow.com"
  }
}
```

### Configuring OIDC with Keycloak

```csharp
// Program.cs — Keycloak
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Keycloak:Authority"];
        options.Audience = builder.Configuration["Keycloak:Audience"];

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Keycloak:Audience"],
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Keycloak:Authority"],
            NameClaimType = "preferred_username",
            RoleClaimType = "roles"
        };

        // Keycloak puts roles in realm_access.roles
        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = context =>
            {
                MapKeycloakRoles(context);
                return Task.CompletedTask;
            }
        };
    });

static void MapKeycloakRoles(TokenValidatedContext context)
{
    var resourceAccess = context.Principal?.FindFirstValue("realm_access");
    if (resourceAccess == null) return;

    var roles = JsonDocument.Parse(resourceAccess)
        .RootElement
        .GetProperty("roles")
        .EnumerateArray()
        .Select(r => new Claim(ClaimTypes.Role, r.GetString()!));

    var identity = context.Principal?.Identity as ClaimsIdentity;
    identity?.AddClaims(roles);
}
```

---

## Core Concept #8 — Refresh Tokens

### Why This Matters

Access tokens should be short-lived (minutes to hours). Refresh tokens allow getting new access tokens without re-authenticating.

### What Goes Wrong Without This

**War story — The Long-Lived Token:**
Team used 30-day access tokens for "convenience." Token leaked in logs. Attacker had access for weeks before anyone noticed.

### Refresh Token Flow

```csharp
// Token refresh endpoint
app.MapPost("/auth/refresh", async (
    RefreshTokenRequest request,
    ITokenService tokenService,
    IRefreshTokenRepository refreshTokens) =>
{
    // Validate refresh token
    var storedToken = await refreshTokens.GetAsync(request.RefreshToken);

    if (storedToken is null)
        return Results.Unauthorized();

    if (storedToken.ExpiresAt < DateTime.UtcNow)
    {
        await refreshTokens.RevokeAsync(request.RefreshToken);
        return Results.Unauthorized();
    }

    if (storedToken.IsRevoked)
        return Results.Unauthorized();

    // Rotate refresh token (one-time use)
    await refreshTokens.RevokeAsync(request.RefreshToken);

    var user = await GetUserAsync(storedToken.UserId);

    // Issue new tokens
    var newAccessToken = tokenService.GenerateAccessToken(user);
    var newRefreshToken = tokenService.GenerateRefreshToken();

    await refreshTokens.StoreAsync(new RefreshToken
    {
        Token = newRefreshToken,
        UserId = user.Id,
        ExpiresAt = DateTime.UtcNow.AddDays(7),
        CreatedAt = DateTime.UtcNow
    });

    return Results.Ok(new
    {
        AccessToken = newAccessToken,
        RefreshToken = newRefreshToken,
        ExpiresIn = 3600  // 1 hour
    });
});

public class RefreshToken
{
    public string Token { get; set; } = string.Empty;
    public Guid UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsRevoked { get; set; }
    public DateTime? RevokedAt { get; set; }
    public string? ReplacedByToken { get; set; }
}
```

### Secure Refresh Token Storage

```csharp
// Store refresh tokens securely
public class RefreshTokenRepository : IRefreshTokenRepository
{
    private readonly OrderFlowDbContext _context;

    public async Task StoreAsync(RefreshToken token)
    {
        // Hash the token before storing
        var hashedToken = HashToken(token.Token);

        _context.RefreshTokens.Add(new RefreshTokenEntity
        {
            TokenHash = hashedToken,
            UserId = token.UserId,
            ExpiresAt = token.ExpiresAt,
            CreatedAt = token.CreatedAt
        });

        await _context.SaveChangesAsync();
    }

    public async Task<RefreshToken?> GetAsync(string token)
    {
        var hashedToken = HashToken(token);

        var entity = await _context.RefreshTokens
            .FirstOrDefaultAsync(t => t.TokenHash == hashedToken);

        if (entity is null) return null;

        return new RefreshToken
        {
            Token = token,  // Return original for response
            UserId = entity.UserId,
            ExpiresAt = entity.ExpiresAt,
            IsRevoked = entity.IsRevoked
        };
    }

    private static string HashToken(string token)
    {
        using var sha256 = SHA256.Create();
        var bytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(token));
        return Convert.ToBase64String(bytes);
    }
}
```

---

## Core Concept #9 — Session Management

### Why This Matters

Sessions track authenticated users across requests. Poor session management leads to:
- Session hijacking
- Session fixation
- Concurrent session abuse

### Cookie-Based Sessions

```csharp
// Secure session configuration
builder.Services.AddSession(options =>
{
    options.Cookie.Name = "OrderFlow.Session";
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.IsEssential = true;
});

// Distributed session storage (for multiple instances)
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "OrderFlow:";
});

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
});
```

### Session Invalidation

```csharp
// Force logout / session invalidation
public class SessionService
{
    private readonly IDistributedCache _cache;
    private readonly IHttpContextAccessor _httpContext;

    public async Task InvalidateSessionAsync(string userId)
    {
        // Store invalidation timestamp
        var key = $"session:invalidated:{userId}";
        await _cache.SetStringAsync(
            key,
            DateTime.UtcNow.ToString("O"),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(7)
            });
    }

    public async Task<bool> IsSessionValidAsync(ClaimsPrincipal user)
    {
        var userId = user.FindFirstValue(ClaimTypes.NameIdentifier);
        if (userId is null) return false;

        var tokenIssuedAt = user.FindFirstValue("iat");
        if (tokenIssuedAt is null) return false;

        var key = $"session:invalidated:{userId}";
        var invalidatedAt = await _cache.GetStringAsync(key);

        if (invalidatedAt is null) return true;

        // Token must be issued after invalidation
        var tokenTime = DateTimeOffset.FromUnixTimeSeconds(long.Parse(tokenIssuedAt));
        var invalidationTime = DateTime.Parse(invalidatedAt);

        return tokenTime.UtcDateTime > invalidationTime;
    }
}

// Middleware to check session validity
public class SessionValidationMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context, SessionService sessionService)
    {
        if (context.User.Identity?.IsAuthenticated == true)
        {
            if (!await sessionService.IsSessionValidAsync(context.User))
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsJsonAsync(new
                {
                    error = "session_invalidated",
                    message = "Your session has been invalidated. Please log in again."
                });
                return;
            }
        }

        await _next(context);
    }
}
```

---

## Core Concept #10 — Multi-Factor Authentication

### Why This Matters

Passwords alone aren't enough. MFA adds a second factor:
- Something you know (password)
- Something you have (phone, hardware key)
- Something you are (biometrics)

### MFA with TOTP

```csharp
// Configure MFA with ASP.NET Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.SignIn.RequireConfirmedAccount = true;
    options.Tokens.AuthenticatorTokenProvider = TokenOptions.DefaultAuthenticatorProvider;
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();

// Enable MFA for user
public class MfaService
{
    private readonly UserManager<ApplicationUser> _userManager;

    public async Task<MfaSetupResult> SetupMfaAsync(string userId)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user is null) throw new NotFoundException("User not found");

        // Generate authenticator key
        await _userManager.ResetAuthenticatorKeyAsync(user);
        var key = await _userManager.GetAuthenticatorKeyAsync(user);

        // Generate QR code URI
        var email = await _userManager.GetEmailAsync(user);
        var uri = GenerateQrCodeUri(email!, key!);

        return new MfaSetupResult
        {
            SharedKey = key!,
            QrCodeUri = uri
        };
    }

    public async Task<bool> VerifyMfaCodeAsync(string userId, string code)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user is null) return false;

        var isValid = await _userManager.VerifyTwoFactorTokenAsync(
            user,
            _userManager.Options.Tokens.AuthenticatorTokenProvider,
            code);

        if (isValid)
        {
            await _userManager.SetTwoFactorEnabledAsync(user, true);
        }

        return isValid;
    }

    private static string GenerateQrCodeUri(string email, string key)
    {
        return $"otpauth://totp/OrderFlow:{email}?secret={key}&issuer=OrderFlow&digits=6";
    }
}

// Login with MFA
app.MapPost("/auth/login", async (
    LoginRequest request,
    SignInManager<ApplicationUser> signInManager,
    UserManager<ApplicationUser> userManager) =>
{
    var user = await userManager.FindByEmailAsync(request.Email);
    if (user is null)
        return Results.Unauthorized();

    var result = await signInManager.CheckPasswordSignInAsync(
        user, request.Password, lockoutOnFailure: true);

    if (result.RequiresTwoFactor)
    {
        // Return partial token for MFA step
        return Results.Ok(new
        {
            RequiresMfa = true,
            MfaToken = GenerateMfaToken(user.Id)
        });
    }

    if (!result.Succeeded)
        return Results.Unauthorized();

    // Generate full tokens
    var tokens = await GenerateTokensAsync(user);
    return Results.Ok(tokens);
});

app.MapPost("/auth/mfa", async (
    MfaRequest request,
    SignInManager<ApplicationUser> signInManager,
    UserManager<ApplicationUser> userManager) =>
{
    var userId = ValidateMfaToken(request.MfaToken);
    if (userId is null)
        return Results.Unauthorized();

    var user = await userManager.FindByIdAsync(userId);
    if (user is null)
        return Results.Unauthorized();

    var result = await signInManager.TwoFactorAuthenticatorSignInAsync(
        request.Code, isPersistent: false, rememberClient: false);

    if (!result.Succeeded)
        return Results.Unauthorized();

    var tokens = await GenerateTokensAsync(user);
    return Results.Ok(tokens);
});
```

---

## Core Concept #11 — API Key Authentication

### Why This Matters

API keys are simpler than OAuth for service-to-service or simple integrations. But they're often mishandled:
- Sent over HTTP (not HTTPS)
- Logged in plain text
- Never rotated
- Shared in source code

### Implementing API Key Auth

```csharp
// API Key authentication handler
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
{
    private const string ApiKeyHeaderName = "X-API-Key";
    private readonly IApiKeyRepository _apiKeyRepository;

    public ApiKeyAuthenticationHandler(
        IOptionsMonitor<ApiKeyAuthenticationOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        IApiKeyRepository apiKeyRepository)
        : base(options, logger, encoder)
    {
        _apiKeyRepository = apiKeyRepository;
    }

    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(ApiKeyHeaderName, out var apiKeyHeader))
        {
            return AuthenticateResult.NoResult();
        }

        var providedKey = apiKeyHeader.ToString();

        var apiKey = await _apiKeyRepository.ValidateAsync(providedKey);
        if (apiKey is null)
        {
            Logger.LogWarning(
                "Invalid API key attempted. Key prefix: {KeyPrefix}",
                providedKey.Substring(0, Math.Min(8, providedKey.Length)));

            return AuthenticateResult.Fail("Invalid API key");
        }

        if (apiKey.ExpiresAt < DateTime.UtcNow)
        {
            return AuthenticateResult.Fail("API key expired");
        }

        // Update last used timestamp
        await _apiKeyRepository.UpdateLastUsedAsync(apiKey.Id);

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, apiKey.ClientId),
            new Claim(ClaimTypes.Name, apiKey.ClientName),
            new Claim("api_key_id", apiKey.Id.ToString())
        };

        foreach (var scope in apiKey.Scopes)
        {
            claims = claims.Append(new Claim("scope", scope)).ToArray();
        }

        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }
}

// API Key model
public class ApiKey
{
    public Guid Id { get; set; }
    public string KeyHash { get; set; } = string.Empty;  // Never store plain key
    public string KeyPrefix { get; set; } = string.Empty;  // First 8 chars for identification
    public string ClientId { get; set; } = string.Empty;
    public string ClientName { get; set; } = string.Empty;
    public List<string> Scopes { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public DateTime? ExpiresAt { get; set; }
    public DateTime? LastUsedAt { get; set; }
    public bool IsRevoked { get; set; }
}

// API Key generation
public class ApiKeyService
{
    private readonly IApiKeyRepository _repository;

    public async Task<ApiKeyCreateResult> CreateAsync(CreateApiKeyRequest request)
    {
        // Generate secure random key
        var keyBytes = RandomNumberGenerator.GetBytes(32);
        var key = $"of_{Convert.ToBase64String(keyBytes).Replace("+", "-").Replace("/", "_")}";

        var apiKey = new ApiKey
        {
            Id = Guid.NewGuid(),
            KeyHash = HashKey(key),
            KeyPrefix = key.Substring(0, 8),
            ClientId = request.ClientId,
            ClientName = request.ClientName,
            Scopes = request.Scopes,
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddYears(1)
        };

        await _repository.CreateAsync(apiKey);

        // Return plain key only once — it cannot be recovered
        return new ApiKeyCreateResult
        {
            ApiKey = key,  // Show once, never again
            KeyId = apiKey.Id,
            ExpiresAt = apiKey.ExpiresAt
        };
    }

    private static string HashKey(string key)
    {
        using var sha256 = SHA256.Create();
        var bytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(key));
        return Convert.ToBase64String(bytes);
    }
}
```

---

## Hands-On: OrderFlow Authentication

### Task 1 — JWT Configuration

Configure JWT authentication:
- Token validation parameters
- Event handlers for logging
- Custom 401 responses

### Task 2 — Identity Provider Integration

Integrate with one provider:
- Azure AD, Auth0, or Keycloak
- Configure scopes
- Test token acquisition

### Task 3 — Refresh Token Flow

Implement:
- Refresh token endpoint
- Token rotation
- Secure storage

### Task 4 — API Keys

Build API key system for:
- External integrations
- Secure generation
- Rotation workflow

---

## Deliverables

1. **JWT configuration** with proper validation
2. **OIDC integration** with one provider
3. **Refresh token flow** with rotation
4. **API key system** for service integrations

---

## Resources

### Must-Read
- [JWT.io](https://jwt.io/) — JWT debugger and intro
- [OAuth 2.0 Simplified](https://www.oauth.com/)
- [Microsoft Identity Platform](https://learn.microsoft.com/en-us/azure/active-directory/develop/)

### Videos
- [OAuth 2.0 and OpenID Connect](https://www.youtube.com/watch?v=996OiexHze0)
- [Nick Chapsas: JWT Authentication](https://www.youtube.com/watch?v=Y-MjFcRCBtw)
- [Auth0: PKCE Flow Explained](https://www.youtube.com/watch?v=gAP0WRBiPaI)

### Tools
- [jwt.io](https://jwt.io/) — JWT decoder
- [oauth.tools](https://oauth.tools/) — OAuth playground
- [Postman](https://www.postman.com/) — API testing with auth

---

## Reflection Questions

1. Why should access tokens be short-lived?
2. What's the difference between OAuth2 and OIDC?
3. Why use PKCE for public clients?
4. How do you handle token revocation?

---

**Next:** [Part 3 — Authorization](./part-3.md)
