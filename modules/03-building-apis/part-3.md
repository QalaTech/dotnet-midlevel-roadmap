# 03 — Building Web APIs with ASP.NET Core

## Part 3 / 4 — Pagination, Files, and Background Processing

---

## Why This Matters

Real APIs don't just return single records. They handle:
- Thousands of records that need pagination
- File uploads and downloads
- Long-running operations that can't block HTTP requests
- Integration with external systems via HTTP clients

This part covers the patterns that make APIs usable at scale.

---

## Core Concept #9 — Pagination Implementation

### Why This Matters

Module 02 covered pagination design. Now we implement it correctly:
- Database-efficient queries (not loading everything into memory)
- Consistent response shapes
- Proper handling of edge cases

### What Goes Wrong Without This

**Real scenario:** A developer loads all orders into memory, then paginates in C#. Works fine with 100 orders. With 100,000 orders, the server runs out of memory and crashes.

**Real scenario:** Offset pagination with `OFFSET 50000` on a table with millions of rows. Query takes 30 seconds. Database CPU spikes. Other queries queue up.

### Offset Pagination (Simple but Limited)

```csharp
// Query parameters
public record OrderQueryParameters
{
    public int Page { get; init; } = 1;
    public int PageSize { get; init; } = 20;
    public string? Status { get; init; }
    public string? Sort { get; init; }

    public int Skip => (Page - 1) * PageSize;
}

// Response wrapper
public record PagedResponse<T>
{
    public IReadOnlyList<T> Data { get; init; } = Array.Empty<T>();
    public PaginationMetadata Pagination { get; init; } = null!;
}

public record PaginationMetadata
{
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalCount { get; init; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}

// Repository implementation
public class OrderRepository : IOrderRepository
{
    private readonly OrderFlowDbContext _context;

    public async Task<PagedResponse<Order>> GetPagedAsync(
        OrderQueryParameters query,
        CancellationToken ct)
    {
        var baseQuery = _context.Orders.AsQueryable();

        // Apply filters
        if (!string.IsNullOrEmpty(query.Status))
        {
            if (Enum.TryParse<OrderStatus>(query.Status, true, out var status))
            {
                baseQuery = baseQuery.Where(o => o.Status == status);
            }
        }

        // Get total count (executes COUNT query)
        var totalCount = await baseQuery.CountAsync(ct);

        // Apply sorting
        baseQuery = ApplySorting(baseQuery, query.Sort);

        // Apply pagination (executes SELECT with OFFSET/LIMIT)
        var items = await baseQuery
            .Skip(query.Skip)
            .Take(query.PageSize)
            .AsNoTracking()
            .ToListAsync(ct);

        return new PagedResponse<Order>
        {
            Data = items,
            Pagination = new PaginationMetadata
            {
                Page = query.Page,
                PageSize = query.PageSize,
                TotalCount = totalCount
            }
        };
    }

    private static IQueryable<Order> ApplySorting(IQueryable<Order> query, string? sort)
    {
        return sort?.ToLowerInvariant() switch
        {
            "createdat" or "createdat_asc" => query.OrderBy(o => o.CreatedAt),
            "createdat_desc" or "-createdat" => query.OrderByDescending(o => o.CreatedAt),
            "total" or "total_asc" => query.OrderBy(o => o.Total.Amount),
            "total_desc" or "-total" => query.OrderByDescending(o => o.Total.Amount),
            _ => query.OrderByDescending(o => o.CreatedAt)  // Default sort
        };
    }
}
```

### Cursor Pagination (Better for Large Datasets)

```csharp
public record CursorQueryParameters
{
    public string? Cursor { get; init; }
    public int Limit { get; init; } = 20;
}

public record CursorPagedResponse<T>
{
    public IReadOnlyList<T> Data { get; init; } = Array.Empty<T>();
    public string? NextCursor { get; init; }
    public bool HasMore { get; init; }
}

public class OrderRepository : IOrderRepository
{
    public async Task<CursorPagedResponse<Order>> GetWithCursorAsync(
        CursorQueryParameters query,
        CancellationToken ct)
    {
        var baseQuery = _context.Orders
            .OrderByDescending(o => o.CreatedAt)
            .ThenBy(o => o.Id);  // Tiebreaker for stable pagination

        // Apply cursor filter
        if (!string.IsNullOrEmpty(query.Cursor))
        {
            var (createdAt, id) = DecodeCursor(query.Cursor);
            baseQuery = baseQuery.Where(o =>
                o.CreatedAt < createdAt ||
                (o.CreatedAt == createdAt && o.Id.CompareTo(id) > 0));
        }

        // Fetch one extra to determine if there's more
        var items = await baseQuery
            .Take(query.Limit + 1)
            .AsNoTracking()
            .ToListAsync(ct);

        var hasMore = items.Count > query.Limit;
        var resultItems = items.Take(query.Limit).ToList();

        string? nextCursor = null;
        if (hasMore && resultItems.Count > 0)
        {
            var lastItem = resultItems[^1];
            nextCursor = EncodeCursor(lastItem.CreatedAt, lastItem.Id);
        }

        return new CursorPagedResponse<Order>
        {
            Data = resultItems,
            NextCursor = nextCursor,
            HasMore = hasMore
        };
    }

    private static string EncodeCursor(DateTime createdAt, Guid id)
    {
        var data = $"{createdAt:O}|{id}";
        return Convert.ToBase64String(Encoding.UTF8.GetBytes(data));
    }

    private static (DateTime CreatedAt, Guid Id) DecodeCursor(string cursor)
    {
        var data = Encoding.UTF8.GetString(Convert.FromBase64String(cursor));
        var parts = data.Split('|');
        return (DateTime.Parse(parts[0]), Guid.Parse(parts[1]));
    }
}
```

### Endpoint with Pagination

```csharp
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders").WithTags("Orders");

        group.MapGet("/", GetOrders)
            .WithName("GetOrders")
            .Produces<PagedResponse<OrderResponse>>(200);
    }

    private static async Task<IResult> GetOrders(
        [AsParameters] OrderQueryParameters query,
        IOrderService service,
        CancellationToken ct)
    {
        // Validate page size
        if (query.PageSize > 100)
        {
            return Results.Problem(
                title: "Invalid page size",
                detail: "Page size cannot exceed 100",
                statusCode: 400);
        }

        var result = await service.GetOrdersAsync(query, ct);
        return Results.Ok(result);
    }
}
```

---

## Core Concept #10 — File Handling

### Why This Matters

Almost every API needs to handle files:
- Profile pictures
- Documents
- Import/export data
- Reports

### What Goes Wrong Without This

**Real scenario:** A developer loads an entire 500MB file into memory to process it. Multiple concurrent uploads crash the server.

**Real scenario:** Files are stored in the web root. A URL pattern exposes all files. Sensitive documents become publicly accessible.

### File Upload

```csharp
// Endpoint for file upload
app.MapPost("/orders/{id:guid}/documents", async (
    Guid id,
    IFormFile file,
    IOrderService orderService,
    IFileStorageService fileStorage,
    CancellationToken ct) =>
{
    // Validate file
    if (file.Length == 0)
    {
        return Results.Problem(
            title: "Invalid file",
            detail: "File is empty",
            statusCode: 400);
    }

    if (file.Length > 10 * 1024 * 1024)  // 10MB limit
    {
        return Results.Problem(
            title: "File too large",
            detail: "Maximum file size is 10MB",
            statusCode: 400);
    }

    // Validate file type
    var allowedTypes = new[] { ".pdf", ".jpg", ".png", ".docx" };
    var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
    if (!allowedTypes.Contains(extension))
    {
        return Results.Problem(
            title: "Invalid file type",
            detail: $"Allowed types: {string.Join(", ", allowedTypes)}",
            statusCode: 400);
    }

    // Store file
    var fileId = await fileStorage.UploadAsync(
        file.OpenReadStream(),
        file.FileName,
        file.ContentType,
        ct);

    // Associate with order
    await orderService.AddDocumentAsync(id, fileId, file.FileName, ct);

    return Results.Created($"/orders/{id}/documents/{fileId}", new { fileId });
})
.DisableAntiforgery()  // Required for file uploads
.Accepts<IFormFile>("multipart/form-data")
.Produces(201)
.ProducesProblem(400)
.ProducesProblem(404);
```

### File Download with Streaming

```csharp
app.MapGet("/orders/{orderId:guid}/documents/{fileId:guid}", async (
    Guid orderId,
    Guid fileId,
    IOrderService orderService,
    IFileStorageService fileStorage,
    CancellationToken ct) =>
{
    // Verify file belongs to order
    var document = await orderService.GetDocumentAsync(orderId, fileId, ct);
    if (document is null)
    {
        return Results.NotFound();
    }

    // Stream file (don't load into memory)
    var stream = await fileStorage.OpenReadStreamAsync(fileId, ct);
    if (stream is null)
    {
        return Results.NotFound();
    }

    return Results.File(
        stream,
        contentType: document.ContentType,
        fileDownloadName: document.FileName);
});
```

### File Storage Service

```csharp
public interface IFileStorageService
{
    Task<Guid> UploadAsync(Stream stream, string fileName, string contentType, CancellationToken ct);
    Task<Stream?> OpenReadStreamAsync(Guid fileId, CancellationToken ct);
    Task DeleteAsync(Guid fileId, CancellationToken ct);
}

// Local storage implementation (for development)
public class LocalFileStorageService : IFileStorageService
{
    private readonly string _basePath;
    private readonly ILogger<LocalFileStorageService> _logger;

    public LocalFileStorageService(
        IOptions<FileStorageSettings> options,
        ILogger<LocalFileStorageService> logger)
    {
        _basePath = options.Value.LocalPath;
        _logger = logger;
        Directory.CreateDirectory(_basePath);
    }

    public async Task<Guid> UploadAsync(
        Stream stream,
        string fileName,
        string contentType,
        CancellationToken ct)
    {
        var fileId = Guid.NewGuid();
        var extension = Path.GetExtension(fileName);
        var filePath = GetFilePath(fileId, extension);

        await using var fileStream = File.Create(filePath);
        await stream.CopyToAsync(fileStream, ct);

        _logger.LogInformation(
            "File uploaded: {FileId}, Size: {Size} bytes",
            fileId,
            fileStream.Length);

        return fileId;
    }

    public Task<Stream?> OpenReadStreamAsync(Guid fileId, CancellationToken ct)
    {
        var files = Directory.GetFiles(_basePath, $"{fileId}.*");
        if (files.Length == 0)
        {
            return Task.FromResult<Stream?>(null);
        }

        return Task.FromResult<Stream?>(File.OpenRead(files[0]));
    }

    public Task DeleteAsync(Guid fileId, CancellationToken ct)
    {
        var files = Directory.GetFiles(_basePath, $"{fileId}.*");
        foreach (var file in files)
        {
            File.Delete(file);
        }
        return Task.CompletedTask;
    }

    private string GetFilePath(Guid fileId, string extension)
    {
        return Path.Combine(_basePath, $"{fileId}{extension}");
    }
}

// Azure Blob Storage implementation (for production)
public class AzureBlobStorageService : IFileStorageService
{
    private readonly BlobContainerClient _containerClient;

    public AzureBlobStorageService(IOptions<FileStorageSettings> options)
    {
        var client = new BlobServiceClient(options.Value.AzureConnectionString);
        _containerClient = client.GetBlobContainerClient(options.Value.ContainerName);
    }

    public async Task<Guid> UploadAsync(
        Stream stream,
        string fileName,
        string contentType,
        CancellationToken ct)
    {
        var fileId = Guid.NewGuid();
        var blobName = $"{fileId}{Path.GetExtension(fileName)}";

        var blobClient = _containerClient.GetBlobClient(blobName);

        await blobClient.UploadAsync(
            stream,
            new BlobHttpHeaders { ContentType = contentType },
            cancellationToken: ct);

        return fileId;
    }

    public async Task<Stream?> OpenReadStreamAsync(Guid fileId, CancellationToken ct)
    {
        // Find blob by prefix
        await foreach (var blob in _containerClient.GetBlobsAsync(prefix: fileId.ToString(), cancellationToken: ct))
        {
            var blobClient = _containerClient.GetBlobClient(blob.Name);
            return await blobClient.OpenReadAsync(cancellationToken: ct);
        }
        return null;
    }

    public async Task DeleteAsync(Guid fileId, CancellationToken ct)
    {
        await foreach (var blob in _containerClient.GetBlobsAsync(prefix: fileId.ToString(), cancellationToken: ct))
        {
            await _containerClient.DeleteBlobAsync(blob.Name, cancellationToken: ct);
        }
    }
}
```

---

## Core Concept #11 — Background Jobs

### Why This Matters

Some operations shouldn't block HTTP requests:
- Sending emails
- Processing imports
- Generating reports
- Syncing with external systems

### What Goes Wrong Without This

**Real scenario:** Order creation sends confirmation emails synchronously. Email service is slow. Every order creation takes 5 seconds. Users abandon carts.

**Real scenario:** A large CSV import is processed in the HTTP request. It takes 10 minutes. The load balancer times out at 60 seconds. Import fails halfway through.

### Simple Background Jobs with IHostedService

```csharp
public class OrderNotificationService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderNotificationService> _logger;
    private readonly Channel<OrderCreatedEvent> _channel;

    public OrderNotificationService(
        IServiceScopeFactory scopeFactory,
        ILogger<OrderNotificationService> logger,
        Channel<OrderCreatedEvent> channel)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
        _channel = channel;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order notification service started");

        await foreach (var orderEvent in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await ProcessNotificationAsync(orderEvent, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(
                    ex,
                    "Failed to process notification for order {OrderId}",
                    orderEvent.OrderId);
            }
        }
    }

    private async Task ProcessNotificationAsync(
        OrderCreatedEvent orderEvent,
        CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
        var orderRepo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();

        var order = await orderRepo.GetByIdAsync(orderEvent.OrderId, ct);
        if (order is null) return;

        await emailService.SendOrderConfirmationAsync(order, ct);

        _logger.LogInformation(
            "Sent confirmation email for order {OrderId}",
            orderEvent.OrderId);
    }
}

// Registration
builder.Services.AddSingleton(Channel.CreateUnbounded<OrderCreatedEvent>());
builder.Services.AddHostedService<OrderNotificationService>();

// Publishing events (in OrderService)
public class OrderService : IOrderService
{
    private readonly Channel<OrderCreatedEvent> _eventChannel;

    public async Task<OrderResponse> CreateAsync(CreateOrderRequest request, CancellationToken ct)
    {
        var order = await CreateOrderInternal(request, ct);

        // Queue notification (doesn't block)
        await _eventChannel.Writer.WriteAsync(
            new OrderCreatedEvent(order.Id),
            ct);

        return MapToResponse(order);
    }
}
```

### Scheduled Jobs with Timer

```csharp
public class OverdueOrderProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OverdueOrderProcessor> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromHours(1);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Overdue order processor started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessOverdueOrdersAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing overdue orders");
            }

            await Task.Delay(_interval, stoppingToken);
        }
    }

    private async Task ProcessOverdueOrdersAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var orderRepo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();

        var overdueOrders = await orderRepo.GetOverdueOrdersAsync(ct);

        _logger.LogInformation(
            "Found {Count} overdue orders to process",
            overdueOrders.Count);

        foreach (var order in overdueOrders)
        {
            await ProcessSingleOverdueOrderAsync(scope.ServiceProvider, order, ct);
        }
    }

    private async Task ProcessSingleOverdueOrderAsync(
        IServiceProvider services,
        Order order,
        CancellationToken ct)
    {
        var orderService = services.GetRequiredService<IOrderService>();
        var emailService = services.GetRequiredService<IEmailService>();

        await orderService.MarkOverdueAsync(order.Id, ct);
        await emailService.SendOverdueNotificationAsync(order, ct);

        _logger.LogInformation(
            "Processed overdue order {OrderId}",
            order.Id);
    }
}
```

### Using Hangfire for Production Jobs

```csharp
// Install: dotnet add package Hangfire.AspNetCore
// Install: dotnet add package Hangfire.SqlServer

// Program.cs
builder.Services.AddHangfire(config =>
    config.UseSqlServerStorage(connectionString));
builder.Services.AddHangfireServer();

app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new HangfireAuthorizationFilter() }
});

// Define jobs
public interface IOrderJobs
{
    Task ProcessOrderAsync(Guid orderId, CancellationToken ct);
    Task SendOrderConfirmationAsync(Guid orderId, CancellationToken ct);
    Task GenerateInvoiceAsync(Guid orderId, CancellationToken ct);
}

// Queue a job
public class OrderService : IOrderService
{
    private readonly IBackgroundJobClient _backgroundJobs;

    public async Task<OrderResponse> CreateAsync(CreateOrderRequest request, CancellationToken ct)
    {
        var order = await CreateOrderInternal(request, ct);

        // Fire-and-forget job
        _backgroundJobs.Enqueue<IOrderJobs>(
            jobs => jobs.SendOrderConfirmationAsync(order.Id, CancellationToken.None));

        // Delayed job (generate invoice after 24 hours)
        _backgroundJobs.Schedule<IOrderJobs>(
            jobs => jobs.GenerateInvoiceAsync(order.Id, CancellationToken.None),
            TimeSpan.FromHours(24));

        return MapToResponse(order);
    }
}

// Recurring jobs
RecurringJob.AddOrUpdate<IOrderJobs>(
    "process-overdue-orders",
    jobs => jobs.ProcessOverdueOrdersAsync(CancellationToken.None),
    Cron.Hourly);
```

---

## Core Concept #12 — HTTP Client Best Practices

### Why This Matters

Most APIs call other APIs. Doing this wrong causes:
- Socket exhaustion
- DNS issues
- No retry logic
- Blocking the thread pool

### What Goes Wrong Without This

**Real scenario:** A developer creates a new `HttpClient` in every request. Under load, the server runs out of sockets. Connections start failing. The fix requires a restart.

**Real scenario:** An external API goes down. Every call fails immediately. Customers see errors. Nobody retries. When the API recovers, nothing auto-heals.

### Using IHttpClientFactory

```csharp
// Program.cs
builder.Services.AddHttpClient<IPaymentGateway, StripePaymentGateway>(client =>
{
    client.BaseAddress = new Uri("https://api.stripe.com/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

// Typed client
public class StripePaymentGateway : IPaymentGateway
{
    private readonly HttpClient _client;
    private readonly ILogger<StripePaymentGateway> _logger;

    public StripePaymentGateway(HttpClient client, ILogger<StripePaymentGateway> logger)
    {
        _client = client;  // Injected, managed by factory
        _logger = logger;
    }

    public async Task<PaymentResult> ChargeAsync(
        string customerId,
        decimal amount,
        string currency,
        CancellationToken ct)
    {
        var request = new
        {
            customer = customerId,
            amount = (int)(amount * 100),  // Stripe uses cents
            currency = currency.ToLowerInvariant()
        };

        var response = await _client.PostAsJsonAsync("v1/charges", request, ct);

        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadAsStringAsync(ct);
            _logger.LogError(
                "Payment failed: {StatusCode} - {Error}",
                response.StatusCode,
                error);

            throw new PaymentFailedException(error);
        }

        var result = await response.Content.ReadFromJsonAsync<StripeCharge>(ct);
        return new PaymentResult(result!.Id, result.Status == "succeeded");
    }
}
```

### Resilience with Polly

```csharp
// Install: dotnet add package Microsoft.Extensions.Http.Polly

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()  // 5xx and 408 (timeout)
        .OrResult(msg => msg.StatusCode == HttpStatusCode.TooManyRequests)
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt =>
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),  // Exponential backoff
            onRetry: (outcome, delay, retryAttempt, context) =>
            {
                // Log retry (context has ILogger if you add it)
                Console.WriteLine($"Retry {retryAttempt} after {delay.TotalSeconds}s");
            });
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (result, duration) =>
            {
                Console.WriteLine($"Circuit breaker opened for {duration.TotalSeconds}s");
            },
            onReset: () =>
            {
                Console.WriteLine("Circuit breaker reset");
            });
}
```

---

## Hands-On: OrderFlow Advanced Features

### Task 1 — Implement Pagination

Add pagination to the Orders endpoint:
- Support both offset and cursor pagination
- Add filtering by status, date range, customer
- Add sorting by multiple fields
- Return proper metadata

### Task 2 — Add File Upload/Download

Create endpoints for order documents:
- `POST /orders/{id}/documents` — Upload document
- `GET /orders/{id}/documents` — List documents
- `GET /orders/{id}/documents/{fileId}` — Download document
- `DELETE /orders/{id}/documents/{fileId}` — Delete document

Implement both local and Azure storage services.

### Task 3 — Add Background Processing

Implement background jobs for:
- Order confirmation emails
- Invoice generation
- Overdue order processing
- Daily summary reports

### Task 4 — Integrate External Service

Create a payment gateway integration:
- Use IHttpClientFactory
- Add retry and circuit breaker policies
- Handle failures gracefully
- Log all external calls

---

## Deliverables

1. **Paginated endpoints** with cursor and offset support
2. **File storage service** with local and Azure implementations
3. **Background job system** using hosted services or Hangfire
4. **External HTTP client** with resilience policies

---

## Resources

### Must-Read
- [IHttpClientFactory Documentation](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory)
- [Background Tasks](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)
- [Azure Blob Storage SDK](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-dotnet)

### Videos
- [Nick Chapsas: HttpClient Best Practices](https://www.youtube.com/watch?v=Z6Y2adsMnAA)
- [Nick Chapsas: Polly v8 Resilience](https://www.youtube.com/watch?v=3truYmkHt5s)
- [Milan Jovanovic: Background Jobs](https://www.youtube.com/watch?v=XPwrS-Qkzm4)
- [Raw Coding: File Upload in ASP.NET Core](https://www.youtube.com/watch?v=VzgIh4_gvJQ)

### Libraries
- [Polly](https://github.com/App-vNext/Polly) — Resilience and transient-fault handling
- [Hangfire](https://www.hangfire.io/) — Background job processing
- [Azure.Storage.Blobs](https://www.nuget.org/packages/Azure.Storage.Blobs/)

---

## Reflection Questions

1. When should you use cursor pagination vs offset pagination?
2. Why is creating HttpClient instances directly problematic?
3. What happens when a circuit breaker opens?
4. When should a job be synchronous vs background?

---

**Next:** [Part 4 — Authentication, Testing, and Deployment](./part-4.md)
