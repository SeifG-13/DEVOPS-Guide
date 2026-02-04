# PromQL Fundamentals

## What is PromQL?

PromQL (Prometheus Query Language) is a functional query language for selecting and aggregating time series data.

```
┌─────────────────────────────────────────────────────────────┐
│                      PROMQL                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PromQL allows you to:                                      │
│  • Select time series by metric name and labels            │
│  • Aggregate across dimensions                              │
│  • Apply mathematical operations                            │
│  • Calculate rates, increases, histograms                   │
│  • Join multiple metrics                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Types

### Instant Vector
Current value of each time series.

```promql
# Returns current value for each matching time series
http_requests_total

# Result:
# http_requests_total{method="GET", status="200"} 1234
# http_requests_total{method="POST", status="200"} 567
```

### Range Vector
Values over a time range.

```promql
# Returns values for the last 5 minutes
http_requests_total[5m]

# Result:
# http_requests_total{method="GET"}
#   1200 @1705312800
#   1215 @1705312815
#   1234 @1705312830
```

### Scalar
Single numeric value.

```promql
# Returns a scalar
count(http_requests_total)  # e.g., 10
```

### String
Single string value (rarely used).

---

## Selectors

### Basic Selectors

```promql
# By metric name
http_requests_total

# With exact label match
http_requests_total{method="GET"}

# Multiple labels
http_requests_total{method="GET", status="200"}
```

### Label Matchers

```promql
# Exact match (=)
http_requests_total{method="GET"}

# Not equal (!=)
http_requests_total{method!="OPTIONS"}

# Regex match (=~)
http_requests_total{status=~"2.."}        # 2xx status codes
http_requests_total{method=~"GET|POST"}   # GET or POST

# Negative regex (!~)
http_requests_total{path!~"/health.*"}    # Exclude health endpoints
```

### Time Range Selectors

```promql
# Last 5 minutes
http_requests_total[5m]

# Last 1 hour
http_requests_total[1h]

# Offset - look back in time
http_requests_total offset 1h              # Value from 1 hour ago
http_requests_total[5m] offset 1d          # 5min range from yesterday
```

### Time Units

```
ms  - milliseconds
s   - seconds
m   - minutes
h   - hours
d   - days
w   - weeks
y   - years
```

---

## Operators

### Arithmetic Operators

```promql
# Add
node_memory_MemTotal_bytes + node_memory_Buffers_bytes

# Subtract
node_memory_MemTotal_bytes - node_memory_MemFree_bytes

# Multiply
rate(http_requests_total[5m]) * 60    # Convert to per-minute

# Divide
node_memory_MemFree_bytes / node_memory_MemTotal_bytes * 100  # % free

# Modulo
hour() % 12

# Power
2 ^ 10
```

### Comparison Operators

```promql
# Greater than
http_requests_total > 1000

# Less than
node_memory_MemFree_bytes < 1000000000

# Equal
http_requests_total == 100

# Not equal
http_requests_total != 0

# Greater or equal
cpu_usage >= 80

# Less or equal
response_time <= 0.5

# With bool modifier (return 0 or 1 instead of filtering)
http_requests_total > bool 1000
```

### Logical Operators

```promql
# AND - intersection (both sides must have same labels)
http_requests_total{status="200"} and http_requests_total{method="GET"}

# OR - union
http_requests_total{status="500"} or http_requests_total{status="503"}

# UNLESS - complement (left side unless right side matches)
http_requests_total unless http_requests_total{method="OPTIONS"}
```

### Vector Matching

```promql
# One-to-one matching (default)
method_requests / method_total

# Ignoring labels
method_requests / ignoring(status) method_total

# Matching on specific labels
method_requests / on(method) method_total

# Many-to-one / one-to-many
method_requests * on(method) group_left(team) method_metadata
method_requests * on(method) group_right method_metadata
```

---

## Functions

### Rate Functions

```promql
# rate() - per-second average rate of increase
rate(http_requests_total[5m])

# irate() - instant rate (last two samples)
irate(http_requests_total[5m])

# increase() - total increase over range
increase(http_requests_total[1h])

# Use rate() for:
# - Alerting
# - Graphing (smooth lines)

# Use irate() for:
# - Quick changes detection
# - Volatile metrics
```

### Aggregation Functions

```promql
# Sum all values
sum(http_requests_total)

# Sum by label
sum by (method) (http_requests_total)

# Sum excluding label
sum without (instance) (http_requests_total)

# Count number of time series
count(http_requests_total)

# Average
avg(node_cpu_seconds_total)

# Min/Max
min(node_memory_MemFree_bytes)
max(response_time_seconds)

# Standard deviation
stddev(response_time_seconds)

# Quantile (0-1)
quantile(0.95, response_time_seconds)

# Top/Bottom K
topk(5, http_requests_total)
bottomk(3, node_memory_MemFree_bytes)

# Count values matching condition
count_values("version", build_info)
```

### Aggregation with `by` and `without`

```promql
# Group by specific labels
sum by (method, status) (rate(http_requests_total[5m]))

# Group by all except specific labels
sum without (instance, pod) (rate(http_requests_total[5m]))

# Multiple aggregations
avg by (job) (
  rate(http_request_duration_seconds_sum[5m])
  /
  rate(http_request_duration_seconds_count[5m])
)
```

### Math Functions

```promql
# Absolute value
abs(temperature_celsius)

# Ceiling/Floor
ceil(request_duration)
floor(request_duration)

# Natural log / log base 2/10
ln(value)
log2(value)
log10(value)

# Exponential
exp(value)

# Square root
sqrt(value)

# Round to nearest integer
round(value)
round(value, 0.5)  # Round to 0.5

# Clamp values
clamp(cpu_usage, 0, 100)
clamp_min(value, 0)
clamp_max(value, 100)
```

### Time Functions

```promql
# Current Unix timestamp
time()

# Day of month (1-31)
day_of_month()

# Day of week (0=Sunday, 6=Saturday)
day_of_week()

# Hour (0-23)
hour()

# Minute (0-59)
minute()

# Month (1-12)
month()

# Year
year()

# Timestamp of each sample
timestamp(http_requests_total)
```

### Label Functions

```promql
# Extract label value
label_replace(
  http_requests_total,
  "host",                    # new label
  "$1",                      # replacement
  "instance",                # source label
  "([^:]+):\\d+"            # regex
)

# Join labels
label_join(
  http_requests_total,
  "full_path",               # new label
  "/",                       # separator
  "method", "path"           # labels to join
)
```

### Histogram Functions

```promql
# Calculate quantile from histogram
histogram_quantile(
  0.99,                                           # 99th percentile
  rate(http_request_duration_seconds_bucket[5m])  # histogram buckets
)

# 50th, 90th, 99th percentiles
histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# By label
histogram_quantile(
  0.99,
  sum by (le, method) (rate(http_request_duration_seconds_bucket[5m]))
)
```

### Prediction Functions

```promql
# Predict value using linear regression
predict_linear(node_filesystem_avail_bytes[1h], 4*3600)
# Predict disk space in 4 hours

# Derivative (rate of change)
deriv(node_memory_MemFree_bytes[1h])

# Changes in range
changes(up[1h])

# Resets (counter resets)
resets(http_requests_total[1h])
```

### Missing Data Functions

```promql
# Check if vector is empty
absent(up{job="myapp"})
# Returns 1 if no matching series exist

# Fill with value if absent
absent_over_time(up{job="myapp"}[5m])
# Returns 1 if no samples in range
```

---

## Common Query Patterns

### Request Rate

```promql
# Requests per second
rate(http_requests_total[5m])

# Requests per second by endpoint
sum by (path) (rate(http_requests_total[5m]))

# Total requests per minute
sum(rate(http_requests_total[5m])) * 60
```

### Error Rate

```promql
# Error percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Error rate by service
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (service) (rate(http_requests_total[5m]))
```

### Latency

```promql
# Average latency
rate(http_request_duration_seconds_sum[5m])
/
rate(http_request_duration_seconds_count[5m])

# 99th percentile latency
histogram_quantile(
  0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# Latency by endpoint
histogram_quantile(
  0.99,
  sum by (le, path) (rate(http_request_duration_seconds_bucket[5m]))
)
```

### Resource Usage

```promql
# CPU usage percentage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage percentage
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

### Availability

```promql
# Service uptime
avg_over_time(up{job="myapp"}[24h]) * 100

# Availability SLO
sum(rate(http_requests_total{status!~"5.."}[30d]))
/
sum(rate(http_requests_total[30d]))
* 100
```

### Saturation

```promql
# Thread pool saturation
thread_pool_active_threads / thread_pool_max_threads

# Queue depth
rate(queue_items_total[5m]) - rate(queue_items_processed_total[5m])
```

---

## Subqueries

```promql
# Calculate max of 5-minute rates over 1 hour
max_over_time(
  rate(http_requests_total[5m])[1h:1m]  # 1h range, 1m resolution
)

# Average rate over time
avg_over_time(
  rate(http_requests_total[5m])[24h:5m]
)

# Detect spikes
max_over_time(rate(http_requests_total[5m])[1h:])
>
avg_over_time(rate(http_requests_total[5m])[1h:]) * 2
```

---

## Recording Rules

Pre-compute expensive queries:

```yaml
# recording-rules.yml
groups:
  - name: request-rates
    interval: 30s
    rules:
      # Record request rate
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # Record error rate
      - record: job:http_errors_total:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))

      # Record error percentage
      - record: job:http_error_ratio:rate5m
        expr: |
          job:http_errors_total:rate5m
          /
          job:http_requests_total:rate5m

      # Record latency percentiles
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

Use recorded metrics:

```promql
# Instead of recalculating
sum by (job) (rate(http_requests_total[5m]))

# Use pre-computed
job:http_requests_total:rate5m
```

---

## Best Practices

### DO

```promql
# Use rate() for counters
rate(http_requests_total[5m])

# Use appropriate time ranges
rate(metric[5m])    # Good for real-time
rate(metric[1h])    # Good for trends

# Be specific with labels
http_requests_total{job="api", environment="prod"}

# Use recording rules for complex queries
job:requests:rate5m
```

### DON'T

```promql
# Don't use rate on gauges
rate(temperature_celsius[5m])  # Wrong!

# Don't use very short ranges
rate(metric[30s])  # Too short, unreliable

# Don't create high-cardinality queries
sum by (user_id) (...)  # High cardinality!

# Don't compare counters directly
http_requests_total > 1000  # Use rate() instead
```

---

## Summary

| Function | Purpose | Example |
|----------|---------|---------|
| `rate()` | Per-second rate | `rate(requests[5m])` |
| `sum()` | Aggregate values | `sum by (job) (metric)` |
| `avg()` | Average value | `avg(cpu_usage)` |
| `histogram_quantile()` | Percentiles | `histogram_quantile(0.99, ...)` |
| `increase()` | Total increase | `increase(requests[1h])` |
| `topk()` | Top K series | `topk(5, requests)` |
| `absent()` | Check missing | `absent(up{job="x"})` |
| `predict_linear()` | Prediction | `predict_linear(disk[1h], 3600)` |
