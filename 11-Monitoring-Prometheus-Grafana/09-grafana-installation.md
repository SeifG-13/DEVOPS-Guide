# Grafana Installation

## What is Grafana?

Grafana is an open-source analytics and visualization platform for time series data.

```
┌─────────────────────────────────────────────────────────────┐
│                       GRAFANA                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  DATA SOURCES                        │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │   │
│  │  │Prometheus│  │   Loki   │  │InfluxDB │  ...     │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘         │   │
│  └───────┼─────────────┼─────────────┼───────────────┘   │
│          └─────────────┼─────────────┘                     │
│                        ▼                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    GRAFANA                          │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐      │   │
│  │  │ Dashboards│  │  Alerts   │  │  Explore  │      │   │
│  │  └───────────┘  └───────────┘  └───────────┘      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Installation Options

| Method | Best For |
|--------|----------|
| Docker | Quick setup, development |
| Docker Compose | Multi-container environments |
| Binary | Linux servers |
| Helm | Kubernetes |
| Grafana Cloud | Managed service |

---

## Docker Installation

### Simple Run

```bash
# Create volume for persistence
docker volume create grafana-data

# Run Grafana
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  grafana/grafana:latest

# Access at http://localhost:3000
# Default credentials: admin / admin
```

### With Environment Variables

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  -e GF_SECURITY_ADMIN_USER=admin \
  -e GF_SECURITY_ADMIN_PASSWORD=secure_password \
  -e GF_USERS_ALLOW_SIGN_UP=false \
  -e GF_SERVER_ROOT_URL=https://grafana.example.com \
  grafana/grafana:latest
```

---

## Docker Compose Installation

### Basic Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
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

volumes:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

### With Prometheus Stack

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    depends_on:
      - prometheus
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:

networks:
  monitoring:
```

---

## Binary Installation (Linux)

```bash
# Install dependencies
sudo apt-get install -y adduser libfontconfig1 musl

# Download
GRAFANA_VERSION="10.2.3"
wget https://dl.grafana.com/oss/release/grafana_${GRAFANA_VERSION}_amd64.deb

# Install
sudo dpkg -i grafana_${GRAFANA_VERSION}_amd64.deb

# Start service
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Check status
sudo systemctl status grafana-server

# View logs
sudo journalctl -u grafana-server -f
```

### Configuration File

```bash
# Main config file
sudo nano /etc/grafana/grafana.ini
```

---

## Kubernetes Installation (Helm)

### Using kube-prometheus-stack (Recommended)

```bash
# This installs Grafana with Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=secure_password
```

### Standalone Grafana

```bash
# Add Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set adminPassword=secure_password \
  --set persistence.enabled=true \
  --set persistence.size=10Gi
```

### Custom Values

```yaml
# grafana-values.yaml
adminUser: admin
adminPassword: secure_password

persistence:
  enabled: true
  size: 10Gi

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server:9090
        isDefault: true
        access: proxy

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default

dashboards:
  default:
    node-exporter:
      gnetId: 1860
      revision: 31
      datasource: Prometheus
```

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  -f grafana-values.yaml
```

### Access Grafana

```bash
# Port forward
kubectl port-forward -n monitoring svc/grafana 3000:80

# Or get LoadBalancer IP
kubectl get svc -n monitoring grafana

# Get admin password (if randomly generated)
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

---

## Data Source Configuration

### Via Provisioning (Recommended)

```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false

  - name: InfluxDB
    type: influxdb
    access: proxy
    url: http://influxdb:8086
    database: mydb
    user: admin
    secureJsonData:
      password: secret
```

### Via UI

```
1. Login to Grafana
2. Go to Configuration → Data Sources
3. Click "Add data source"
4. Select "Prometheus"
5. Enter URL: http://prometheus:9090
6. Click "Save & Test"
```

### Via API

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  http://localhost:3000/api/datasources \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true
  }'
```

---

## Dashboard Provisioning

### Auto-Load Dashboards

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    folderUid: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

### Directory Structure

```
grafana/
├── provisioning/
│   ├── datasources/
│   │   └── datasources.yml
│   └── dashboards/
│       └── dashboards.yml
└── dashboards/
    ├── node-exporter.json
    ├── docker.json
    └── application.json
```

---

## Configuration Options

### grafana.ini / Environment Variables

```ini
# /etc/grafana/grafana.ini

[server]
# Protocol (http, https, socket)
protocol = http

# The ip address to bind to
http_addr = 0.0.0.0

# The http port
http_port = 3000

# The public facing domain name
domain = grafana.example.com

# The full public facing url
root_url = https://grafana.example.com

[database]
# Database type (mysql, postgres, sqlite3)
type = sqlite3
path = grafana.db

[security]
# Default admin user
admin_user = admin

# Default admin password (change this!)
admin_password = admin

# Disable gravatar
disable_gravatar = true

# Secret key for signing
secret_key = SW2YcwTIb9zpOOhoPsMm

[users]
# Disable user signup
allow_sign_up = false

# Allow org admins to create users
allow_org_create = false

[auth.anonymous]
# Enable anonymous access
enabled = false

[smtp]
enabled = true
host = smtp.gmail.com:587
user = alerts@example.com
password = app_password
from_address = alerts@example.com
from_name = Grafana
```

### Environment Variables

```bash
# Equivalent environment variables (prefix with GF_)
GF_SERVER_HTTP_PORT=3000
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=secret
GF_USERS_ALLOW_SIGN_UP=false
GF_AUTH_ANONYMOUS_ENABLED=false
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.gmail.com:587
```

---

## Authentication

### Basic Auth

```ini
[auth.basic]
enabled = true
```

### OAuth (Azure AD)

```ini
[auth.azuread]
name = Azure AD
enabled = true
allow_sign_up = true
client_id = your-client-id
client_secret = your-client-secret
scopes = openid email profile
auth_url = https://login.microsoftonline.com/your-tenant-id/oauth2/v2.0/authorize
token_url = https://login.microsoftonline.com/your-tenant-id/oauth2/v2.0/token
allowed_domains = example.com
allowed_groups =
```

### OAuth (GitHub)

```ini
[auth.github]
enabled = true
allow_sign_up = true
client_id = your-github-client-id
client_secret = your-github-client-secret
scopes = user:email,read:org
auth_url = https://github.com/login/oauth/authorize
token_url = https://github.com/login/oauth/access_token
api_url = https://api.github.com/user
allowed_organizations = your-org
```

---

## Verify Installation

```bash
# Health check
curl http://localhost:3000/api/health

# Get Grafana version
curl http://localhost:3000/api/frontend/settings | jq '.buildInfo.version'

# Test data source
curl -u admin:admin123 http://localhost:3000/api/datasources/proxy/1/api/v1/query?query=up

# List dashboards
curl -u admin:admin123 http://localhost:3000/api/search
```

---

## Common Commands

```bash
# Docker
docker logs grafana
docker exec -it grafana grafana-cli plugins list

# Systemd
sudo systemctl status grafana-server
sudo systemctl restart grafana-server
sudo journalctl -u grafana-server -f

# Kubernetes
kubectl logs -n monitoring deployment/grafana
kubectl exec -n monitoring deployment/grafana -- grafana-cli plugins list
```

---

## Install Plugins

### Via CLI

```bash
# Docker
docker exec -it grafana grafana-cli plugins install grafana-piechart-panel
docker restart grafana

# Systemd
sudo grafana-cli plugins install grafana-piechart-panel
sudo systemctl restart grafana-server

# Kubernetes (via values)
plugins:
  - grafana-piechart-panel
  - grafana-clock-panel
```

### Popular Plugins

```
grafana-piechart-panel    - Pie chart
grafana-clock-panel       - Clock display
grafana-worldmap-panel    - World map
grafana-polystat-panel    - Polygon statistics
jdbranham-diagram-panel   - Flowchart diagrams
```

---

## Summary

| Installation | Command |
|--------------|---------|
| **Docker** | `docker run -p 3000:3000 grafana/grafana` |
| **Compose** | `docker-compose up -d` |
| **Helm** | `helm install grafana grafana/grafana` |
| **kube-prometheus-stack** | Includes Grafana automatically |

### Quick Start

```bash
# Fastest way to start
docker run -d -p 3000:3000 --name grafana grafana/grafana

# Access
open http://localhost:3000
# Login: admin / admin
```

### Default Ports

```
Grafana UI:  3000
Prometheus:  9090
Alertmanager: 9093
```
