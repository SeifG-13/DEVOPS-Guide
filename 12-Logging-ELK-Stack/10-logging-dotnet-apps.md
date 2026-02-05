# Logging .NET 10 Applications to ELK

## Overview

Configure .NET 10 applications to send structured logs to Elasticsearch using Serilog.

```
┌─────────────────────────────────────────────────────────────┐
│              .NET LOGGING TO ELK                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Option 1: Direct to Elasticsearch                          │
│  ┌─────────────┐        ┌─────────────────┐               │
│  │ .NET App    │───────▶│  Elasticsearch  │               │
│  │ (Serilog)   │        └─────────────────┘               │
│  └─────────────┘                                           │
│                                                             │
│  Option 2: Via Filebeat (Recommended)                      │
│  ┌─────────────┐   ┌──────────┐   ┌───────────────────┐  │
│  │ .NET App    │──▶│ Filebeat │──▶│  Elasticsearch    │  │
│  │ (JSON file) │   └──────────┘   └───────────────────┘  │
│  └─────────────┘                                           │
│                                                             │
│  Option 3: Via Logstash                                    │
│  ┌─────────────┐   ┌──────────┐   ┌──────┐  ┌─────────┐ │
│  │ .NET App    │──▶│ Filebeat │──▶│Logst.│─▶│   ES    │ │
│  └─────────────┘   └──────────┘   └──────┘  └─────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Install Serilog Packages

```bash
# Core packages
dotnet add package Serilog
dotnet add package Serilog.AspNetCore

# Sinks (outputs)
dotnet add package Serilog.Sinks.Elasticsearch
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File

# Enrichers
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
dotnet add package Serilog.Enrichers.Process

# Formatting
dotnet add package Serilog.Formatting.Elasticsearch
dotnet add package Serilog.Formatting.Compact
```

---

## Basic Serilog Setup

### Program.cs

```csharp
using Serilog;
using Serilog.Events;
using Serilog.Formatting.Elasticsearch;
using Serilog.Sinks.Elasticsearch;

var builder = WebApplication.CreateBuilder(args);

// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console()
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
    {
        IndexFormat = $"logs-myapp-{builder.Environment.EnvironmentName.ToLower()}-{DateTime.UtcNow:yyyy.MM}",
        AutoRegisterTemplate = true,
        AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv8,
        TypeName = null,
        BatchAction = ElasticOpType.Create,
        ModifyConnectionSettings = x => x.BasicAuthentication("elastic", "changeme"),
        FailureCallback = (e, ex) => Console.WriteLine($"Log failed: {ex?.Message}"),
        EmitEventFailure = EmitEventFailureHandling.WriteToSelfLog |
                           EmitEventFailureHandling.RaiseCallback
    })
    .CreateLogger();

builder.Host.UseSerilog();

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Use Serilog request logging
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
        diagnosticContext.Set("ClientIP", httpContext.Connection.RemoteIpAddress?.ToString());
    };
});

app.UseSwagger();
app.UseSwaggerUI();
app.MapControllers();

try
{
    Log.Information("Starting application");
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

---

## Configuration via appsettings.json

### appsettings.json

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.Elasticsearch"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "Elasticsearch",
        "Args": {
          "nodeUris": "http://elasticsearch:9200",
          "indexFormat": "logs-myapp-{0:yyyy.MM.dd}",
          "autoRegisterTemplate": true,
          "autoRegisterTemplateVersion": "ESv8"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithEnvironmentName", "WithThreadId"],
    "Properties": {
      "Application": "MyApp.API",
      "Environment": "Production"
    }
  }
}
```

### Program.cs (Configuration-based)

```csharp
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Configure Serilog from appsettings.json
builder.Host.UseSerilog((context, configuration) =>
{
    configuration.ReadFrom.Configuration(context.Configuration);
});

// ... rest of setup
```

---

## Structured Logging Best Practices

### Use Message Templates

```csharp
// GOOD - Structured logging with named properties
Log.Information("User {UserId} logged in from {IpAddress}", userId, ipAddress);

// BAD - String interpolation loses structure
Log.Information($"User {userId} logged in from {ipAddress}");
```

### Log Context / Scopes

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> ProcessOrderAsync(int orderId)
    {
        using (LogContext.PushProperty("OrderId", orderId))
        {
            _logger.LogInformation("Starting order processing");

            // All logs within this scope include OrderId
            var order = await GetOrderAsync(orderId);
            _logger.LogInformation("Order retrieved with {ItemCount} items", order.Items.Count);

            await ValidateOrderAsync(order);
            _logger.LogInformation("Order validated");

            await ChargePaymentAsync(order);
            _logger.LogInformation("Payment processed for {Amount:C}", order.Total);

            return order;
        }
    }
}
```

### Exception Logging

```csharp
try
{
    await ProcessPaymentAsync(order);
}
catch (PaymentException ex)
{
    // Include exception as first parameter
    _logger.LogError(ex, "Payment failed for order {OrderId} with amount {Amount}",
        order.Id, order.Total);
    throw;
}
```

### Request Correlation

```csharp
// Middleware to add correlation ID
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Response.Headers["X-Correlation-ID"] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}

// Register in Program.cs
app.UseMiddleware<CorrelationIdMiddleware>();
```

---

## Elastic Common Schema (ECS)

### Install ECS Package

```bash
dotnet add package Elastic.CommonSchema.Serilog
```

### Configure ECS Formatting

```csharp
using Elastic.CommonSchema.Serilog;
using Serilog;

Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
    {
        IndexFormat = "logs-myapp-{0:yyyy.MM.dd}",
        CustomFormatter = new EcsTextFormatter()
    })
    .CreateLogger();
```

### ECS Output Format

```json
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "log.level": "Information",
  "message": "User 12345 logged in from 192.168.1.1",
  "ecs": {
    "version": "1.6.0"
  },
  "event": {
    "created": "2024-01-15T10:23:45.123Z",
    "timezone": "UTC"
  },
  "host": {
    "name": "server-01"
  },
  "log": {
    "logger": "MyApp.Services.AuthService"
  },
  "process": {
    "thread": {
      "id": 15
    }
  },
  "service": {
    "name": "MyApp.API",
    "environment": "production"
  },
  "user": {
    "id": "12345"
  },
  "client": {
    "ip": "192.168.1.1"
  }
}
```

---

## Logging to File (for Filebeat)

### Write JSON Logs to File

```csharp
using Serilog;
using Serilog.Formatting.Compact;
using Serilog.Formatting.Elasticsearch;

Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "MyApp.API")
    .Enrich.WithProperty("Environment", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT"))
    .WriteTo.Console()
    .WriteTo.File(
        new ElasticsearchJsonFormatter(),
        "/var/log/myapp/app.json",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 7,
        fileSizeLimitBytes: 100_000_000,  // 100MB
        rollOnFileSizeLimit: true
    )
    .CreateLogger();
```

### Filebeat Configuration

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.json
    json.keys_under_root: true
    json.add_error_key: true

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "logs-myapp-%{+yyyy.MM.dd}"

setup.kibana:
  host: "http://kibana:5601"
```

---

## Controller Logging Example

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly ILogger<OrdersController> _logger;
    private readonly IOrderService _orderService;

    public OrdersController(ILogger<OrdersController> logger, IOrderService orderService)
    {
        _logger = logger;
        _orderService = orderService;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        _logger.LogInformation("Creating order for customer {CustomerId} with {ItemCount} items",
            request.CustomerId, request.Items.Count);

        try
        {
            var order = await _orderService.CreateAsync(request);

            _logger.LogInformation("Order {OrderId} created successfully with total {Total:C}",
                order.Id, order.Total);

            return Ok(order);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation failed for order request from customer {CustomerId}",
                request.CustomerId);
            return BadRequest(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create order for customer {CustomerId}",
                request.CustomerId);
            return StatusCode(500, "An error occurred");
        }
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(int id)
    {
        using (_logger.BeginScope(new Dictionary<string, object> { ["OrderId"] = id }))
        {
            _logger.LogDebug("Retrieving order");

            var order = await _orderService.GetByIdAsync(id);

            if (order == null)
            {
                _logger.LogWarning("Order not found");
                return NotFound();
            }

            _logger.LogDebug("Order retrieved successfully");
            return Ok(order);
        }
    }
}
```

---

## Health Check Logging

```csharp
// Add health check with logging
builder.Services.AddHealthChecks()
    .AddElasticsearch(
        builder.Configuration["Elasticsearch:Url"]!,
        name: "elasticsearch",
        tags: new[] { "ready" });

// Log health check results
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        var result = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                duration = e.Value.Duration.TotalMilliseconds
            })
        };

        Log.Information("Health check completed with status {Status}", report.Status);

        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(result);
    }
});
```

---

## Docker Configuration

### Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApp.API.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Create log directory
RUN mkdir -p /var/log/myapp

ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

### Docker Compose with ELK

```yaml
version: '3.8'

services:
  myapp-api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - Elasticsearch__Url=http://elasticsearch:9200
    volumes:
      - app-logs:/var/log/myapp
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - app-logs:/var/log/myapp:ro
    depends_on:
      - elasticsearch

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"

volumes:
  app-logs:
```

---

## Summary

| Package | Purpose |
|---------|---------|
| `Serilog.AspNetCore` | ASP.NET Core integration |
| `Serilog.Sinks.Elasticsearch` | Direct ES output |
| `Serilog.Sinks.File` | File output (for Filebeat) |
| `Elastic.CommonSchema.Serilog` | ECS format |

### Best Practices

```
✓ Use structured logging (message templates)
✓ Include correlation IDs
✓ Log exceptions with context
✓ Use ECS format for consistency
✓ Don't log sensitive data
✓ Use appropriate log levels
```
