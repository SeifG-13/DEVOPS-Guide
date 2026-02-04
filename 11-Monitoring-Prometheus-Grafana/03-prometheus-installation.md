# Prometheus Installation

## Installation Options

```
┌─────────────────────────────────────────────────────────────┐
│              INSTALLATION OPTIONS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Option              │ Best For                             │
│  ────────────────────┼──────────────────────────────────────│
│  Binary              │ Linux servers, VMs                   │
│  Docker              │ Quick setup, development             │
│  Docker Compose      │ Multi-container dev environments     │
│  Kubernetes (Helm)   │ Production K8s/AKS                   │
│  Operator            │ Advanced K8s management              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Binary Installation (Linux)

### Download and Install

```bash
# Set version
PROMETHEUS_VERSION="2.48.0"

# Download
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz

# Extract
tar xvfz prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz

# Move to /usr/local
sudo mv prometheus-${PROMETHEUS_VERSION}.linux-amd64 /usr/local/prometheus

# Create symlinks
sudo ln -s /usr/local/prometheus/prometheus /usr/local/bin/prometheus
sudo ln -s /usr/local/prometheus/promtool /usr/local/bin/promtool
```

### Create User and Directories

```bash
# Create prometheus user
sudo useradd --no-create-home --shell /bin/false prometheus

# Create directories
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus

# Set ownership
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo chown -R prometheus:prometheus /usr/local/prometheus
```

### Basic Configuration

```bash
# Create config file
sudo tee /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF

sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### Create Systemd Service

```bash
sudo tee /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/usr/local/prometheus/consoles \
  --web.console.libraries=/usr/local/prometheus/console_libraries \
  --storage.tsdb.retention.time=15d \
  --web.enable-lifecycle

ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start Prometheus

```bash
# Reload systemd
sudo systemctl daemon-reload

# Start prometheus
sudo systemctl start prometheus

# Enable on boot
sudo systemctl enable prometheus

# Check status
sudo systemctl status prometheus

# View logs
sudo journalctl -u prometheus -f
```

### Verify Installation

```bash
# Check version
prometheus --version

# Validate config
promtool check config /etc/prometheus/prometheus.yml

# Access web UI
curl http://localhost:9090/-/healthy
# Open browser: http://localhost:9090
```

---

## Docker Installation

### Simple Docker Run

```bash
# Create directories
mkdir -p ~/prometheus/data

# Create config
cat > ~/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF

# Run Prometheus
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v ~/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v ~/prometheus/data:/prometheus \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --web.enable-lifecycle

# Check logs
docker logs prometheus

# Access: http://localhost:9090
```

### Docker with Custom Network

```bash
# Create network
docker network create monitoring

# Run Prometheus on network
docker run -d \
  --name prometheus \
  --network monitoring \
  -p 9090:9090 \
  -v ~/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus-data:/prometheus \
  prom/prometheus:latest
```

---

## Docker Compose Installation

### Complete Monitoring Stack

```yaml
# docker-compose.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    networks:
      - monitoring
```

### Prometheus Configuration for Compose

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'grafana'
    static_configs:
      - targets: ['grafana:3000']
```

### Alertmanager Configuration

```yaml
# alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default'

receivers:
  - name: 'default'
    # Configure receivers later
```

### Start the Stack

```bash
# Create directory structure
mkdir -p prometheus/rules alertmanager grafana/provisioning

# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f prometheus

# Stop all
docker-compose down
```

---

## Kubernetes Installation (Helm)

### Using kube-prometheus-stack

```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install with default values
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring

# Or with custom values
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f custom-values.yaml
```

### Custom Values File

```yaml
# custom-values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests:
        memory: 512Mi
        cpu: 250m
      limits:
        memory: 2Gi
        cpu: 1000m
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

grafana:
  adminPassword: "your-secure-password"
  persistence:
    enabled: true
    size: 10Gi

  # Configure default datasource
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://prometheus-kube-prometheus-prometheus:9090
          isDefault: true

# Node exporter for host metrics
nodeExporter:
  enabled: true

# kube-state-metrics for K8s metrics
kubeStateMetrics:
  enabled: true
```

### Access Services

```bash
# Port forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Port forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Port forward Alertmanager
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
```

### Verify Installation

```bash
# Check pods
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# Check Prometheus targets
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090/targets
```

---

## AKS-Specific Installation

### Enable Azure Monitor (Alternative)

```bash
# Enable Container Insights (Azure native)
az aks enable-addons \
  --name aks-myapp-prod \
  --resource-group rg-myapp-prod \
  --addons monitoring
```

### Prometheus on AKS

```bash
# Create AKS cluster with monitoring addon
az aks create \
  --name aks-myapp-prod \
  --resource-group rg-myapp-prod \
  --enable-managed-identity \
  --node-count 3 \
  --enable-addons monitoring

# Get credentials
az aks get-credentials --name aks-myapp-prod --resource-group rg-myapp-prod

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=15d
```

---

## Verify Installation

### Health Checks

```bash
# Prometheus health
curl -s http://localhost:9090/-/healthy

# Prometheus ready
curl -s http://localhost:9090/-/ready

# Check targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# Check config
curl -s http://localhost:9090/api/v1/status/config | jq '.data.yaml' | head -20
```

### Web UI Endpoints

```
┌────────────────────────────────────────────────────────────┐
│ Endpoint                │ Purpose                          │
├─────────────────────────┼──────────────────────────────────┤
│ http://localhost:9090   │ Prometheus Web UI                │
│ /graph                  │ Query interface                  │
│ /targets                │ Scrape targets status            │
│ /config                 │ Current configuration            │
│ /rules                  │ Alerting/recording rules         │
│ /alerts                 │ Active alerts                    │
│ /status                 │ Runtime information              │
│ /-/healthy              │ Health check                     │
│ /-/ready                │ Readiness check                  │
│ /-/reload               │ Reload config (POST)             │
└─────────────────────────┴──────────────────────────────────┘
```

---

## Summary

| Method | Command | Best For |
|--------|---------|----------|
| **Binary** | `./prometheus --config.file=...` | Production VMs |
| **Docker** | `docker run prom/prometheus` | Quick testing |
| **Compose** | `docker-compose up` | Dev environments |
| **Helm** | `helm install prometheus-community/kube-prometheus-stack` | Kubernetes |

### Quick Start Commands

```bash
# Docker (quickest)
docker run -p 9090:9090 prom/prometheus

# Docker Compose
docker-compose up -d

# Kubernetes
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# Verify
curl http://localhost:9090/-/healthy
```
