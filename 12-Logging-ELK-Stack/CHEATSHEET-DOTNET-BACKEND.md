# ELK Stack Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET Logging to ELK

### Serilog Setup
```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Elasticsearch
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Process
```

### Basic Configuration
```csharp
// Program.cs
using Serilog;
using Serilog.Sinks.Elasticsearch;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .WriteTo.Console()
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
    {
        AutoRegisterTemplate = true,
        IndexFormat = $"myapp-logs-{DateTime.UtcNow:yyyy-MM}",
        NumberOfShards = 2,
        NumberOfReplicas = 1
    })
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();

var app = builder.Build();
app.UseSerilogRequestLogging();
app.Run();
```

### appsettings.json Configuration
```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "System": "Warning"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "Elasticsearch",
        "Args": {
          "nodeUris": "http://elasticsearch:9200",
          "indexFormat": "myapp-logs-{0:yyyy.MM.dd}",
          "autoRegisterTemplate": true,
          "autoRegisterTemplateVersion": "ESv7"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithEnvironmentName"],
    "Properties": {
      "Application": "MyDotNetApp"
    }
  }
}
```

---

## Structured Logging

### Logging Best Practices
```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public async Task ProcessOrderAsync(Order order)
    {
        // Structured logging with properties
        _logger.LogInformation("Processing order {OrderId} for customer {CustomerId}",
            order.Id, order.CustomerId);

        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = order.Id,
            ["CustomerId"] = order.CustomerId
        }))
        {
            try
            {
                await DoProcessingAsync(order);

                _logger.LogInformation("Order {OrderId} processed successfully. Amount: {Amount}",
                    order.Id, order.TotalAmount);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order {OrderId}. Error: {ErrorMessage}",
                    order.Id, ex.Message);
                throw;
            }
        }
    }
}
```

### Log Output (JSON)
```json
{
  "@timestamp": "2024-01-15T10:30:00.123Z",
  "level": "Information",
  "messageTemplate": "Processing order {OrderId} for customer {CustomerId}",
  "message": "Processing order 12345 for customer C001",
  "fields": {
    "OrderId": 12345,
    "CustomerId": "C001",
    "Application": "MyDotNetApp",
    "Environment": "Production",
    "MachineName": "web-server-01"
  }
}
```

---

## Request Logging

### HTTP Request Logging
```csharp
// Program.cs
app.UseSerilogRequestLogging(options =>
{
    // Customize the message template
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";

    // Attach additional properties
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
        diagnosticContext.Set("ClientIP", httpContext.Connection.RemoteIpAddress?.ToString());

        if (httpContext.User.Identity?.IsAuthenticated == true)
        {
            diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
        }
    };

    // Customize log level based on status code
    options.GetLevel = (httpContext, elapsed, ex) =>
    {
        if (ex != null || httpContext.Response.StatusCode >= 500)
            return LogEventLevel.Error;
        if (httpContext.Response.StatusCode >= 400)
            return LogEventLevel.Warning;
        return LogEventLevel.Information;
    };
});
```

---

## Correlation IDs

### Add Correlation ID
```csharp
// Middleware
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

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers["X-Correlation-ID"] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}

// Program.cs
app.UseMiddleware<CorrelationIdMiddleware>();
```

---

## Exception Logging

### Global Exception Handler
```csharp
public class ExceptionLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionLoggingMiddleware> _logger;

    public ExceptionLoggingMiddleware(RequestDelegate next, ILogger<ExceptionLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Unhandled exception for {Method} {Path}. TraceId: {TraceId}",
                context.Request.Method,
                context.Request.Path,
                context.TraceIdentifier);

            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new
            {
                Error = "An error occurred",
                TraceId = context.TraceIdentifier
            });
        }
    }
}
```

---

## Kibana Queries for .NET

### Common KQL Queries
```
# Find errors
level: "Error" OR level: "Fatal"

# Find by correlation ID
fields.CorrelationId: "abc-123"

# Find by user
fields.UserId: "user123"

# Find slow requests
fields.Elapsed > 1000

# Find specific endpoint errors
fields.RequestPath: "/api/orders*" AND level: "Error"

# Find exceptions by type
fields.ExceptionType: "SqlException"
```

### Useful Kibana Visualizations
1. **Error rate over time** - Line chart of error logs
2. **Top error endpoints** - Bar chart by RequestPath
3. **Response time percentiles** - Line chart of Elapsed field
4. **Error breakdown** - Pie chart by ExceptionType
5. **User activity** - Data table by UserId

---

## Interview Q&A

### Q1: How do you implement logging in a .NET application for ELK?
**A:**
1. Use Serilog with Elasticsearch sink
2. Configure structured JSON logging
3. Add enrichers (environment, machine, correlation ID)
4. Use `UseSerilogRequestLogging()` for HTTP requests
5. Send logs directly to Elasticsearch or via Filebeat

### Q2: What is structured logging and why is it important?
**A:**
```csharp
// Structured (good) - creates searchable fields
_logger.LogInformation("Order {OrderId} processed", orderId);

// Unstructured (bad) - just text
_logger.LogInformation($"Order {orderId} processed");
```
Structured logging creates indexed fields for efficient searching in Elasticsearch.

### Q3: How do you handle log volume in a .NET application?
**A:**
- Set appropriate log levels
- Use sampling for high-frequency events
- Filter out noisy logs (health checks)
- Use async logging sinks
- Batch log entries

```csharp
.WriteTo.Elasticsearch(new ElasticsearchSinkOptions(uri)
{
    BatchAction = ElasticOpType.Create,
    BatchPostingLimit = 50,
    Period = TimeSpan.FromSeconds(2)
})
```

### Q4: How do you correlate logs across microservices?
**A:**
- Use correlation IDs in headers
- Propagate via `X-Correlation-ID` header
- Include in all log entries
- Use OpenTelemetry for distributed tracing

### Q5: How do you log sensitive data safely?
**A:**
```csharp
// Never log sensitive data
_logger.LogInformation("User logged in: {Email}", email); // Bad

// Mask or exclude
_logger.LogInformation("User logged in: {UserId}", userId); // Good

// Use destructuring with exclusions
public record User([property: NotLogged] string Password, string Email);
```

### Q6: How do you search for logs related to a specific request?
**A:**
1. Add correlation ID to all logs
2. Search in Kibana: `fields.CorrelationId: "abc-123"`
3. Sort by timestamp
4. See complete request flow

---

## Docker Setup for Development

```yaml
version: '3.8'

services:
  myapi:
    build: .
    environment:
      - Serilog__WriteTo__1__Args__nodeUris=http://elasticsearch:9200
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
```

---

## Best Practices

1. **Use structured logging** - Properties, not string interpolation
2. **Add correlation IDs** - Track requests across services
3. **Log appropriate levels** - Debug for development, Info for production
4. **Include context** - UserId, RequestPath, TraceId
5. **Don't log sensitive data** - Passwords, tokens, PII
6. **Use async sinks** - Don't block request processing
7. **Set up log rotation** - Index lifecycle management
8. **Monitor log volume** - Alert on unusual spikes
