# Prometheus Exporters

## What are Exporters?

Exporters are programs that collect metrics from systems/applications and expose them in Prometheus format.

```
┌─────────────────────────────────────────────────────────────┐
│                    EXPORTERS                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐  │
│  │   System    │     │  Exporter   │     │ Prometheus  │  │
│  │  (Linux,    │────▶│  Translates │────▶│  Scrapes    │  │
│  │   MySQL)    │     │  to metrics │     │  /metrics   │  │
│  └─────────────┘     └─────────────┘     └─────────────┘  │
│                                                             │
│  Exporter exposes metrics at HTTP endpoint /metrics        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Common Exporters

```
┌────────────────────────────────────────────────────────────┐
│ Exporter              │ Port  │ Purpose                    │
├───────────────────────┼───────┼────────────────────────────┤
│ Node Exporter         │ 9100  │ Linux/Unix host metrics    │
│ Windows Exporter      │ 9182  │ Windows host metrics       │
│ cAdvisor              │ 8080  │ Container metrics          │
│ Blackbox Exporter     │ 9115  │ HTTP/TCP/ICMP probing      │
│ MySQL Exporter        │ 9104  │ MySQL database metrics     │
│ PostgreSQL Exporter   │ 9187  │ PostgreSQL metrics         │
│ Redis Exporter        │ 9121  │ Redis metrics              │
│ MongoDB Exporter      │ 9216  │ MongoDB metrics            │
│ Elasticsearch Exporter│ 9114  │ Elasticsearch metrics      │
│ RabbitMQ Exporter     │ 9419  │ RabbitMQ metrics           │
│ Kafka Exporter        │ 9308  │ Kafka metrics              │
│ NGINX Exporter        │ 9113  │ NGINX metrics              │
│ HAProxy Exporter      │ 9101  │ HAProxy metrics            │
└───────────────────────┴───────┴────────────────────────────┘
```

---

## Node Exporter

Exposes Linux/Unix host metrics (CPU, memory, disk, network).

### Installation

```bash
# Download
NODE_EXPORTER_VERSION="1.7.0"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz

# Extract and install
tar xvfz node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo mv node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/

# Create user
sudo useradd --no-create-home --shell /bin/false node_exporter
```

### Systemd Service

```bash
sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd \
  --collector.processes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

### Docker

```bash
docker run -d \
  --name node-exporter \
  --net host \
  --pid host \
  -v /:/host:ro,rslave \
  prom/node-exporter:latest \
  --path.rootfs=/host
```

### Key Metrics

```promql
# CPU usage percentage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage percentage
(1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) * 100

# Network traffic
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])

# Disk I/O
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])

# Load average
node_load1
node_load5
node_load15
```

### Prometheus Config

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'server1:9100'
          - 'server2:9100'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):\d+'
        target_label: instance
        replacement: '${1}'
```

---

## cAdvisor

Container metrics for Docker/Kubernetes.

### Docker Installation

```bash
docker run -d \
  --name cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:latest
```

### Key Metrics

```promql
# Container CPU usage
rate(container_cpu_usage_seconds_total{name!=""}[5m])

# Container memory usage
container_memory_usage_bytes{name!=""}

# Container network I/O
rate(container_network_receive_bytes_total{name!=""}[5m])
rate(container_network_transmit_bytes_total{name!=""}[5m])

# Container disk I/O
rate(container_fs_reads_bytes_total{name!=""}[5m])
rate(container_fs_writes_bytes_total{name!=""}[5m])
```

### Prometheus Config

```yaml
scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metric_relabel_configs:
      # Drop high-cardinality container_* metrics
      - source_labels: [__name__]
        regex: 'container_(network_tcp_usage_total|tasks_state|memory_failures_total)'
        action: drop
```

---

## Blackbox Exporter

Probe endpoints over HTTP, HTTPS, DNS, TCP, ICMP.

### Installation

```bash
docker run -d \
  --name blackbox-exporter \
  -p 9115:9115 \
  -v $(pwd)/blackbox.yml:/config/blackbox.yml \
  prom/blackbox-exporter:latest \
  --config.file=/config/blackbox.yml
```

### Configuration

```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 204]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: "ip4"

  http_post_2xx:
    prober: http
    timeout: 5s
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{}'

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"

  dns:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"
```

### Prometheus Config

```yaml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://myapp.com
          - https://api.myapp.com/health
          - https://google.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox-tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - db.example.com:5432
          - redis.example.com:6379
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### Key Metrics

```promql
# Probe success (1 = up, 0 = down)
probe_success{job="blackbox-http"}

# HTTP response time
probe_http_duration_seconds

# SSL certificate expiry
probe_ssl_earliest_cert_expiry - time()

# DNS lookup time
probe_dns_lookup_time_seconds
```

---

## MySQL Exporter

### Installation

```bash
docker run -d \
  --name mysql-exporter \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="exporter:password@(mysql:3306)/" \
  prom/mysqld-exporter:latest
```

### Create MySQL User

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY 'password';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
FLUSH PRIVILEGES;
```

### Key Metrics

```promql
# MySQL up
mysql_up

# Queries per second
rate(mysql_global_status_queries[5m])

# Connections
mysql_global_status_threads_connected
mysql_global_variables_max_connections

# Slow queries
rate(mysql_global_status_slow_queries[5m])

# InnoDB buffer pool usage
mysql_global_status_innodb_buffer_pool_pages_data / mysql_global_status_innodb_buffer_pool_pages_total
```

---

## PostgreSQL Exporter

### Installation

```bash
docker run -d \
  --name postgres-exporter \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://postgres:password@postgres:5432/postgres?sslmode=disable" \
  prometheuscommunity/postgres-exporter:latest
```

### Key Metrics

```promql
# PostgreSQL up
pg_up

# Active connections
pg_stat_activity_count{state="active"}

# Database size
pg_database_size_bytes

# Transaction rate
rate(pg_stat_database_xact_commit[5m])

# Replication lag
pg_replication_lag
```

---

## Redis Exporter

### Installation

```bash
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  -e REDIS_ADDR=redis:6379 \
  oliver006/redis_exporter:latest
```

### Key Metrics

```promql
# Redis up
redis_up

# Connected clients
redis_connected_clients

# Memory usage
redis_memory_used_bytes
redis_memory_max_bytes

# Commands per second
rate(redis_commands_processed_total[5m])

# Cache hit ratio
redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total)
```

---

## NGINX Exporter

### Enable NGINX Status

```nginx
# nginx.conf
server {
    location /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
}
```

### Installation

```bash
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:latest \
  -nginx.scrape-uri=http://nginx:80/nginx_status
```

### Key Metrics

```promql
# Active connections
nginx_connections_active

# Request rate
rate(nginx_http_requests_total[5m])

# Connection states
nginx_connections_reading
nginx_connections_writing
nginx_connections_waiting
```

---

## Creating Custom Exporters

### Python Example

```python
# custom_exporter.py
from prometheus_client import start_http_server, Gauge, Counter
import time
import random

# Define metrics
REQUEST_COUNT = Counter('app_requests_total', 'Total requests', ['method', 'endpoint'])
RESPONSE_TIME = Gauge('app_response_time_seconds', 'Response time', ['endpoint'])
ACTIVE_USERS = Gauge('app_active_users', 'Active users')

def collect_metrics():
    """Simulate collecting metrics from your application"""
    # Simulate request count
    REQUEST_COUNT.labels(method='GET', endpoint='/api/users').inc(random.randint(1, 10))
    REQUEST_COUNT.labels(method='POST', endpoint='/api/orders').inc(random.randint(0, 5))

    # Simulate response time
    RESPONSE_TIME.labels(endpoint='/api/users').set(random.uniform(0.1, 0.5))
    RESPONSE_TIME.labels(endpoint='/api/orders').set(random.uniform(0.2, 1.0))

    # Simulate active users
    ACTIVE_USERS.set(random.randint(50, 200))

if __name__ == '__main__':
    # Start HTTP server on port 8000
    start_http_server(8000)
    print("Exporter running on http://localhost:8000/metrics")

    while True:
        collect_metrics()
        time.sleep(15)
```

### Go Example

```go
// main.go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    requestCount = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "app_requests_total",
            Help: "Total requests",
        },
        []string{"method", "endpoint"},
    )
    responseTime = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "app_response_time_seconds",
            Help: "Response time",
        },
        []string{"endpoint"},
    )
)

func init() {
    prometheus.MustRegister(requestCount)
    prometheus.MustRegister(responseTime)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8000", nil)
}
```

---

## Exporter Best Practices

### DO

```
✓ Use meaningful metric names
✓ Add helpful HELP text
✓ Use appropriate metric types
✓ Keep cardinality low
✓ Include version/build info metric
✓ Implement health checks
```

### DON'T

```
✗ Use high-cardinality labels (user_id, request_id)
✗ Expose sensitive data in labels
✗ Create too many metrics
✗ Use counters for values that can decrease
✗ Forget to handle errors gracefully
```

---

## Summary

| Exporter | Use Case | Key Metrics |
|----------|----------|-------------|
| **Node Exporter** | Linux hosts | CPU, memory, disk, network |
| **cAdvisor** | Containers | Container CPU, memory |
| **Blackbox** | Endpoint probing | Probe success, latency |
| **MySQL** | MySQL databases | Queries, connections |
| **PostgreSQL** | PostgreSQL | Transactions, connections |
| **Redis** | Redis cache | Hit ratio, memory |
| **Custom** | Your applications | Application-specific |

### Quick Setup

```yaml
# docker-compose.yml addition
services:
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"

  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox.yml:/config/blackbox.yml
    command:
      - '--config.file=/config/blackbox.yml'
```
