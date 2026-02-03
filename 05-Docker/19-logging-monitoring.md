# Docker Logging and Monitoring

## Container Logs

### docker logs Command

```bash
# View logs
docker logs container_name

# Follow logs (real-time)
docker logs -f container_name

# Show timestamps
docker logs -t container_name

# Tail last N lines
docker logs --tail 100 container_name

# Logs since time
docker logs --since 2024-01-01T00:00:00 container_name
docker logs --since 1h container_name

# Logs until time
docker logs --until 2024-01-01T12:00:00 container_name

# Combined options
docker logs -f --tail 50 -t container_name
```

### Log Output Streams

```bash
# stdout and stderr are captured
docker logs container_name          # Both
docker logs container_name 2>&1     # Both (explicit)

# In container, write to stdout/stderr
echo "info" >&1    # stdout → docker logs
echo "error" >&2   # stderr → docker logs
```

## Logging Drivers

### Available Drivers

| Driver | Description |
|--------|-------------|
| `json-file` | Default, JSON format |
| `local` | Optimized local logging |
| `syslog` | Syslog server |
| `journald` | Systemd journal |
| `fluentd` | Fluentd collector |
| `awslogs` | Amazon CloudWatch |
| `gcplogs` | Google Cloud Logging |
| `splunk` | Splunk |
| `gelf` | Graylog Extended Log Format |
| `none` | Disable logging |

### Configure Logging Driver

```bash
# Per container
docker run --log-driver=json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    nginx

# Global (daemon.json)
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### json-file Driver

```bash
# Default driver, stores logs as JSON

# Options
docker run \
    --log-driver=json-file \
    --log-opt max-size=10m \
    --log-opt max-file=5 \
    --log-opt compress=true \
    nginx

# Log location
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

### syslog Driver

```bash
docker run \
    --log-driver=syslog \
    --log-opt syslog-address=udp://192.168.0.42:514 \
    --log-opt syslog-facility=daemon \
    --log-opt tag="{{.Name}}" \
    nginx
```

### fluentd Driver

```bash
docker run \
    --log-driver=fluentd \
    --log-opt fluentd-address=localhost:24224 \
    --log-opt tag="docker.{{.Name}}" \
    nginx
```

### AWS CloudWatch

```bash
docker run \
    --log-driver=awslogs \
    --log-opt awslogs-region=us-east-1 \
    --log-opt awslogs-group=myapp-logs \
    --log-opt awslogs-stream=container-1 \
    nginx
```

## Docker Compose Logging

```yaml
version: '3.8'

services:
  app:
    image: myapp
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  api:
    image: myapi
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: api.{{.Name}}

  worker:
    image: worker
    logging:
      driver: none  # Disable logging
```

## Monitoring Containers

### docker stats

```bash
# Real-time resource usage
docker stats

# Specific containers
docker stats container1 container2

# No streaming (one snapshot)
docker stats --no-stream

# Custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# JSON output
docker stats --format json
```

### Output Columns

| Column | Description |
|--------|-------------|
| CONTAINER ID | Container identifier |
| NAME | Container name |
| CPU % | CPU utilization |
| MEM USAGE / LIMIT | Memory usage |
| MEM % | Memory percentage |
| NET I/O | Network traffic |
| BLOCK I/O | Disk I/O |
| PIDS | Number of processes |

### docker top

```bash
# View processes in container
docker top container_name

# With specific ps options
docker top container_name aux
docker top container_name -ef
```

## Health Checks

### Dockerfile HEALTHCHECK

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# TCP health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD nc -z localhost 3000 || exit 1

# Custom script
HEALTHCHECK --interval=1m --timeout=10s \
    CMD /app/healthcheck.sh || exit 1

# Disable health check
HEALTHCHECK NONE
```

### Health Check Options

| Option | Description | Default |
|--------|-------------|---------|
| `--interval` | Time between checks | 30s |
| `--timeout` | Check timeout | 30s |
| `--start-period` | Startup grace period | 0s |
| `--retries` | Failures before unhealthy | 3 |

### Check Health Status

```bash
# View health status
docker inspect --format '{{.State.Health.Status}}' container_name

# Health status values: starting, healthy, unhealthy

# View health check logs
docker inspect --format '{{json .State.Health}}' container_name | jq
```

### Docker Compose Health Check

```yaml
services:
  app:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## Centralized Logging Stack

### ELK Stack (Elasticsearch, Logstash, Kibana)

```yaml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  app:
    image: myapp
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"

volumes:
  es_data:
```

### Loki + Grafana

```yaml
version: '3.8'

services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  loki_data:
  grafana_data:
```

## Prometheus Monitoring

### Docker Daemon Metrics

```bash
# Enable metrics endpoint
# /etc/docker/daemon.json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}

# Metrics available at http://localhost:9323/metrics
```

### cAdvisor

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
```

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

## Docker Events

```bash
# Watch all events
docker events

# Filter by container
docker events --filter container=myapp

# Filter by event type
docker events --filter event=start
docker events --filter event=stop
docker events --filter event=die

# Filter by time
docker events --since "2024-01-01"
docker events --until "2024-01-02"

# Format output
docker events --format '{{.Time}} {{.Actor.Attributes.name}} {{.Action}}'

# JSON format
docker events --format '{{json .}}'
```

## Troubleshooting

### Check Container Logs

```bash
# Recent errors
docker logs --tail 100 container_name | grep -i error

# Logs around specific time
docker logs --since "2024-01-01T10:00:00" --until "2024-01-01T11:00:00" container_name
```

### Resource Issues

```bash
# Check resource usage
docker stats --no-stream

# Check disk usage
docker system df
docker system df -v

# Check for OOM kills
docker inspect container_name | jq '.[0].State.OOMKilled'
```

### Log File Location

```bash
# Find container log files
docker inspect --format='{{.LogPath}}' container_name

# View raw log file
cat /var/lib/docker/containers/<id>/<id>-json.log
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker logs` | View logs |
| `docker logs -f` | Follow logs |
| `docker stats` | Resource usage |
| `docker top` | Container processes |
| `docker events` | System events |
| `docker inspect` | Container details |
| `--log-driver` | Set logging driver |
| `HEALTHCHECK` | Define health check |

---

**Previous:** [18-optimization.md](18-optimization.md) | **Next:** [20-best-practices.md](20-best-practices.md)
