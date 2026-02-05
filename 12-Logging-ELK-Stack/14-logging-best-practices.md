# Logging Best Practices

## What to Log

### Essential Log Events

```
┌────────────────────────────────────────────────────────────┐
│                    WHAT TO LOG                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Application Lifecycle                                     │
│  ├── Startup/shutdown                                     │
│  ├── Configuration loaded                                 │
│  └── Health check results                                 │
│                                                            │
│  Authentication & Authorization                           │
│  ├── Login success/failure                                │
│  ├── Logout events                                        │
│  ├── Permission denied                                    │
│  └── Token refresh                                        │
│                                                            │
│  Business Transactions                                    │
│  ├── Order created/updated/cancelled                     │
│  ├── Payment processed                                    │
│  └── Critical workflow steps                              │
│                                                            │
│  API Requests                                             │
│  ├── Method, path, status code                           │
│  ├── Response time                                        │
│  └── Client IP, user agent                               │
│                                                            │
│  Errors & Exceptions                                      │
│  ├── Stack traces                                         │
│  ├── Error context                                        │
│  └── Recovery actions                                     │
│                                                            │
│  External Service Calls                                   │
│  ├── Request/response times                              │
│  ├── Success/failure                                      │
│  └── Retry attempts                                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### What NOT to Log

```
NEVER LOG:
✗ Passwords or credentials
✗ API keys or secrets
✗ Full credit card numbers
✗ Social Security Numbers
✗ Personal health information
✗ Session tokens (full)
✗ Private encryption keys

MASK OR REDACT:
• Email: j***@example.com
• Credit card: ****-****-****-1234
• Phone: ***-***-5678
• SSN: ***-**-1234
```

---

## Structured Logging

### Log Format Standards

```json
// Good: Structured JSON
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "level": "INFO",
  "message": "Order created successfully",
  "service": "order-service",
  "traceId": "abc-123-xyz",
  "order": {
    "id": "ORD-12345",
    "total": 99.99,
    "itemCount": 3
  },
  "user": {
    "id": "USR-67890"
  }
}

// Bad: Unstructured text
2024-01-15 10:23:45 INFO Order ORD-12345 created for user USR-67890 with total $99.99 (3 items)
```

### Field Naming Conventions

```yaml
Recommendations:
  - Use snake_case or camelCase consistently
  - Follow ECS (Elastic Common Schema) when possible
  - Use meaningful, descriptive names
  - Nest related fields (user.id, user.email)

Common Fields:
  @timestamp    - Event timestamp (ISO 8601)
  level         - Log level (INFO, ERROR, etc.)
  message       - Human-readable message
  service       - Service name
  traceId       - Distributed trace ID
  spanId        - Span ID
  host          - Hostname
  environment   - prod, staging, dev
```

---

## Log Levels

### When to Use Each Level

```
TRACE (Most Verbose)
└── Detailed debugging, method entry/exit
└── Only in development, never in production

DEBUG
└── Diagnostic information
└── Variable values, flow decisions
└── Development and troubleshooting

INFO
└── Normal operations
└── Business events
└── Application lifecycle

WARN
└── Unexpected but handled situations
└── Approaching limits
└── Deprecated feature usage

ERROR
└── Operation failures
└── Exceptions that affect functionality
└── Requires attention

FATAL/CRITICAL
└── Application cannot continue
└── Data corruption risk
└── Immediate action required
```

### Level Guidelines

```csharp
// INFO - Normal operations
logger.LogInformation("User {UserId} logged in", userId);
logger.LogInformation("Order {OrderId} created with total {Total}", orderId, total);

// WARN - Potential issues
logger.LogWarning("Cache miss for user {UserId}, falling back to database", userId);
logger.LogWarning("API rate limit at 80% for client {ClientId}", clientId);

// ERROR - Actual failures
logger.LogError(ex, "Failed to process payment for order {OrderId}", orderId);
logger.LogError("Database connection failed after {RetryCount} retries", retryCount);
```

---

## Correlation and Tracing

### Implement Correlation IDs

```csharp
// Middleware to add correlation ID
public class CorrelationIdMiddleware
{
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
```

### Propagate Across Services

```csharp
// When calling other services
public async Task<OrderResponse> CreateOrderAsync(OrderRequest request)
{
    var correlationId = _httpContextAccessor.HttpContext?.Items["CorrelationId"]?.ToString();

    _httpClient.DefaultRequestHeaders.Add("X-Correlation-ID", correlationId);

    logger.LogInformation("Calling payment service for order {OrderId}", request.OrderId);

    var response = await _httpClient.PostAsJsonAsync("/api/payments", request);

    logger.LogInformation("Payment service responded with status {StatusCode}", response.StatusCode);

    return response;
}
```

---

## Index Lifecycle Management

### Retention Strategy

```
┌────────────────────────────────────────────────────────────┐
│                  INDEX LIFECYCLE                           │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  HOT (0-7 days)                                           │
│  └── Fast storage (SSD)                                   │
│  └── Full replicas                                        │
│  └── Frequent writes                                      │
│                                                            │
│  WARM (7-30 days)                                         │
│  └── Regular storage                                      │
│  └── Read-only                                            │
│  └── Force merge for efficiency                           │
│                                                            │
│  COLD (30-90 days)                                        │
│  └── Cheap storage                                        │
│  └── Minimal resources                                    │
│  └── Infrequent access                                    │
│                                                            │
│  DELETE (90+ days)                                        │
│  └── Archive to S3 if needed                              │
│  └── Delete from cluster                                  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### ILM Policy

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## Performance Optimization

### Elasticsearch Tuning

```yaml
# Index settings for logs
PUT _template/logs-template
{
  "index_patterns": ["logs-*"],
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s",
    "translog.durability": "async",
    "translog.sync_interval": "5s"
  }
}
```

### Bulk Indexing

```
# Batch logs for better performance
- Buffer logs locally
- Send in batches (500-1000 docs)
- Use bulk API
- Compress if network is bottleneck
```

### Cardinality Control

```
# Avoid high-cardinality fields
BAD:  requestId, sessionId, userId (as keywords for aggregation)
GOOD: Use these only for filtering, not aggregation

# Limit unique values
- user_id: OK if limited users
- request_id: BAD for aggregations
```

---

## Security

### Secure ELK Stack

```yaml
# Elasticsearch security
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true

# User roles
POST _security/role/log_reader
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read"]
    }
  ]
}

POST _security/role/log_writer
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["write", "create_index"]
    }
  ]
}
```

### Log Sanitization

```csharp
// Sanitize sensitive data before logging
public static class LogSanitizer
{
    private static readonly string[] SensitiveFields =
        { "password", "token", "secret", "apiKey", "creditCard" };

    public static object Sanitize(object data)
    {
        if (data is IDictionary<string, object> dict)
        {
            var sanitized = new Dictionary<string, object>();
            foreach (var kvp in dict)
            {
                if (SensitiveFields.Any(f =>
                    kvp.Key.Contains(f, StringComparison.OrdinalIgnoreCase)))
                {
                    sanitized[kvp.Key] = "***REDACTED***";
                }
                else
                {
                    sanitized[kvp.Key] = Sanitize(kvp.Value);
                }
            }
            return sanitized;
        }
        return data;
    }
}
```

---

## Monitoring the Logging System

### Key Metrics to Track

```
Elasticsearch:
- Cluster health status
- Index size and growth rate
- Query latency
- Indexing rate
- JVM heap usage

Logstash:
- Events per second
- Pipeline latency
- Errors/dropped events
- Queue depth

Filebeat:
- Harvested files
- Published events
- Errors
- Memory usage
```

### Alerting on Log Pipeline

```json
// Alert: No logs received
PUT _watcher/watch/log_pipeline_health
{
  "trigger": { "schedule": { "interval": "5m" } },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "query": {
            "range": { "@timestamp": { "gte": "now-5m" } }
          }
        }
      }
    }
  },
  "condition": {
    "compare": { "ctx.payload.hits.total.value": { "lt": 100 } }
  },
  "actions": {
    "notify": {
      "webhook": { "url": "https://alerts.example.com" }
    }
  }
}
```

---

## Checklist

### Development

```
□ Use structured logging (JSON)
□ Include correlation IDs
□ Use appropriate log levels
□ Sanitize sensitive data
□ Test logging output
□ Document logging conventions
```

### Production

```
□ Configure appropriate retention
□ Set up ILM policies
□ Enable security (TLS, auth)
□ Monitor cluster health
□ Set up alerts for log pipeline
□ Plan capacity (storage, throughput)
□ Document runbooks for common issues
```

### Regular Maintenance

```
□ Review log volume trends
□ Tune retention policies
□ Update alert thresholds
□ Clean up unused indices
□ Review and optimize queries
□ Update dashboards
```

---

## Summary

| Practice | Description |
|----------|-------------|
| **Structure** | Use JSON, follow ECS |
| **Context** | Include trace IDs, user info |
| **Security** | Sanitize sensitive data |
| **Retention** | ILM policies for lifecycle |
| **Performance** | Bulk indexing, tune settings |
| **Monitoring** | Alert on pipeline health |

### Key Takeaways

```
1. Log what matters, not everything
2. Always use structured logging
3. Include correlation for tracing
4. Never log sensitive data
5. Plan retention from the start
6. Monitor your logging infrastructure
```
