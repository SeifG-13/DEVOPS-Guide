# Prometheus & Grafana Cheat Sheet for DevOps Engineers

## Quick Reference - Prometheus

### PromQL Basics
```promql
# Instant vector (current value)
http_requests_total

# Range vector (over time)
http_requests_total[5m]

# With labels
http_requests_total{job="api", status="200"}

# Label matching
http_requests_total{status=~"2.."}      # Regex match
http_requests_total{status!="500"}      # Not equal
http_requests_total{method=~"GET|POST"} # OR regex
```

### Common Functions
```promql
# Rate (per-second average)
rate(http_requests_total[5m])

# Increase (total increase over time)
increase(http_requests_total[1h])

# Sum by label
sum(rate(http_requests_total[5m])) by (status)

# Average
avg(node_cpu_seconds_total) by (instance)

# Max/Min
max(container_memory_usage_bytes) by (pod)

# Count
count(up == 1)

# Histogram quantile
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Offset (compare to past)
http_requests_total - http_requests_total offset 1h

# Absent (for alerting on missing metrics)
absent(up{job="myapp"})
```

### Aggregation Operators
```promql
sum, avg, min, max, count, stddev, stdvar
topk(3, http_requests_total)        # Top 3
bottomk(3, http_requests_total)     # Bottom 3
count_values("version", build_info) # Count by value
```

---

## Prometheus Configuration

### prometheus.yml
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alerts/*.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

### Alert Rules
```yaml
# alerts/app-alerts.yml
groups:
  - name: app-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | printf \"%.2f\" }}%"

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"

      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
```

---

## Common Exporters

| Exporter | Port | Metrics |
|----------|------|---------|
| Node Exporter | 9100 | Linux system metrics |
| cAdvisor | 8080 | Container metrics |
| Blackbox Exporter | 9115 | HTTP/TCP probes |
| MySQL Exporter | 9104 | MySQL metrics |
| PostgreSQL Exporter | 9187 | PostgreSQL metrics |
| Redis Exporter | 9121 | Redis metrics |
| Nginx Exporter | 9113 | Nginx metrics |

### Node Exporter Metrics
```promql
# CPU usage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Disk usage
(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100

# Network throughput
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

---

## Grafana

### Dashboard JSON Structure
```json
{
  "dashboard": {
    "title": "My Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{status}}"
          }
        ]
      }
    ]
  }
}
```

### Common Dashboard Panels

#### Request Rate
```promql
sum(rate(http_requests_total[5m])) by (status)
```

#### Error Rate Percentage
```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

#### Latency (95th percentile)
```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

#### Availability
```promql
avg(up{job="myapp"}) * 100
```

---

## Interview Q&A

### Q1: What is Prometheus and how does it work?
**A:** Prometheus is an open-source monitoring system that:
- Uses a pull model (scrapes metrics from targets)
- Stores time-series data
- Has powerful query language (PromQL)
- Supports alerting via Alertmanager
- Has service discovery for dynamic environments

### Q2: What is the difference between rate() and increase()?
**A:**
- **rate()**: Per-second average rate of increase
- **increase()**: Total increase over time period
Both handle counter resets. Use rate() for graphs, increase() for totals.

### Q3: How do you monitor a Kubernetes cluster?
**A:**
- **kube-state-metrics**: Cluster object state
- **Node Exporter**: Node-level metrics
- **cAdvisor**: Container metrics
- **Metrics Server**: Resource metrics for HPA
- **Prometheus Operator**: Manages Prometheus in K8s

### Q4: What is the difference between metrics and logs?
**A:**
- **Metrics**: Numeric, aggregatable, low cardinality, real-time monitoring
- **Logs**: Text-based, high cardinality, debugging, forensics

### Q5: How do you set up alerts in Prometheus?
**A:**
1. Define alert rules in YAML
2. Configure Alertmanager for routing
3. Set up notification channels (Slack, PagerDuty, email)
4. Use `for` clause to prevent flapping

### Q6: What are the four golden signals?
**A:**
1. **Latency**: Time to serve requests
2. **Traffic**: Requests per second
3. **Errors**: Error rate
4. **Saturation**: Resource utilization

### Q7: How do you handle high cardinality in Prometheus?
**A:**
- Avoid labels with high cardinality (user IDs, request IDs)
- Use recording rules for frequently used queries
- Limit label values
- Use separate metrics for debugging

### Q8: What is a recording rule?
**A:** Pre-computed PromQL expressions stored as new metrics:
```yaml
groups:
  - name: recording
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

### Q9: How do you integrate Grafana with Prometheus?
**A:**
1. Add Prometheus as data source (http://prometheus:9090)
2. Create dashboards using PromQL queries
3. Set up variables for dynamic filtering
4. Configure alerts in Grafana

### Q10: What is Alertmanager and how does it work?
**A:** Alertmanager handles alerts from Prometheus:
- Deduplication
- Grouping
- Silencing
- Inhibition
- Routing to receivers (Slack, PagerDuty, email)

---

## Kubernetes Deployment

### Prometheus Stack with Helm
```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### ServiceMonitor for Custom Apps
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

---

## Best Practices

1. **Use labels wisely** - Avoid high cardinality
2. **Set appropriate scrape intervals** - 15-60s for most cases
3. **Use recording rules** - Pre-compute expensive queries
4. **Alert on symptoms, not causes** - User-facing issues
5. **Have runbooks for alerts** - Document response procedures
6. **Use dashboards effectively** - Overview → Details hierarchy
7. **Implement SLOs** - Service level objectives
8. **Backup Prometheus data** - Persistent storage or remote write
