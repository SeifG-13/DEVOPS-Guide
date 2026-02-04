# Prometheus Configuration

## Configuration File Structure

```yaml
# prometheus.yml - Complete structure
global:
  # Default settings for all scrape configs
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    region: 'us-east-1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Rule files for alerts and recording rules
rule_files:
  - /etc/prometheus/rules/*.yml

# Scrape configurations
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

---

## Global Configuration

### Common Global Settings

```yaml
global:
  # How often to scrape targets
  scrape_interval: 15s

  # Timeout for scraping
  scrape_timeout: 10s

  # How often to evaluate rules
  evaluation_interval: 15s

  # Labels added to all metrics and alerts
  external_labels:
    cluster: 'production'
    environment: 'prod'
    region: 'eastus'
```

### Production-Ready Global Config

```yaml
global:
  scrape_interval: 30s      # Less frequent for large deployments
  scrape_timeout: 10s
  evaluation_interval: 30s

  external_labels:
    monitor: 'prometheus'
    cluster: 'prod-aks-eastus'

  # Query logging (optional)
  query_log_file: /var/log/prometheus/query.log
```

---

## Scrape Configurations

### Basic Static Config

```yaml
scrape_configs:
  # Each job_name creates a separate scrape configuration
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          environment: 'monitoring'
```

### Multiple Targets

```yaml
scrape_configs:
  - job_name: 'web-servers'
    static_configs:
      - targets:
          - 'web1.example.com:9100'
          - 'web2.example.com:9100'
          - 'web3.example.com:9100'
        labels:
          role: 'webserver'
          datacenter: 'dc1'

      - targets:
          - 'web4.example.com:9100'
          - 'web5.example.com:9100'
        labels:
          role: 'webserver'
          datacenter: 'dc2'
```

### Custom Scrape Settings

```yaml
scrape_configs:
  - job_name: 'slow-exporter'
    scrape_interval: 60s       # Override global
    scrape_timeout: 30s        # Override global
    metrics_path: /custom/metrics
    scheme: https
    static_configs:
      - targets: ['slow-app:8443']
```

### With Authentication

```yaml
scrape_configs:
  # Basic Auth
  - job_name: 'authenticated-app'
    basic_auth:
      username: 'prometheus'
      password: 'secret'
    static_configs:
      - targets: ['app:8080']

  # Bearer Token
  - job_name: 'token-auth-app'
    bearer_token: 'your-token-here'
    # Or from file:
    # bearer_token_file: /etc/prometheus/token
    static_configs:
      - targets: ['app:8080']

  # TLS Configuration
  - job_name: 'tls-app'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/ca.crt
      cert_file: /etc/prometheus/client.crt
      key_file: /etc/prometheus/client.key
      insecure_skip_verify: false
    static_configs:
      - targets: ['secure-app:8443']
```

---

## Service Discovery

### Kubernetes Service Discovery

```yaml
scrape_configs:
  # Discover pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      # Use custom path if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Use custom port if specified
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # Add pod labels
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)

      # Add namespace label
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace

      # Add pod name label
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # Discover services
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  # Discover endpoints
  - job_name: 'kubernetes-endpoints'
    kubernetes_sd_configs:
      - role: endpoints
```

### File-Based Service Discovery

```yaml
scrape_configs:
  - job_name: 'file-targets'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
          - '/etc/prometheus/targets/*.yml'
        refresh_interval: 5m
```

```json
// /etc/prometheus/targets/web-servers.json
[
  {
    "targets": ["web1:9100", "web2:9100"],
    "labels": {
      "env": "production",
      "role": "webserver"
    }
  }
]
```

```yaml
# /etc/prometheus/targets/databases.yml
- targets:
    - 'db1:9104'
    - 'db2:9104'
  labels:
    env: production
    role: database
```

### Consul Service Discovery

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'consul.example.com:8500'
        services: []  # All services, or list specific
    relabel_configs:
      - source_labels: [__meta_consul_tags]
        regex: .*,prometheus,.*
        action: keep
```

### Azure Service Discovery

```yaml
scrape_configs:
  - job_name: 'azure-vms'
    azure_sd_configs:
      - subscription_id: 'your-subscription-id'
        tenant_id: 'your-tenant-id'
        client_id: 'your-client-id'
        client_secret: 'your-client-secret'
        resource_group: 'rg-myapp-prod'
        port: 9100
```

---

## Relabeling

### Relabel Configs Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    RELABELING FLOW                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Service Discovery → relabel_configs → Scrape → metric_    │
│                                                relabel_     │
│                                                configs      │
│                                                             │
│  relabel_configs:                                          │
│  • Applied BEFORE scrape                                   │
│  • Modify target labels                                    │
│  • Filter targets (keep/drop)                              │
│                                                             │
│  metric_relabel_configs:                                   │
│  • Applied AFTER scrape                                    │
│  • Modify metric labels                                    │
│  • Drop unwanted metrics                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Common Relabel Actions

```yaml
relabel_configs:
  # KEEP - Only keep targets matching regex
  - source_labels: [__meta_kubernetes_pod_label_app]
    action: keep
    regex: myapp

  # DROP - Drop targets matching regex
  - source_labels: [__meta_kubernetes_namespace]
    action: drop
    regex: kube-system

  # REPLACE - Replace label value
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: namespace

  # LABELMAP - Copy labels matching regex
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
    replacement: pod_${1}

  # LABELDROP - Remove labels matching regex
  - action: labeldrop
    regex: __meta_.*

  # LABELKEEP - Keep only labels matching regex
  - action: labelkeep
    regex: (job|instance|namespace|pod)

  # HASHMOD - Sharding across multiple Prometheus
  - source_labels: [__address__]
    modulus: 3
    target_label: __tmp_hash
    action: hashmod
```

### Practical Relabel Examples

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with prometheus annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      # Get metrics path from annotation or default to /metrics
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Get port from annotation
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # Add useful labels
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app

    # Drop high-cardinality metrics after scrape
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: go_.*
        action: drop
```

---

## Metric Relabeling

### Drop Unwanted Metrics

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
    metric_relabel_configs:
      # Drop all go_* metrics
      - source_labels: [__name__]
        regex: 'go_.*'
        action: drop

      # Drop specific high-cardinality metrics
      - source_labels: [__name__]
        regex: 'node_filesystem_.*'
        action: drop

      # Keep only specific metrics
      - source_labels: [__name__]
        regex: 'node_(cpu|memory|disk|network)_.*'
        action: keep
```

### Rename Metrics

```yaml
metric_relabel_configs:
  # Rename metric
  - source_labels: [__name__]
    regex: 'old_metric_name'
    target_label: __name__
    replacement: 'new_metric_name'

  # Add prefix to all metrics
  - source_labels: [__name__]
    regex: '(.*)'
    target_label: __name__
    replacement: 'myapp_${1}'
```

---

## Alerting Configuration

```yaml
alerting:
  # Alert relabel configs
  alert_relabel_configs:
    - source_labels: [severity]
      regex: 'warning'
      action: drop  # Don't send warnings to alertmanager

  # Alertmanager targets
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager1:9093'
            - 'alertmanager2:9093'

      # Optional: customize alertmanager connection
      scheme: https
      tls_config:
        ca_file: /etc/prometheus/am-ca.crt
      basic_auth:
        username: 'prometheus'
        password: 'secret'
```

---

## Rule Files

```yaml
rule_files:
  - /etc/prometheus/rules/alerting-rules.yml
  - /etc/prometheus/rules/recording-rules.yml
  - /etc/prometheus/rules/*.yml
```

### Example Rule File

```yaml
# /etc/prometheus/rules/alerting-rules.yml
groups:
  - name: example-alerts
    interval: 30s  # Override global evaluation_interval
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors/sec"
```

---

## Storage Configuration

### Command Line Flags

```bash
prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB \
  --storage.tsdb.wal-compression
```

### Storage Settings

```
┌────────────────────────────────────────────────────────────┐
│ Flag                              │ Description            │
├───────────────────────────────────┼────────────────────────┤
│ --storage.tsdb.path               │ Data directory         │
│ --storage.tsdb.retention.time     │ How long to keep data  │
│ --storage.tsdb.retention.size     │ Max storage size       │
│ --storage.tsdb.wal-compression    │ Compress WAL           │
│ --storage.tsdb.min-block-duration │ Min block duration     │
│ --storage.tsdb.max-block-duration │ Max block duration     │
└───────────────────────────────────┴────────────────────────┘
```

---

## Web Configuration

### Enable Admin API

```bash
prometheus \
  --web.enable-lifecycle \      # Enable /-/reload and /-/quit
  --web.enable-admin-api        # Enable /api/v1/admin/* endpoints
```

### Authentication (prometheus.yml won't help - use reverse proxy)

```yaml
# Use nginx or similar for authentication
# prometheus.yml has limited auth support
```

---

## Complete Production Config

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 30s
  scrape_timeout: 10s
  evaluation_interval: 30s
  external_labels:
    cluster: 'prod-aks-eastus'
    environment: 'production'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (infrastructure)
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'node1:9100'
          - 'node2:9100'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):\d+'
        target_label: instance
        replacement: '${1}'

  # Application metrics
  - job_name: 'myapp-api'
    metrics_path: /metrics
    static_configs:
      - targets: ['api1:8080', 'api2:8080']

  # Kubernetes pods (if on K8s)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

---

## Validate Configuration

```bash
# Check syntax
promtool check config /etc/prometheus/prometheus.yml

# Test rules
promtool check rules /etc/prometheus/rules/*.yml

# Reload configuration (if --web.enable-lifecycle)
curl -X POST http://localhost:9090/-/reload

# Or send SIGHUP
kill -HUP $(pgrep prometheus)
```

---

## Summary

| Section | Purpose |
|---------|---------|
| **global** | Default scrape settings |
| **alerting** | Alertmanager configuration |
| **rule_files** | Alert and recording rules |
| **scrape_configs** | Target definitions |
| **relabel_configs** | Modify labels before scrape |
| **metric_relabel_configs** | Modify metrics after scrape |
