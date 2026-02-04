# Monitoring Best Practices

## What to Monitor

### The Four Golden Signals

```
┌─────────────────────────────────────────────────────────────┐
│                  FOUR GOLDEN SIGNALS                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. LATENCY                                                 │
│     ─────────                                               │
│     Time to service a request                               │
│     • Track successful vs failed request latency            │
│     • Use percentiles (p50, p90, p99), not averages        │
│                                                             │
│  2. TRAFFIC                                                 │
│     ───────                                                 │
│     Demand on the system                                    │
│     • Requests per second                                   │
│     • Concurrent users                                      │
│                                                             │
│  3. ERRORS                                                  │
│     ──────                                                  │
│     Rate of failed requests                                 │
│     • HTTP 5xx errors                                       │
│     • Application exceptions                                │
│                                                             │
│  4. SATURATION                                              │
│     ──────────                                              │
│     How "full" the service is                               │
│     • CPU, memory, disk usage                               │
│     • Queue depth                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Minimum Metrics Per Service

```yaml
Application Tier:
  - request_count_total        # Traffic
  - request_duration_seconds   # Latency (histogram)
  - request_errors_total       # Errors
  - active_connections         # Saturation

Infrastructure Tier:
  - node_cpu_seconds_total
  - node_memory_MemAvailable_bytes
  - node_filesystem_avail_bytes
  - node_network_receive_bytes_total

Database Tier:
  - queries_total
  - query_duration_seconds
  - connections_active
  - replication_lag_seconds
```

---

## Alerting Best Practices

### Alert on Symptoms, Not Causes

```yaml
# ✓ GOOD - User impact (symptom)
- alert: HighErrorRate
  expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
  annotations:
    summary: "Users are seeing errors"

# ✗ BAD - Internal detail (cause)
- alert: HighCPU
  expr: node_cpu_usage > 80
  # High CPU doesn't necessarily mean users are affected
```

### Use Appropriate 'for' Duration

```yaml
# ✗ Too short - causes flapping
- alert: ServiceDown
  expr: up == 0
  for: 0s  # Alert storms!

# ✓ Appropriate duration
- alert: ServiceDown
  expr: up == 0
  for: 1m  # Avoid transient failures

- alert: HighLatency
  expr: http_request_duration_seconds > 2
  for: 5m  # Sustained issue, not spike
```

### Severity Levels

```yaml
# Critical - Page immediately
severity: critical
# • Service is down
# • Data loss occurring
# • SLA breach imminent

# Warning - Business hours
severity: warning
# • Approaching thresholds
# • Degraded performance
# • Requires investigation

# Info - No notification
severity: info
# • Tracking/trending
# • Dashboard only
```

### Avoid Alert Fatigue

```
┌────────────────────────────────────────────────────────────┐
│                  ALERT FATIGUE PREVENTION                  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  DON'T:                                                    │
│  ✗ Alert on every minor fluctuation                       │
│  ✗ Set thresholds too low                                 │
│  ✗ Page for warnings                                      │
│  ✗ Create alerts without runbooks                         │
│  ✗ Ignore alerts (leads to desensitization)              │
│                                                            │
│  DO:                                                       │
│  ✓ Alert on user-facing impact                           │
│  ✓ Use appropriate thresholds based on history           │
│  ✓ Include actionable runbook URLs                       │
│  ✓ Review and tune alerts regularly                      │
│  ✓ Delete alerts that are never actioned                 │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Dashboard Design

### Dashboard Hierarchy

```
1. Overview Dashboard
   └── High-level health indicators
   └── Service status (up/down)
   └── Key SLI metrics

2. Service Dashboards
   └── Per-service details
   └── Request rate, errors, latency
   └── Dependencies health

3. Infrastructure Dashboards
   └── Node metrics
   └── Container metrics
   └── Network/storage

4. Debug Dashboards
   └── Detailed metrics for troubleshooting
   └── Individual endpoint analysis
```

### Dashboard Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Row 1: Key Stats (Stat panels)                            │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │
│  │ Req/s  │ │ Error% │ │ P99    │ │ CPU %  │ │ Mem %  │  │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Row 2: Trends (Time series)                               │
│  ┌─────────────────────────┐ ┌─────────────────────────┐  │
│  │   Request Rate          │ │   Error Rate            │  │
│  │   ~~~~~/\~~~~           │ │   ____/\___             │  │
│  └─────────────────────────┘ └─────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Row 3: Details (Tables, Heatmaps)                         │
│  ┌─────────────────────────────────────────────────────┐  │
│  │   Top Endpoints by Request Count                     │  │
│  │   /api/users     | 1234 | 0.5%  | 150ms            │  │
│  │   /api/orders    | 567  | 1.2%  | 200ms            │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Use Consistent Colors

```yaml
Colors:
  Green:  Success, healthy, good
  Yellow: Warning, degraded
  Red:    Error, critical, bad
  Blue:   Informational, neutral

Thresholds (example):
  - value: 0,   color: green   # OK
  - value: 70,  color: yellow  # Warning
  - value: 90,  color: red     # Critical
```

---

## Labeling Best Practices

### Good Labels

```promql
# ✓ Bounded, known values
http_requests_total{method="GET", status="200"}
http_requests_total{service="api", environment="prod"}

# ✓ Useful for aggregation
http_requests_total{region="us-east", cluster="prod-1"}
```

### Bad Labels (High Cardinality)

```promql
# ✗ Unbounded - unique per request
http_requests_total{request_id="abc-123"}
http_requests_total{user_id="12345"}

# ✗ PII
http_requests_total{email="user@example.com"}

# ✗ Timestamp (already in data model)
http_requests_total{timestamp="2024-01-15"}
```

### Label Naming Convention

```yaml
# Use snake_case
environment: "production"  # ✓
Environment: "Production"  # ✗

# Be specific
region: "us-east-1"        # ✓
location: "east"           # ✗

# Use consistent names across metrics
job: "api-server"          # Same everywhere
service: "api-server"      # Don't mix
```

---

## Retention and Storage

### Retention Guidelines

```yaml
High-resolution (15s):
  Retention: 15-30 days
  Use: Real-time monitoring, alerting

Downsampled (5m):
  Retention: 90-180 days
  Use: Trend analysis, capacity planning

Aggregated (1h):
  Retention: 1-2 years
  Use: Long-term trends, compliance
```

### Storage Calculation

```
Storage = samples_per_second × bytes_per_sample × retention_seconds

Example:
- 10,000 time series
- 15s scrape interval
- 15 days retention
- ~2 bytes per sample

Storage ≈ 10,000 × (1/15) × 2 × (15 × 24 × 3600)
        ≈ 1.7 GB

With 20% overhead: ~2 GB
```

---

## High Availability

### Prometheus HA Setup

```yaml
# Run multiple Prometheus instances
# Each scrapes the same targets independently
# Use Thanos or Cortex for deduplication and long-term storage

┌─────────────────────────────────────────────────────────────┐
│                    HA ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ Prometheus 1 │  │ Prometheus 2 │  (Both scrape same)    │
│  └──────┬───────┘  └──────┬───────┘                        │
│         │                 │                                 │
│         └────────┬────────┘                                 │
│                  ▼                                          │
│         ┌──────────────┐                                   │
│         │    Thanos    │  (Dedup, long-term storage)       │
│         │    Query     │                                   │
│         └──────────────┘                                   │
│                  │                                          │
│                  ▼                                          │
│         ┌──────────────┐                                   │
│         │   Grafana    │                                   │
│         └──────────────┘                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Alertmanager HA

```bash
# Cluster mode
alertmanager \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-1:9094 \
  --cluster.peer=alertmanager-2:9094
```

---

## Recording Rules

### When to Use Recording Rules

```yaml
# Use recording rules when:
# - Query is expensive (joins, large aggregations)
# - Query is used in alerts AND dashboards
# - Query takes > 1 second to execute

groups:
  - name: request-metrics
    interval: 30s
    rules:
      # Pre-aggregate request rate
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # Pre-calculate error ratio
      - record: job:http_error_ratio:rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))

      # Pre-calculate latency percentiles
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

---

## SLO Monitoring

### Define SLOs

```yaml
# Example SLOs
Availability:
  Target: 99.9%
  Measurement: successful_requests / total_requests
  Window: 30 days

Latency:
  Target: 95% of requests < 200ms
  Measurement: histogram_quantile(0.95, ...)
  Window: Rolling 30 days
```

### SLO Dashboard

```promql
# Error budget remaining
1 - (
  (1 - sum(rate(http_requests_total{status!~"5.."}[30d])) / sum(rate(http_requests_total[30d])))
  /
  (1 - 0.999)  # 99.9% SLO
)

# Burn rate (how fast we're consuming error budget)
(
  1 - sum(rate(http_requests_total{status!~"5.."}[1h])) / sum(rate(http_requests_total[1h]))
)
/
(1 - 0.999)  # Should be < 1 to stay within budget
```

---

## Operational Checklist

### Daily

```
□ Check active alerts
□ Review error rates
□ Verify monitoring is running
```

### Weekly

```
□ Review alert trends
□ Check for noisy alerts
□ Verify backup/retention
□ Review dashboard usage
```

### Monthly

```
□ Capacity planning review
□ SLO performance review
□ Clean up unused dashboards
□ Update runbooks
□ Review alert thresholds
```

---

## Summary Checklist

```
Metrics:
✓ Monitor the Four Golden Signals
✓ Use appropriate metric types
✓ Keep cardinality low
✓ Use recording rules for expensive queries

Alerting:
✓ Alert on symptoms, not causes
✓ Use appropriate 'for' duration
✓ Set correct severity levels
✓ Include runbook URLs
✓ Review alerts regularly

Dashboards:
✓ Use consistent layout
✓ Include variable dropdowns
✓ Use appropriate visualizations
✓ Set meaningful thresholds

Operations:
✓ Plan for HA
✓ Set appropriate retention
✓ Monitor your monitoring
✓ Document runbooks
```

---

## Quick Reference

### Essential PromQL

```promql
# Rate (per-second)
rate(metric[5m])

# Sum by label
sum by (label) (metric)

# Percentile
histogram_quantile(0.99, rate(metric_bucket[5m]))

# Error ratio
sum(rate(errors[5m])) / sum(rate(total[5m]))
```

### Essential Alerts

```yaml
# Service down
up == 0 for 1m

# High error rate
error_rate > 0.05 for 5m

# High latency
p99_latency > 2 for 5m

# Disk almost full
disk_usage > 85% for 5m
```

### Essential Dashboard Panels

```
1. Stat: Current request rate
2. Stat: Current error rate
3. Stat: P99 latency
4. Time series: Request rate over time
5. Time series: Error rate over time
6. Heatmap: Latency distribution
7. Table: Top endpoints
```
