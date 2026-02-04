# Prometheus Overview

## What is Prometheus?

Prometheus is an open-source monitoring and alerting toolkit, originally built at SoundCloud and now part of the Cloud Native Computing Foundation (CNCF).

```
┌─────────────────────────────────────────────────────────────┐
│                    PROMETHEUS                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  "Prometheus is the de facto standard for           │   │
│  │   monitoring cloud-native applications"             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Key Characteristics:                                       │
│  • Time-series database                                    │
│  • Pull-based model                                        │
│  • Multi-dimensional data model                            │
│  • Powerful query language (PromQL)                        │
│  • No distributed storage dependency                       │
│  • Service discovery                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Targets                                                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                  │
│  │  App    │ │  Node   │ │  K8s    │ │ Database│                  │
│  │/metrics │ │Exporter │ │ Metrics │ │ Exporter│                  │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘                  │
│       │           │           │           │                         │
│       └───────────┴─────┬─────┴───────────┘                         │
│                         │ HTTP Pull (scrape)                        │
│                         ▼                                           │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                      PROMETHEUS SERVER                        │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │ │
│  │  │  Retrieval  │  │    TSDB     │  │    HTTP     │          │ │
│  │  │  (Scraper)  │─▶│  (Storage)  │◀─│   Server    │          │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │ │
│  │                          │                 ▲                  │ │
│  │  ┌─────────────┐        │                 │                  │ │
│  │  │    Rules    │────────┘                 │                  │ │
│  │  │   Engine    │                          │                  │ │
│  │  └──────┬──────┘                          │                  │ │
│  └─────────┼─────────────────────────────────┼──────────────────┘ │
│            │                                 │                     │
│            ▼                                 │                     │
│  ┌─────────────────┐               ┌─────────────────┐            │
│  │  Alertmanager   │               │     Grafana     │            │
│  │                 │               │   (PromQL API)  │            │
│  │ • Routes alerts │               │                 │            │
│  │ • Notifications │               │ • Dashboards    │            │
│  └─────────────────┘               └─────────────────┘            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Prometheus Server

```
Main Components:
┌────────────────────────────────────────────────────────────┐
│ Retrieval      │ Pulls metrics from targets via HTTP      │
│ TSDB           │ Stores time-series data locally          │
│ HTTP Server    │ Exposes PromQL API and web UI            │
│ Rules Engine   │ Evaluates alerting/recording rules       │
└────────────────────────────────────────────────────────────┘
```

### 2. Time Series Database (TSDB)

```
Data Model:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  metric_name{label1="value1", label2="value2"} value       │
│                                                            │
│  Example:                                                  │
│  http_requests_total{method="GET", path="/api"} 1234       │
│                                                            │
│  Components:                                               │
│  ├── Metric name: http_requests_total                     │
│  ├── Labels: method="GET", path="/api"                    │
│  └── Value: 1234 (with timestamp)                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 3. Exporters

```
Exporters expose metrics in Prometheus format:

┌─────────────────┬────────────────────────────────────────┐
│ Exporter        │ Purpose                                │
├─────────────────┼────────────────────────────────────────┤
│ Node Exporter   │ Linux/Unix host metrics                │
│ Windows Exporter│ Windows host metrics                   │
│ cAdvisor        │ Container metrics                      │
│ Blackbox        │ HTTP/TCP/ICMP probing                  │
│ MySQL Exporter  │ MySQL database metrics                 │
│ Redis Exporter  │ Redis metrics                          │
│ Custom          │ Application-specific metrics           │
└─────────────────┴────────────────────────────────────────┘
```

### 4. Alertmanager

```
Handles alerts from Prometheus:

┌────────────────────────────────────────────────────────────┐
│                    ALERTMANAGER                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Functions:                                                │
│  ├── Deduplication    (remove duplicates)                 │
│  ├── Grouping         (group related alerts)              │
│  ├── Routing          (send to correct receiver)          │
│  ├── Silencing        (suppress alerts temporarily)       │
│  └── Inhibition       (suppress if other alert fires)     │
│                                                            │
│  Receivers:                                                │
│  • Email                                                   │
│  • Slack                                                   │
│  • PagerDuty                                              │
│  • Microsoft Teams                                         │
│  • Webhooks                                                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Pull-Based Model

### How Prometheus Scrapes

```
┌──────────────────────────────────────────────────────────────┐
│                    PULL MODEL                                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Prometheus discovers targets (service discovery)         │
│  2. Prometheus sends HTTP GET to /metrics endpoint           │
│  3. Target returns metrics in text format                    │
│  4. Prometheus stores in TSDB with timestamp                 │
│  5. Repeat at configured interval (scrape_interval)          │
│                                                              │
│  ┌────────────┐     GET /metrics      ┌────────────┐        │
│  │ Prometheus │─────────────────────▶│   Target   │        │
│  │            │◀─────────────────────│  /metrics  │        │
│  └────────────┘     Metrics text      └────────────┘        │
│                                                              │
│  Default scrape interval: 15s                               │
│  Default scrape timeout: 10s                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Example /metrics Response

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/api/users"} 1234
http_requests_total{method="POST",path="/api/users"} 567

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 500
http_request_duration_seconds_bucket{le="0.5"} 900
http_request_duration_seconds_bucket{le="1.0"} 990
http_request_duration_seconds_bucket{le="+Inf"} 1000
http_request_duration_seconds_sum 450.5
http_request_duration_seconds_count 1000

# HELP process_cpu_seconds_total Total CPU time
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 123.45
```

---

## Data Model

### Time Series

```
A time series is uniquely identified by:
- Metric name
- Set of labels (key-value pairs)

Example time series:
┌────────────────────────────────────────────────────────────┐
│ http_requests_total{method="GET", status="200"}           │
├────────────────────────────────────────────────────────────┤
│ Timestamp              │ Value                             │
├────────────────────────┼───────────────────────────────────┤
│ 1705312800             │ 1000                              │
│ 1705312815             │ 1015                              │
│ 1705312830             │ 1042                              │
│ 1705312845             │ 1089                              │
└────────────────────────┴───────────────────────────────────┘
```

### Labels

```
Labels enable dimensional data:

Without labels (flat):
  requests_total_get_200 = 1000
  requests_total_get_500 = 50
  requests_total_post_200 = 500

With labels (dimensional):
  http_requests_total{method="GET", status="200"} = 1000
  http_requests_total{method="GET", status="500"} = 50
  http_requests_total{method="POST", status="200"} = 500

Query flexibility:
  sum(http_requests_total)                     # All requests
  sum(http_requests_total{method="GET"})       # GET only
  sum(http_requests_total{status=~"5.."})      # 5xx errors
```

### Label Best Practices

```
Good Labels:
✓ method="GET"           # Bounded, known values
✓ status="200"           # Limited cardinality
✓ instance="server-01"   # Identifiable
✓ environment="prod"     # Meaningful grouping

Bad Labels:
✗ user_id="12345"        # High cardinality!
✗ request_id="abc-xyz"   # Unique per request!
✗ email="user@test.com"  # PII, high cardinality
✗ timestamp="..."        # Already part of data model
```

---

## Service Discovery

### Static Configuration

```yaml
scrape_configs:
  - job_name: 'myapp'
    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
```

### Dynamic Service Discovery

```yaml
# Kubernetes
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod

# Consul
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'localhost:8500'

# Azure
scrape_configs:
  - job_name: 'azure-vms'
    azure_sd_configs:
      - subscription_id: 'xxx'
        tenant_id: 'xxx'
        client_id: 'xxx'
        client_secret: 'xxx'

# File-based
scrape_configs:
  - job_name: 'file-targets'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
```

---

## When to Use Prometheus

### Good Use Cases

```
✓ Cloud-native applications
✓ Microservices monitoring
✓ Kubernetes environments
✓ Dynamic infrastructure
✓ Multi-dimensional metrics
✓ Real-time alerting
```

### Not Ideal For

```
✗ Log aggregation (use Loki/ELK)
✗ Distributed tracing (use Jaeger/Zipkin)
✗ Long-term storage (use Thanos/Cortex)
✗ High-cardinality data
✗ Event-based billing/analytics
```

---

## Prometheus vs Other Tools

```
┌─────────────────────────────────────────────────────────────────┐
│              PROMETHEUS vs ALTERNATIVES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tool          │ Model  │ Query     │ Best For                 │
│  ──────────────┼────────┼───────────┼──────────────────────────│
│  Prometheus    │ Pull   │ PromQL    │ Cloud-native, K8s        │
│  InfluxDB      │ Push   │ InfluxQL  │ IoT, time-series         │
│  Graphite      │ Push   │ Graphite  │ Legacy systems           │
│  Datadog       │ Agent  │ Custom    │ SaaS, all-in-one         │
│  CloudWatch    │ Push   │ Custom    │ AWS-native               │
│  Azure Monitor │ Agent  │ KQL       │ Azure-native             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## PromQL Introduction

### Basic Queries

```promql
# Instant vector - current value
http_requests_total

# Range vector - values over time
http_requests_total[5m]

# With label filter
http_requests_total{method="GET"}

# Regex match
http_requests_total{status=~"5.."}

# Negative match
http_requests_total{method!="OPTIONS"}
```

### Common Functions

```promql
# Rate - per-second rate of increase
rate(http_requests_total[5m])

# Sum - aggregate across labels
sum(rate(http_requests_total[5m]))

# Sum by label
sum by (method) (rate(http_requests_total[5m]))

# Percentile from histogram
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Prometheus Server** | Scrapes, stores, and queries metrics |
| **TSDB** | Time-series database for metric storage |
| **Exporters** | Expose metrics from systems/apps |
| **Alertmanager** | Handles alert routing and notifications |
| **PromQL** | Query language for metrics |
| **Service Discovery** | Automatic target discovery |

### Key Characteristics

```
┌────────────────────────────────────────────────────────────┐
│ ✓ Pull-based model - Prometheus scrapes targets           │
│ ✓ Multi-dimensional - Labels for flexible querying        │
│ ✓ PromQL - Powerful query language                        │
│ ✓ Standalone - No external dependencies                   │
│ ✓ Reliable - Operates during outages                      │
│ ✓ Cloud-native - Built for dynamic environments           │
└────────────────────────────────────────────────────────────┘
```
