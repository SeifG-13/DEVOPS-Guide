# Monitoring .NET 10 Applications

## Overview

Instrumenting .NET 10 applications with Prometheus metrics for comprehensive monitoring.

```
┌─────────────────────────────────────────────────────────────┐
│              .NET 10 METRICS ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              .NET 10 Application                     │   │
│  │  ┌───────────────────────────────────────────────┐ │   │
│  │  │           prometheus-net                       │ │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐      │ │   │
│  │  │  │ Counters│  │ Gauges  │  │Histograms│     │ │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘      │ │   │
│  │  └───────────────────────────────────────────────┘ │   │
│  │                        │                            │   │
│  │                        ▼                            │   │
│  │              /metrics endpoint                      │   │
│  └─────────────────────────┬───────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Prometheus                             │   │
│  │         (scrapes /metrics)                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Install prometheus-net

### NuGet Packages

```bash
# Core metrics library
dotnet add package prometheus-net

# ASP.NET Core integration
dotnet add package prometheus-net.AspNetCore

# Health check integration (optional)
dotnet add package prometheus-net.AspNetCore.HealthChecks

# gRPC metrics (if using gRPC)
dotnet add package prometheus-net.AspNetCore.Grpc

# System diagnostics metrics
dotnet add package prometheus-net.SystemMetrics
```

---

## Basic Setup

### Program.cs

```csharp
using Prometheus;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Use Prometheus HTTP metrics middleware
// Must be early in pipeline to capture all requests
app.UseHttpMetrics();

// Map endpoints
app.UseSwagger();
app.UseSwaggerUI();
app.UseRouting();
app.UseAuthorization();
app.MapControllers();

// Expose /metrics endpoint
app.MapMetrics();

app.Run();
```

### Default Metrics Exposed

```promql
# HTTP request metrics (from UseHttpMetrics)
http_requests_received_total{code="200", method="GET", controller="WeatherForecast", action="Get"}
http_request_duration_seconds_bucket{le="0.1", method="GET", controller="WeatherForecast"}
http_requests_in_progress{method="GET", controller="WeatherForecast"}

# .NET runtime metrics
dotnet_collection_count_total{generation="0"}
dotnet_total_memory_bytes
process_cpu_seconds_total
process_working_set_bytes
process_open_handles
process_num_threads
```

---

## Custom Metrics

### Define Custom Metrics

```csharp
// Metrics/AppMetrics.cs
using Prometheus;

public static class AppMetrics
{
    // Counter - always increases
    public static readonly Counter OrdersCreated = Metrics.CreateCounter(
        "orders_created_total",
        "Total number of orders created",
        new CounterConfiguration
        {
            LabelNames = new[] { "payment_method", "status" }
        });

    // Gauge - can increase or decrease
    public static readonly Gauge ActiveUsers = Metrics.CreateGauge(
        "active_users_current",
        "Number of currently active users");

    public static readonly Gauge QueueDepth = Metrics.CreateGauge(
        "queue_depth",
        "Number of items in queue",
        new GaugeConfiguration
        {
            LabelNames = new[] { "queue_name" }
        });

    // Histogram - measures distribution
    public static readonly Histogram OrderProcessingDuration = Metrics.CreateHistogram(
        "order_processing_duration_seconds",
        "Time to process an order",
        new HistogramConfiguration
        {
            LabelNames = new[] { "order_type" },
            Buckets = new[] { 0.1, 0.25, 0.5, 1, 2.5, 5, 10 }
        });

    // Summary - similar to histogram but calculates quantiles client-side
    public static readonly Summary PaymentDuration = Metrics.CreateSummary(
        "payment_duration_seconds",
        "Payment processing duration",
        new SummaryConfiguration
        {
            LabelNames = new[] { "provider" },
            Objectives = new[]
            {
                new QuantileEpsilonPair(0.5, 0.05),
                new QuantileEpsilonPair(0.9, 0.01),
                new QuantileEpsilonPair(0.99, 0.001)
            }
        });
}
```

### Use Custom Metrics

```csharp
// Controllers/OrdersController.cs
using Prometheus;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(IOrderService orderService, ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        // Time the operation with histogram
        using (AppMetrics.OrderProcessingDuration.WithLabels(request.Type).NewTimer())
        {
            var order = await _orderService.CreateAsync(request);

            // Increment counter
            AppMetrics.OrdersCreated
                .WithLabels(request.PaymentMethod, "success")
                .Inc();

            _logger.LogInformation("Order {OrderId} created", order.Id);

            return Ok(order);
        }
    }

    [HttpGet("active-users")]
    public async Task<IActionResult> GetActiveUsers()
    {
        var count = await _orderService.GetActiveUserCountAsync();

        // Update gauge
        AppMetrics.ActiveUsers.Set(count);

        return Ok(count);
    }
}
```

### Background Service Metrics

```csharp
// Services/QueueProcessorService.cs
public class QueueProcessorService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var queueLength = await GetQueueLength();
            AppMetrics.QueueDepth.WithLabels("orders").Set(queueLength);

            var item = await DequeueItem();
            if (item != null)
            {
                using (AppMetrics.OrderProcessingDuration.WithLabels("queued").NewTimer())
                {
                    await ProcessItem(item);
                }
            }

            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

---

## HTTP Metrics Configuration

### Customize HTTP Metrics

```csharp
// Program.cs
app.UseHttpMetrics(options =>
{
    // Include path in labels (be careful with cardinality!)
    options.AddCustomLabel("path", context => context.Request.Path.Value);

    // Reduce cardinality by grouping
    options.ReduceStatusCodeCardinality = true;

    // Add route template instead of actual path
    options.AddRouteParameter(parameter => parameter);

    // Configure histogram buckets
    options.RequestDuration.Histogram.Buckets = new[] { 0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10 };

    // Exclude certain paths
    options.RequestFilter = context =>
        !context.Request.Path.StartsWithSegments("/health") &&
        !context.Request.Path.StartsWithSegments("/metrics");
});
```

### Custom Labels

```csharp
// Add tenant/customer ID (be careful with cardinality!)
app.UseHttpMetrics(options =>
{
    options.AddCustomLabel("tenant", context =>
    {
        // Extract tenant from header
        return context.Request.Headers["X-Tenant-ID"].FirstOrDefault() ?? "unknown";
    });
});
```

---

## Health Check Metrics

### Setup Health Checks

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "db", "sql" })
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: new[] { "cache" })
    .AddUrlGroup(
        new Uri("https://external-api.com/health"),
        name: "external-api");

var app = builder.Build();

// Expose health check metrics
app.UseHealthChecksPrometheusExporter("/health/metrics");

// Or combine with main metrics endpoint
app.MapMetrics();
app.UseHealthChecks("/health");
```

### Health Check Metrics Output

```promql
# Health check status (1 = healthy, 0 = unhealthy)
healthcheck_status{name="database"} 1
healthcheck_status{name="redis"} 1
healthcheck_status{name="external-api"} 0

# Health check duration
healthcheck_duration_seconds{name="database"} 0.015
```

---

## Minimal API Metrics

```csharp
// Program.cs for Minimal APIs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Define custom metrics
var requestCounter = Metrics.CreateCounter(
    "custom_api_requests_total",
    "Custom API request count",
    new CounterConfiguration { LabelNames = new[] { "endpoint" } });

var requestDuration = Metrics.CreateHistogram(
    "custom_api_request_duration_seconds",
    "Custom API request duration");

app.UseHttpMetrics();

app.MapGet("/api/items", async () =>
{
    using (requestDuration.NewTimer())
    {
        requestCounter.WithLabels("/api/items").Inc();
        var items = await GetItemsAsync();
        return Results.Ok(items);
    }
});

app.MapMetrics();
app.Run();
```

---

## Middleware for Custom Metrics

```csharp
// Middleware/MetricsMiddleware.cs
public class MetricsMiddleware
{
    private readonly RequestDelegate _next;

    private static readonly Counter RequestsTotal = Metrics.CreateCounter(
        "myapp_http_requests_total",
        "Total HTTP requests",
        new CounterConfiguration
        {
            LabelNames = new[] { "method", "endpoint", "status" }
        });

    private static readonly Histogram RequestDuration = Metrics.CreateHistogram(
        "myapp_http_request_duration_seconds",
        "HTTP request duration",
        new HistogramConfiguration
        {
            LabelNames = new[] { "method", "endpoint" },
            Buckets = Histogram.ExponentialBuckets(0.001, 2, 15)
        });

    public MetricsMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var path = context.Request.Path.Value ?? "unknown";
        var method = context.Request.Method;

        using (RequestDuration.WithLabels(method, path).NewTimer())
        {
            await _next(context);

            var statusCode = context.Response.StatusCode.ToString();
            RequestsTotal.WithLabels(method, path, statusCode).Inc();
        }
    }
}

// Program.cs
app.UseMiddleware<MetricsMiddleware>();
```

---

## Entity Framework Metrics

```csharp
// Extensions/EFCoreMetrics.cs
public static class EFCoreMetrics
{
    public static readonly Counter QueriesExecuted = Metrics.CreateCounter(
        "efcore_queries_total",
        "Total EF Core queries executed",
        new CounterConfiguration { LabelNames = new[] { "db_context", "operation" } });

    public static readonly Histogram QueryDuration = Metrics.CreateHistogram(
        "efcore_query_duration_seconds",
        "EF Core query duration",
        new HistogramConfiguration { LabelNames = new[] { "db_context" } });
}

// DbContext with interceptor
public class AppDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.AddInterceptors(new MetricsInterceptor());
    }
}

public class MetricsInterceptor : DbCommandInterceptor
{
    public override DbDataReader ReaderExecuted(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result)
    {
        EFCoreMetrics.QueriesExecuted.WithLabels("AppDbContext", "read").Inc();
        EFCoreMetrics.QueryDuration
            .WithLabels("AppDbContext")
            .Observe(eventData.Duration.TotalSeconds);

        return base.ReaderExecuted(command, eventData, result);
    }
}
```

---

## Prometheus Configuration

### Scrape .NET Application

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'dotnet-api'
    scrape_interval: 15s
    static_configs:
      - targets: ['myapp-api:8080']
    metrics_path: /metrics

  # Kubernetes pod discovery
  - job_name: 'kubernetes-pods-dotnet'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: myapp-api
```

---

## Kubernetes Deployment with Metrics

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: myapp-api
          image: myapp-api:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
```

---

## Common Queries

```promql
# Request rate
rate(http_requests_received_total{job="dotnet-api"}[5m])

# Error rate
sum(rate(http_requests_received_total{job="dotnet-api",code=~"5.."}[5m]))
/
sum(rate(http_requests_received_total{job="dotnet-api"}[5m]))

# P99 latency
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job="dotnet-api"}[5m])))

# Memory usage
process_working_set_bytes{job="dotnet-api"}

# GC collections
rate(dotnet_collection_count_total{job="dotnet-api"}[5m])

# Thread pool
dotnet_threadpool_num_threads{job="dotnet-api"}

# Custom metrics
rate(orders_created_total[5m])
active_users_current
histogram_quantile(0.95, rate(order_processing_duration_seconds_bucket[5m]))
```

---

## Summary

| Package | Purpose |
|---------|---------|
| `prometheus-net` | Core metrics library |
| `prometheus-net.AspNetCore` | HTTP metrics middleware |
| `prometheus-net.AspNetCore.HealthChecks` | Health check metrics |

### Quick Start

```csharp
// Program.cs
using Prometheus;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseHttpMetrics();  // HTTP metrics
app.MapMetrics();      // /metrics endpoint

app.Run();
```

### Metric Types

| Type | Use Case | Example |
|------|----------|---------|
| Counter | Cumulative count | requests_total |
| Gauge | Current value | active_connections |
| Histogram | Distribution | request_duration |
| Summary | Pre-calculated quantiles | payment_duration |
