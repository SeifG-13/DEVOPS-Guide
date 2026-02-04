# Monitoring Fundamentals

## Why Monitoring Matters

Monitoring is essential for maintaining reliable, performant systems in production.

```
┌─────────────────────────────────────────────────────────────┐
│                    WHY MONITOR?                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  Detect Issues  │  │  Understand     │                  │
│  │    Early        │  │   Behavior      │                  │
│  │                 │  │                 │                  │
│  │ • Failures      │  │ • Performance   │                  │
│  │ • Degradation   │  │ • Usage patterns│                  │
│  │ • Anomalies     │  │ • Trends        │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  Enable Fast    │  │  Make Data-     │                  │
│  │   Response      │  │  Driven Decisions│                 │
│  │                 │  │                 │                  │
│  │ • Alert on-call │  │ • Capacity plan │                  │
│  │ • Quick debug   │  │ • Optimization  │                  │
│  │ • Reduce MTTR   │  │ • Cost analysis │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Observability vs Monitoring

### Traditional Monitoring
- Predefined dashboards and alerts
- Known unknowns: "Is the server up?"
- Reactive approach

### Observability
- Ability to understand internal state from external outputs
- Unknown unknowns: "Why is this specific request slow?"
- Proactive exploration

```
┌─────────────────────────────────────────────────────────────┐
│              MONITORING vs OBSERVABILITY                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  MONITORING                    OBSERVABILITY                │
│  ──────────                    ─────────────                │
│                                                             │
│  "Is the system working?"      "Why isn't it working?"     │
│                                                             │
│  Dashboard shows               You can explore and         │
│  CPU at 90%                    find root cause             │
│                                                             │
│  Alert: "Error rate high"      Query: "Which users are     │
│                                 affected and why?"          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Three Pillars of Observability

### 1. Metrics
Numeric measurements collected over time.

```
Examples:
- CPU usage: 45%
- Request count: 1,523 requests/sec
- Response time: p99 = 250ms
- Error rate: 0.5%

Characteristics:
✓ Lightweight
✓ Good for aggregation
✓ Long-term storage efficient
✓ Great for alerting
```

### 2. Logs
Timestamped records of discrete events.

```
Examples:
[2024-01-15 10:23:45] INFO  User 12345 logged in
[2024-01-15 10:23:46] ERROR Payment failed: insufficient funds
[2024-01-15 10:23:47] WARN  Database connection slow: 500ms

Characteristics:
✓ Detailed context
✓ Good for debugging
✓ Human-readable
✗ Storage-intensive
```

### 3. Traces
Records of requests flowing through distributed systems.

```
Trace Example:
┌──────────────────────────────────────────────────────────┐
│ Trace ID: abc-123-xyz                                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ [API Gateway]──────────────────────────────── 250ms     │
│    └─[Auth Service]─────────────── 50ms                 │
│    └─[Order Service]────────────────────── 180ms        │
│         └─[Database]──────────── 120ms                  │
│         └─[Cache]────── 5ms                             │
│                                                          │
└──────────────────────────────────────────────────────────┘

Characteristics:
✓ Shows request flow
✓ Identifies bottlenecks
✓ Cross-service correlation
✗ Complex to implement
```

---

## SLI, SLO, and SLA

### Service Level Indicator (SLI)
A quantitative measure of service behavior.

```
Common SLIs:
- Availability: % of successful requests
- Latency: Response time (p50, p90, p99)
- Throughput: Requests per second
- Error rate: % of failed requests
```

### Service Level Objective (SLO)
Target value for an SLI.

```
Examples:
- "99.9% of requests succeed" (availability)
- "p99 latency < 200ms" (latency)
- "Error rate < 0.1%" (errors)
```

### Service Level Agreement (SLA)
Contract with consequences for missing SLOs.

```
Example SLA:
┌────────────────────────────────────────────────────────┐
│ Service Level Agreement                                │
├────────────────────────────────────────────────────────┤
│ Availability: 99.9% monthly                            │
│                                                        │
│ Breach consequences:                                   │
│ - < 99.9%: 10% service credit                         │
│ - < 99.5%: 25% service credit                         │
│ - < 99.0%: 50% service credit                         │
└────────────────────────────────────────────────────────┘
```

### Relationship

```
┌─────────────────────────────────────────────────────────────┐
│                    SLI → SLO → SLA                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SLI (Measure)        SLO (Target)         SLA (Contract)  │
│  ─────────────        ────────────         ──────────────  │
│                                                             │
│  Request success      99.9% success        "We guarantee   │
│  rate = 99.95%        rate target          99.9% uptime    │
│                       ✓ Meeting SLO        or refund"      │
│                                                             │
│  p99 latency          p99 < 200ms          "Response time  │
│  = 180ms              ✓ Meeting SLO        under 200ms"    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## The Four Golden Signals

Google's SRE book recommends monitoring these four signals:

### 1. Latency
Time to service a request.

```
What to track:
- Successful request latency
- Failed request latency (often faster - important to separate!)
- Percentiles: p50, p90, p95, p99
```

### 2. Traffic
Demand on the system.

```
What to track:
- HTTP requests per second
- Transactions per second
- Active connections
- Network I/O
```

### 3. Errors
Rate of failed requests.

```
What to track:
- HTTP 5xx errors
- HTTP 4xx errors (sometimes)
- Application exceptions
- Failed health checks
```

### 4. Saturation
How "full" the service is.

```
What to track:
- CPU utilization
- Memory usage
- Disk I/O
- Queue depth
- Thread pool utilization
```

---

## RED Method (Microservices)

For request-driven services:

```
┌─────────────────────────────────────────────────────────────┐
│                      RED METHOD                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  R - Rate         Requests per second                       │
│                   "How busy is my service?"                 │
│                                                             │
│  E - Errors       Failed requests per second                │
│                   "How often do requests fail?"             │
│                                                             │
│  D - Duration     Time per request (latency)                │
│                   "How long do requests take?"              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## USE Method (Infrastructure)

For infrastructure resources:

```
┌─────────────────────────────────────────────────────────────┐
│                      USE METHOD                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  U - Utilization  % time resource is busy                   │
│                   CPU: 75% utilized                         │
│                                                             │
│  S - Saturation   Degree of queued work                     │
│                   CPU: 5 processes waiting                  │
│                                                             │
│  E - Errors       Count of error events                     │
│                   Disk: 2 I/O errors                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Apply to each resource:
- CPU
- Memory
- Storage
- Network
```

---

## Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              TYPICAL MONITORING STACK                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Data Sources                      │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐   │   │
│  │  │ Apps   │  │ Servers│  │  K8s   │  │  DBs   │   │   │
│  │  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘   │   │
│  └──────┼───────────┼───────────┼───────────┼────────┘   │
│         │           │           │           │             │
│         └───────────┴─────┬─────┴───────────┘             │
│                           ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Collection & Storage                    │   │
│  │  ┌────────────────┐    ┌────────────────┐          │   │
│  │  │   Prometheus   │    │  Log Storage   │          │   │
│  │  │   (Metrics)    │    │  (ELK/Loki)    │          │   │
│  │  └───────┬────────┘    └───────┬────────┘          │   │
│  └──────────┼─────────────────────┼────────────────────┘   │
│             │                     │                        │
│             └──────────┬──────────┘                        │
│                        ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Visualization & Alerting                │   │
│  │  ┌────────────────┐    ┌────────────────┐          │   │
│  │  │    Grafana     │    │  Alertmanager  │          │   │
│  │  │  (Dashboards)  │    │   (Alerts)     │          │   │
│  │  └────────────────┘    └────────────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Metrics Types

### Counter
Always increases (or resets to zero).

```
Examples:
- http_requests_total
- errors_total
- bytes_sent_total

Usage: Use rate() to calculate per-second rate
rate(http_requests_total[5m])
```

### Gauge
Can go up or down.

```
Examples:
- temperature_celsius
- memory_usage_bytes
- active_connections

Usage: Direct value or delta
memory_usage_bytes
delta(memory_usage_bytes[1h])
```

### Histogram
Samples observations into buckets.

```
Examples:
- http_request_duration_seconds
- response_size_bytes

Usage: Calculate percentiles
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Summary
Similar to histogram but calculates quantiles client-side.

```
Examples:
- request_latency_seconds{quantile="0.99"}

Usage: Pre-calculated quantiles (less flexible)
```

---

## Push vs Pull Model

### Pull Model (Prometheus)

```
┌──────────────┐         ┌──────────────┐
│  Prometheus  │◄────────│   Target     │
│              │  scrape │  /metrics    │
└──────────────┘         └──────────────┘

Advantages:
✓ Central control
✓ Easy to debug (curl endpoint)
✓ No target configuration needed
✓ Automatic up/down detection
```

### Push Model (Graphite, InfluxDB)

```
┌──────────────┐         ┌──────────────┐
│   Backend    │◄────────│   Target     │
│              │   push  │              │
└──────────────┘         └──────────────┘

Advantages:
✓ Works behind firewalls
✓ Short-lived jobs
✓ Event-driven metrics
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Metrics** | Numeric measurements over time |
| **Logs** | Timestamped event records |
| **Traces** | Request flow through systems |
| **SLI** | Measurement (e.g., latency) |
| **SLO** | Target (e.g., p99 < 200ms) |
| **SLA** | Contract with consequences |
| **Golden Signals** | Latency, Traffic, Errors, Saturation |
| **RED** | Rate, Errors, Duration |
| **USE** | Utilization, Saturation, Errors |

### Key Takeaways

```
1. Monitor the Four Golden Signals at minimum
2. Set SLOs before building dashboards
3. Use RED for services, USE for infrastructure
4. Combine metrics, logs, and traces for full observability
5. Alert on symptoms (user impact), not causes
```
