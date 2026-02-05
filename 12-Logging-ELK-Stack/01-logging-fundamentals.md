# Logging Fundamentals

## Why Centralized Logging?

In modern distributed systems, logs are scattered across multiple services, containers, and servers. Centralized logging solves this by aggregating all logs in one place.

```
┌─────────────────────────────────────────────────────────────┐
│              WITHOUT CENTRALIZED LOGGING                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │ Server1 │  │ Server2 │  │ Server3 │  │ Server4 │       │
│  │  logs   │  │  logs   │  │  logs   │  │  logs   │       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│       ↓            ↓            ↓            ↓              │
│    SSH + grep   SSH + grep   SSH + grep   SSH + grep       │
│                                                             │
│  Problems:                                                  │
│  • Manual log hunting across servers                       │
│  • No correlation between services                         │
│  • Logs lost when containers die                           │
│  • No search or analysis capabilities                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              WITH CENTRALIZED LOGGING                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │ Server1 │  │ Server2 │  │ Server3 │  │ Server4 │       │
│  │  logs   │  │  logs   │  │  logs   │  │  logs   │       │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘       │
│       └────────────┴─────┬──────┴────────────┘             │
│                          ▼                                  │
│              ┌─────────────────────┐                       │
│              │   Central Logging   │                       │
│              │   (ELK Stack)       │                       │
│              └─────────────────────┘                       │
│                          │                                  │
│              ┌───────────┴───────────┐                     │
│              ▼                       ▼                      │
│         ┌─────────┐           ┌─────────┐                  │
│         │ Search  │           │ Analyze │                  │
│         │ & Query │           │ & Alert │                  │
│         └─────────┘           └─────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Benefits of Centralized Logging

| Benefit | Description |
|---------|-------------|
| **Single Source** | All logs in one searchable location |
| **Correlation** | Trace requests across services |
| **Persistence** | Logs survive container restarts |
| **Analysis** | Full-text search, aggregations |
| **Alerting** | Trigger alerts based on log patterns |
| **Compliance** | Audit trails, retention policies |
| **Debugging** | Faster incident investigation |

---

## Log Levels

Standard severity levels for categorizing log messages:

```
┌─────────────────────────────────────────────────────────────┐
│                      LOG LEVELS                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level      │ Value │ When to Use                          │
│  ───────────┼───────┼──────────────────────────────────────│
│  TRACE      │  0    │ Very detailed debugging info         │
│  DEBUG      │  1    │ Debugging information                │
│  INFO       │  2    │ Normal operational messages          │
│  WARN       │  3    │ Something unexpected, not an error   │
│  ERROR      │  4    │ Error occurred, operation failed     │
│  FATAL      │  5    │ Critical error, app may crash        │
│                                                             │
│  Production typically uses: INFO and above                  │
│  Development typically uses: DEBUG and above                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Log Level Examples

```
TRACE: Entering method GetUser(id=123)
DEBUG: Cache miss for key 'user:123', fetching from database
INFO:  User 123 logged in successfully
WARN:  API rate limit 80% reached for client 'app-mobile'
ERROR: Failed to process payment: Card declined
FATAL: Database connection pool exhausted, shutting down
```

---

## Structured vs Unstructured Logging

### Unstructured Logging (Bad)

```
2024-01-15 10:23:45 INFO User john@example.com logged in from 192.168.1.1
2024-01-15 10:23:46 ERROR Payment failed for order 12345: Card declined
2024-01-15 10:23:47 INFO Request completed in 250ms
```

**Problems:**
- Hard to parse programmatically
- Inconsistent format
- Difficult to search specific fields
- No standard schema

### Structured Logging (Good)

```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "INFO",
  "message": "User logged in",
  "user": {
    "email": "john@example.com",
    "id": "12345"
  },
  "client": {
    "ip": "192.168.1.1",
    "userAgent": "Mozilla/5.0..."
  },
  "traceId": "abc-123-xyz"
}
```

```json
{
  "timestamp": "2024-01-15T10:23:46.456Z",
  "level": "ERROR",
  "message": "Payment failed",
  "error": {
    "code": "CARD_DECLINED",
    "message": "Card declined"
  },
  "order": {
    "id": "12345",
    "amount": 99.99
  },
  "traceId": "abc-123-xyz"
}
```

**Benefits:**
- Machine-parseable
- Consistent schema
- Easy to search/filter
- Supports aggregations

---

## Elastic Common Schema (ECS)

ECS is a standard field naming convention for structured logs.

```json
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "log.level": "info",
  "message": "User logged in successfully",
  "service": {
    "name": "auth-service",
    "version": "1.2.3",
    "environment": "production"
  },
  "host": {
    "name": "server-01",
    "ip": ["10.0.0.5"]
  },
  "user": {
    "id": "12345",
    "email": "john@example.com"
  },
  "http": {
    "request": {
      "method": "POST",
      "body.bytes": 256
    },
    "response": {
      "status_code": 200,
      "body.bytes": 1024
    }
  },
  "event": {
    "action": "user-login",
    "outcome": "success",
    "duration": 150000000
  },
  "trace": {
    "id": "abc-123-xyz"
  }
}
```

### Common ECS Fields

| Field | Description |
|-------|-------------|
| `@timestamp` | Event timestamp |
| `message` | Human-readable message |
| `log.level` | Log severity |
| `service.name` | Service identifier |
| `host.name` | Hostname |
| `trace.id` | Distributed trace ID |
| `error.message` | Error description |
| `http.request.*` | HTTP request details |
| `user.*` | User information |

---

## Log Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    LOG LIFECYCLE                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. GENERATION                                              │
│     Application writes log entry                            │
│     └─ Use structured logging                               │
│                                                             │
│  2. COLLECTION                                              │
│     Log shipper reads logs (Filebeat, Fluentd)             │
│     └─ Parse, enrich, transform                            │
│                                                             │
│  3. TRANSPORTATION                                          │
│     Ship to central system                                  │
│     └─ Buffer for reliability                              │
│                                                             │
│  4. STORAGE                                                 │
│     Index in Elasticsearch                                  │
│     └─ Optimize for search                                 │
│                                                             │
│  5. ANALYSIS                                                │
│     Search, visualize in Kibana                            │
│     └─ Create dashboards, alerts                           │
│                                                             │
│  6. RETENTION                                               │
│     Archive or delete old logs                              │
│     └─ ILM policies                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## What to Log

### DO Log

```
✓ Application startup and shutdown
✓ Authentication events (login, logout, failures)
✓ Authorization failures
✓ Business transactions (orders, payments)
✓ API requests (method, path, status, duration)
✓ Database queries (slow queries)
✓ External service calls
✓ Errors and exceptions (with stack traces)
✓ Configuration changes
✓ Health check results
```

### DON'T Log

```
✗ Passwords or credentials
✗ Credit card numbers (full)
✗ Social Security Numbers
✗ Personal health information
✗ API keys or secrets
✗ Session tokens
✗ PII without consent/masking
```

### PII Masking Example

```json
// Before masking
{
  "user": {
    "email": "john@example.com",
    "ssn": "123-45-6789",
    "creditCard": "4111111111111111"
  }
}

// After masking
{
  "user": {
    "email": "j***@example.com",
    "ssn": "***-**-6789",
    "creditCard": "************1111"
  }
}
```

---

## Correlation and Tracing

### Correlation ID / Trace ID

Link logs across services for a single request:

```
┌─────────────────────────────────────────────────────────────┐
│                    REQUEST FLOW                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User Request                                               │
│       │                                                     │
│       ▼  traceId: abc-123                                  │
│  ┌─────────────┐                                           │
│  │ API Gateway │ → Log: "Request received" [abc-123]       │
│  └──────┬──────┘                                           │
│         │ traceId: abc-123                                 │
│         ▼                                                   │
│  ┌─────────────┐                                           │
│  │ Order Svc   │ → Log: "Creating order" [abc-123]         │
│  └──────┬──────┘                                           │
│         │ traceId: abc-123                                 │
│         ▼                                                   │
│  ┌─────────────┐                                           │
│  │ Payment Svc │ → Log: "Processing payment" [abc-123]     │
│  └──────┬──────┘                                           │
│         │ traceId: abc-123                                 │
│         ▼                                                   │
│  ┌─────────────┐                                           │
│  │ Database    │ → Log: "Insert order" [abc-123]           │
│  └─────────────┘                                           │
│                                                             │
│  In Kibana: Search for traceId:abc-123 shows all logs      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Log Aggregation Patterns

### Pattern 1: Direct Shipping

```
Application → Elasticsearch
(Simple, but tightly coupled)
```

### Pattern 2: Sidecar/Agent

```
Application → File → Filebeat → Elasticsearch
(Decoupled, recommended for containers)
```

### Pattern 3: With Processing

```
Application → Filebeat → Logstash → Elasticsearch
(Transformation, enrichment)
```

### Pattern 4: Kafka Buffer

```
Application → Filebeat → Kafka → Logstash → Elasticsearch
(High throughput, reliability)
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Centralized Logging** | Aggregate all logs in one place |
| **Log Levels** | TRACE, DEBUG, INFO, WARN, ERROR, FATAL |
| **Structured Logging** | JSON format with consistent fields |
| **ECS** | Elastic Common Schema for standard fields |
| **Correlation ID** | Link logs across services |
| **Log Lifecycle** | Generate → Collect → Store → Analyze → Retain |

### Key Takeaways

```
1. Always use structured logging (JSON)
2. Include trace/correlation IDs
3. Never log sensitive data
4. Use appropriate log levels
5. Follow ECS naming conventions
6. Plan retention from the start
```
