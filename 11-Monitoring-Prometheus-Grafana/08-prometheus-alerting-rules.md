# Prometheus Alerting Rules

## Alert Rules Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    ALERTING FLOW                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  prometheus.yml          rules/*.yml         alertmanager   │
│  ┌─────────────┐        ┌─────────────┐     ┌───────────┐  │
│  │rule_files:  │───────▶│alert: Name  │────▶│ Routes    │  │
│  │  - rules/   │        │expr: ...    │     │ Receivers │  │
│  └─────────────┘        │for: 5m      │     └───────────┘  │
│                         │labels: ...  │                     │
│                         │annotations: │                     │
│                         └─────────────┘                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Rule File Structure

```yaml
# /etc/prometheus/rules/alerting-rules.yml
groups:
  - name: example-group
    interval: 30s  # Optional: override global evaluation_interval
    rules:
      - alert: AlertName
        expr: <PromQL expression>
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Short summary"
          description: "Detailed description"
```

### Enable in Prometheus

```yaml
# prometheus.yml
rule_files:
  - /etc/prometheus/rules/*.yml
  - /etc/prometheus/rules/alerting/*.yml
```

---

## Alert Rule Syntax

```yaml
- alert: HighErrorRate
  # PromQL expression - fires when result is > 0
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
    /
    sum(rate(http_requests_total[5m]))
    > 0.05

  # How long condition must be true before firing
  for: 5m

  # Labels added to alert
  labels:
    severity: critical
    team: backend

  # Annotations for notification templates
  annotations:
    summary: "High error rate detected"
    description: "Error rate is {{ $value | printf \"%.2f\" }}% (threshold: 5%)"
    runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
```

---

## Common Alert Rules

### Infrastructure Alerts

```yaml
groups:
  - name: infrastructure
    rules:
      # High CPU Usage
      - alert: HighCPUUsage
        expr: |
          100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}%"

      # High Memory Usage
      - alert: HighMemoryUsage
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | printf \"%.1f\" }}%"

      # Disk Space Low
      - alert: DiskSpaceLow
        expr: |
          (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} is {{ $value | printf \"%.1f\" }}% full"

      # Disk Space Critical
      - alert: DiskSpaceCritical
        expr: |
          (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) * 100 > 95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space critical on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} is {{ $value | printf \"%.1f\" }}% full"

      # Disk Will Fill Soon
      - alert: DiskWillFillIn24Hours
        expr: |
          predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 24*3600) < 0
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Disk will fill within 24 hours on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} will be full based on current trend"

      # Node Down
      - alert: NodeDown
        expr: up{job="node-exporter"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} is down"
          description: "Node exporter is not responding"
```

### Application Alerts

```yaml
groups:
  - name: application
    rules:
      # Service Down
      - alert: ServiceDown
        expr: up{job="myapp"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Instance {{ $labels.instance }} is not responding"

      # High Error Rate
      - alert: HighErrorRate
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in {{ $labels.job }}"
          description: "Error rate is {{ $value | printf \"%.2f\" }}%"

      # High Latency (P99)
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency in {{ $labels.job }}"
          description: "P99 latency is {{ $value | printf \"%.2f\" }}s"

      # High Latency (P99) Critical
      - alert: HighLatencyP99Critical
        expr: |
          histogram_quantile(0.99, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))) > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Critical P99 latency in {{ $labels.job }}"
          description: "P99 latency is {{ $value | printf \"%.2f\" }}s"

      # Too Many Requests
      - alert: HighRequestRate
        expr: |
          sum by (job) (rate(http_requests_total[5m])) > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High request rate in {{ $labels.job }}"
          description: "Request rate is {{ $value | printf \"%.0f\" }} req/s"

      # No Requests (possible outage)
      - alert: NoRequests
        expr: |
          sum by (job) (rate(http_requests_total[5m])) == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "No requests to {{ $labels.job }}"
          description: "Service may be down or unreachable"
```

### Database Alerts

```yaml
groups:
  - name: database
    rules:
      # MySQL Down
      - alert: MySQLDown
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL is down"
          description: "MySQL instance {{ $labels.instance }} is not responding"

      # Too Many Connections
      - alert: MySQLTooManyConnections
        expr: |
          mysql_global_status_threads_connected
          /
          mysql_global_variables_max_connections
          > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL too many connections"
          description: "{{ $value | printf \"%.0f\" }}% of max connections used"

      # Slow Queries
      - alert: MySQLSlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL slow queries detected"
          description: "{{ $value | printf \"%.2f\" }} slow queries/sec"

      # PostgreSQL Down
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"
          description: "PostgreSQL instance {{ $labels.instance }} is not responding"

      # PostgreSQL Replication Lag
      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication lag"
          description: "Replication lag is {{ $value }}s"

      # Redis Down
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis is down"
          description: "Redis instance {{ $labels.instance }} is not responding"

      # Redis Memory High
      - alert: RedisMemoryHigh
        expr: |
          redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage high"
          description: "Redis memory usage is {{ $value | printf \"%.0f\" }}%"
```

### Container/Kubernetes Alerts

```yaml
groups:
  - name: kubernetes
    rules:
      # Pod CrashLooping
      - alert: PodCrashLooping
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
          description: "Pod has restarted {{ $value }} times in the last hour"

      # Pod Not Ready
      - alert: PodNotReady
        expr: |
          kube_pod_status_ready{condition="true"} == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"
          description: "Pod has been not ready for more than 5 minutes"

      # Container CPU High
      - alert: ContainerCPUHigh
        expr: |
          sum by (namespace, pod, container) (rate(container_cpu_usage_seconds_total[5m])) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.container }} CPU high"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}%"

      # Container Memory High
      - alert: ContainerMemoryHigh
        expr: |
          container_memory_usage_bytes / container_spec_memory_limit_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.container }} memory high"
          description: "Memory usage is {{ $value | printf \"%.1f\" }}%"

      # Deployment Replicas Mismatch
      - alert: DeploymentReplicasMismatch
        expr: |
          kube_deployment_spec_replicas != kube_deployment_status_replicas_available
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} replicas mismatch"
          description: "Desired replicas != available replicas"

      # PVC Almost Full
      - alert: PVCAlmostFull
        expr: |
          kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PVC {{ $labels.persistentvolumeclaim }} almost full"
          description: "PVC is {{ $value | printf \"%.1f\" }}% full"
```

### Blackbox/Endpoint Alerts

```yaml
groups:
  - name: blackbox
    rules:
      # Endpoint Down
      - alert: EndpointDown
        expr: probe_success == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"
          description: "Endpoint has been unreachable for more than 1 minute"

      # Slow Endpoint
      - alert: SlowEndpoint
        expr: probe_http_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Endpoint {{ $labels.instance }} is slow"
          description: "Response time is {{ $value | printf \"%.2f\" }}s"

      # SSL Certificate Expiring Soon
      - alert: SSLCertificateExpiringSoon
        expr: |
          (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate expiring soon for {{ $labels.instance }}"
          description: "Certificate expires in {{ $value | printf \"%.0f\" }} days"

      # SSL Certificate Expiring Very Soon
      - alert: SSLCertificateExpiringVerySoon
        expr: |
          (probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "SSL certificate expiring very soon for {{ $labels.instance }}"
          description: "Certificate expires in {{ $value | printf \"%.0f\" }} days"
```

---

## Recording Rules

Pre-compute expensive queries to speed up dashboards and alerts.

```yaml
groups:
  - name: recording-rules
    interval: 30s
    rules:
      # Request rate by job
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # Error rate by job
      - record: job:http_errors_total:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))

      # Error percentage
      - record: job:http_error_ratio
        expr: |
          job:http_errors_total:rate5m / job:http_requests_total:rate5m

      # P99 latency
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )

      # CPU usage by instance
      - record: instance:node_cpu_utilization:ratio
        expr: |
          1 - avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m]))

      # Memory usage by instance
      - record: instance:node_memory_utilization:ratio
        expr: |
          1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

Use in alerts:

```yaml
- alert: HighErrorRate
  expr: job:http_error_ratio > 0.05  # Use recording rule
  for: 5m
```

---

## Template Variables

```yaml
annotations:
  # Use $labels to access alert labels
  summary: "High CPU on {{ $labels.instance }}"

  # Use $value for the expression result
  description: "CPU usage is {{ $value }}%"

  # Format numbers
  description: "Value is {{ $value | printf \"%.2f\" }}"

  # Humanize bytes
  description: "Disk free: {{ $value | humanize1024 }}B"

  # Include runbook link
  runbook_url: "https://wiki.example.com/runbooks/{{ $labels.alertname }}"
```

---

## Validate Rules

```bash
# Check syntax
promtool check rules /etc/prometheus/rules/*.yml

# Test rules against data
promtool test rules test.yml

# Example test file
# test.yml
rule_files:
  - /etc/prometheus/rules/alerting-rules.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{job="api",status="500"}'
        values: '0+10x10'
      - series: 'http_requests_total{job="api",status="200"}'
        values: '0+100x10'

    alert_rule_test:
      - eval_time: 10m
        alertname: HighErrorRate
        exp_alerts:
          - exp_labels:
              job: api
              severity: critical
```

---

## Best Practices

### Alerting Philosophy

```
┌────────────────────────────────────────────────────────────┐
│                  ALERTING BEST PRACTICES                   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  DO:                                                       │
│  ✓ Alert on symptoms, not causes                          │
│  ✓ Use 'for' duration to avoid flapping                   │
│  ✓ Include actionable runbook links                       │
│  ✓ Set appropriate severity levels                        │
│  ✓ Test alerts before deploying                           │
│                                                            │
│  DON'T:                                                    │
│  ✗ Alert on every metric                                  │
│  ✗ Set 'for' too short (causes noise)                     │
│  ✗ Ignore alerts (leads to alert fatigue)                 │
│  ✗ Page for warnings                                      │
│  ✗ Create duplicate alerts                                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Severity Levels

```yaml
# Critical - Page immediately, wake people up
severity: critical
# - Service is down
# - Data loss occurring
# - SLA breach imminent

# Warning - Needs attention during business hours
severity: warning
# - Approaching thresholds
# - Degraded performance
# - Potential issues

# Info - For dashboards/tracking, no notification
severity: info
# - Informational events
# - Trends to watch
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **expr** | PromQL expression defining condition |
| **for** | Duration condition must be true |
| **labels** | Added to alert for routing |
| **annotations** | Information for notifications |
| **Recording rules** | Pre-compute expensive queries |

### Quick Reference

```yaml
# Alert template
- alert: AlertName
  expr: metric > threshold
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "{{ $labels.instance }}"
    description: "Value: {{ $value }}"
```
