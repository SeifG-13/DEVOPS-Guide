# Prometheus & Grafana Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET Metrics

### Setup prometheus-net
```bash
dotnet add package prometheus-net.AspNetCore
```

### Basic Configuration
```csharp
// Program.cs
using Prometheus;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Enable HTTP metrics
app.UseHttpMetrics();

// Expose /metrics endpoint
app.MapMetrics();

app.Run();
```

### Custom Metrics
```csharp
using Prometheus;

public class OrderService
{
    // Counter - only increases
    private static readonly Counter OrdersProcessed = Metrics
        .CreateCounter("orders_processed_total", "Total orders processed",
            new CounterConfiguration
            {
                LabelNames = new[] { "status" }
            });

    // Gauge - can increase or decrease
    private static readonly Gauge OrdersInProgress = Metrics
        .CreateGauge("orders_in_progress", "Orders currently being processed");

    // Histogram - measures distribution
    private static readonly Histogram OrderDuration = Metrics
        .CreateHistogram("order_processing_duration_seconds", "Order processing duration",
            new HistogramConfiguration
            {
                Buckets = Histogram.ExponentialBuckets(0.01, 2, 10)
            });

    // Summary - similar to histogram with quantiles
    private static readonly Summary OrderSize = Metrics
        .CreateSummary("order_size_bytes", "Size of orders",
            new SummaryConfiguration
            {
                Objectives = new[]
                {
                    new QuantileEpsilonPair(0.5, 0.05),
                    new QuantileEpsilonPair(0.9, 0.01),
                    new QuantileEpsilonPair(0.99, 0.001)
                }
            });

    public async Task ProcessOrderAsync(Order order)
    {
        OrdersInProgress.Inc();

        using (OrderDuration.NewTimer())
        {
            try
            {
                await DoProcessingAsync(order);
                OrdersProcessed.WithLabels("success").Inc();
            }
            catch
            {
                OrdersProcessed.WithLabels("failed").Inc();
                throw;
            }
            finally
            {
                OrdersInProgress.Dec();
            }
        }

        OrderSize.Observe(order.SizeInBytes);
    }
}
```

---

## HTTP Metrics Configuration

### Detailed HTTP Metrics
```csharp
// Program.cs
app.UseHttpMetrics(options =>
{
    // Include path in labels (be careful with cardinality!)
    options.AddRouteParameter("path");

    // Custom labels
    options.AddCustomLabel("version", context => "1.0.0");

    // Reduce cardinality
    options.ReduceStatusCodeCardinality();
});
```

### Default HTTP Metrics Exposed
```promql
# Request count
http_requests_received_total{method="GET", controller="Users", action="GetAll", code="200"}

# Request duration
http_request_duration_seconds_bucket{le="0.1", method="GET"}
http_request_duration_seconds_sum
http_request_duration_seconds_count

# Requests in progress
http_requests_in_progress
```

---

## Entity Framework Metrics

### Setup EF Core Metrics
```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString);
    options.EnableDetailedErrors();
});

// Custom EF metrics
public class EfMetricsMiddleware
{
    private static readonly Counter DbQueries = Metrics
        .CreateCounter("ef_queries_total", "Total database queries",
            new CounterConfiguration { LabelNames = new[] { "operation" } });

    private static readonly Histogram DbQueryDuration = Metrics
        .CreateHistogram("ef_query_duration_seconds", "Database query duration");

    // Implement middleware logic
}
```

### Database Connection Pool Metrics
```csharp
// Expose SQL Server connection pool metrics
public static class SqlMetrics
{
    private static readonly Gauge ActiveConnections = Metrics
        .CreateGauge("sql_active_connections", "Active SQL connections");

    public static void UpdateMetrics()
    {
        // Get connection pool stats
        // SqlConnection.ClearAllPools() for reset
    }
}
```

---

## Health Checks with Metrics

### Combine Health Checks and Metrics
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(redisConnectionString)
    .ForwardToPrometheus();  // Expose as Prometheus metrics

app.MapHealthChecks("/health");
app.MapMetrics();

// This creates metrics like:
// healthcheck_status{name="DbContext"} 1 (healthy) or 0 (unhealthy)
```

---

## Grafana Dashboards for .NET

### Essential PromQL Queries

#### Request Rate
```promql
sum(rate(http_requests_received_total[5m])) by (controller, action)
```

#### Error Rate
```promql
sum(rate(http_requests_received_total{code=~"5.."}[5m]))
/
sum(rate(http_requests_received_total[5m])) * 100
```

#### Latency (95th percentile)
```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, controller)
)
```

#### Active Requests
```promql
sum(http_requests_in_progress) by (controller)
```

#### Custom Business Metric
```promql
sum(rate(orders_processed_total[5m])) by (status)
```

---

## Complete Monitoring Setup

### Docker Compose
```yaml
version: '3.8'

services:
  myapi:
    build: .
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

### prometheus.yml
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'dotnet-api'
    static_configs:
      - targets: ['myapi:8080']
    metrics_path: /metrics
```

---

## Interview Q&A

### Q1: How do you expose metrics from a .NET application?
**A:**
1. Add `prometheus-net.AspNetCore` package
2. Call `app.UseHttpMetrics()` for HTTP metrics
3. Call `app.MapMetrics()` to expose `/metrics` endpoint
4. Create custom metrics using Counter, Gauge, Histogram

### Q2: What types of metrics should you track in a .NET API?
**A:**
- **RED metrics**: Rate, Errors, Duration
- **Request count** by endpoint and status
- **Latency** percentiles (p50, p95, p99)
- **Error rate** by type
- **Business metrics** (orders, signups)
- **Resource usage** (connections, memory)

### Q3: How do you avoid high cardinality in .NET metrics?
**A:**
- Don't use user IDs, request IDs as labels
- Group similar endpoints (use controller/action, not full path)
- Use `ReduceStatusCodeCardinality()` for HTTP codes
- Limit enum values in labels

```csharp
// Bad - high cardinality
.WithLabels(userId, requestId)

// Good - low cardinality
.WithLabels(statusCategory, endpoint)
```

### Q4: How do you measure request latency properly?
**A:**
Use histograms for latency, not averages:
```csharp
private static readonly Histogram RequestDuration = Metrics
    .CreateHistogram("request_duration_seconds", "Request duration",
        new HistogramConfiguration
        {
            Buckets = new[] { 0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10 }
        });

using (RequestDuration.NewTimer())
{
    await ProcessAsync();
}
```

### Q5: How do you alert on .NET application issues?
**A:**
```yaml
# High error rate
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_received_total{code=~"5.."}[5m]))
    /
    sum(rate(http_requests_received_total[5m])) > 0.01
  for: 5m

# High latency
- alert: HighLatency
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
    ) > 1
  for: 5m
```

### Q6: How do you integrate with Application Insights?
**A:**
Use both for different purposes:
- **Prometheus**: Infrastructure, custom metrics, Kubernetes
- **Application Insights**: APM, distributed tracing, Azure integration

```csharp
// Both can coexist
builder.Services.AddApplicationInsightsTelemetry();
app.UseHttpMetrics();
app.MapMetrics();
```

---

## Kubernetes ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dotnet-api
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapi
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

---

## Best Practices

1. **Use histogram for latency** - Not averages or gauges
2. **Track RED metrics** - Rate, Errors, Duration
3. **Add business metrics** - Domain-specific measurements
4. **Avoid high cardinality** - Limit label values
5. **Use recording rules** - Pre-compute common queries
6. **Set appropriate buckets** - Match your SLOs
7. **Monitor dependencies** - Database, cache, external APIs
8. **Create actionable alerts** - With runbooks
