# Grafana Dashboards

## Dashboard Basics

```
┌─────────────────────────────────────────────────────────────┐
│                    DASHBOARD STRUCTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Dashboard                                          │   │
│  │  ├── Variables (dropdowns for filtering)            │   │
│  │  ├── Row 1                                          │   │
│  │  │   ├── Panel (Graph)                             │   │
│  │  │   ├── Panel (Stat)                              │   │
│  │  │   └── Panel (Gauge)                             │   │
│  │  ├── Row 2                                          │   │
│  │  │   ├── Panel (Table)                             │   │
│  │  │   └── Panel (Heatmap)                           │   │
│  │  └── Annotations (event markers)                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Creating Dashboards

### Via UI

```
1. Click "+" → "New Dashboard"
2. Click "Add new panel"
3. Configure query, visualization, and options
4. Click "Apply"
5. Save dashboard (Ctrl+S)
```

### Panel Types

| Panel | Use Case |
|-------|----------|
| **Time series** | Trends over time |
| **Stat** | Single value with optional sparkline |
| **Gauge** | Progress toward a goal |
| **Bar gauge** | Comparative values |
| **Table** | Tabular data |
| **Heatmap** | Distribution over time |
| **Pie chart** | Proportions |
| **Logs** | Log entries |
| **Alert list** | Active alerts |

---

## Variables (Templating)

### Query Variable

```yaml
# Variable configuration
Name: instance
Label: Instance
Type: Query
Data source: Prometheus
Query: label_values(up, instance)
Refresh: On Dashboard Load
Multi-value: true
Include All option: true
```

### Using Variables in Queries

```promql
# Filter by variable
rate(http_requests_total{instance="$instance"}[5m])

# Multiple values (regex)
rate(http_requests_total{instance=~"$instance"}[5m])

# All values
rate(http_requests_total{instance=~".*"}[5m])
```

### Common Variable Types

```yaml
# Query - from data source
Type: Query
Query: label_values(node_cpu_seconds_total, instance)

# Custom - static values
Type: Custom
Values: prod,staging,dev

# Interval - time intervals
Type: Interval
Values: 1m,5m,10m,30m,1h

# Data source - select data source
Type: Data source
Type filter: prometheus
```

### Chained Variables

```yaml
# First variable: namespace
Query: label_values(kube_pod_info, namespace)

# Second variable: pod (filtered by namespace)
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
```

---

## Query Examples

### Basic Queries

```promql
# Current value
http_requests_total{job="myapp"}

# Rate per second
rate(http_requests_total[5m])

# Sum by label
sum by (method) (rate(http_requests_total[5m]))
```

### For Stat Panels

```promql
# Total requests
sum(increase(http_requests_total[$__range]))

# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Average latency
rate(http_request_duration_seconds_sum[5m])
/
rate(http_request_duration_seconds_count[5m])
```

### For Time Series

```promql
# Request rate by status
sum by (status) (rate(http_requests_total[5m]))

# CPU usage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

### For Gauges

```promql
# CPU percentage
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Disk usage percentage
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Memory percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

---

## Panel Configuration

### Time Series Options

```yaml
Panel Options:
  Title: Request Rate
  Description: HTTP requests per second

Graph Styles:
  Style: Lines
  Line interpolation: Smooth
  Line width: 2
  Fill opacity: 10
  Point size: 5

Axis:
  Placement: Auto
  Label: Requests/sec
  Scale: Linear

Legend:
  Mode: Table
  Placement: Bottom
  Values: [Last, Max, Mean]

Thresholds:
  - value: 0, color: green
  - value: 100, color: yellow
  - value: 500, color: red
```

### Stat Panel Options

```yaml
Panel Options:
  Title: Error Rate

Value Options:
  Calculation: Last
  Fields: Numeric

Stat Styles:
  Orientation: Auto
  Color mode: Background
  Graph mode: Area
  Text mode: Auto

Thresholds:
  Mode: Percentage
  Steps:
    - value: 0, color: green
    - value: 1, color: yellow
    - value: 5, color: red
```

### Gauge Options

```yaml
Panel Options:
  Title: CPU Usage

Value Options:
  Calculation: Last

Gauge:
  Show threshold labels: true
  Show threshold markers: true
  Min: 0
  Max: 100

Thresholds:
  - value: 0, color: green
  - value: 70, color: yellow
  - value: 90, color: red
```

---

## Importing Dashboards

### From Grafana.com

```
1. Go to grafana.com/grafana/dashboards
2. Find dashboard and copy ID
3. In Grafana: Dashboards → Import
4. Enter ID and click Load
5. Select data source
6. Click Import
```

### Popular Dashboard IDs

| ID | Name | For |
|----|------|-----|
| 1860 | Node Exporter Full | Linux servers |
| 3662 | Prometheus 2.0 Overview | Prometheus |
| 7362 | MySQL Overview | MySQL |
| 9628 | PostgreSQL Database | PostgreSQL |
| 763 | Redis Dashboard | Redis |
| 11074 | Node Exporter for Prometheus | Linux (simpler) |
| 315 | Kubernetes cluster monitoring | K8s |
| 6417 | Kubernetes Cluster | K8s |

### From JSON File

```bash
# Export dashboard
curl -u admin:admin123 \
  http://localhost:3000/api/dashboards/uid/abc123 \
  | jq '.dashboard' > dashboard.json

# Import dashboard
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  http://localhost:3000/api/dashboards/db \
  -d @dashboard.json
```

---

## Dashboard Provisioning

### Configuration

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: 'Provisioned'
    folderUid: 'provisioned'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

### Dashboard JSON Structure

```json
{
  "dashboard": {
    "id": null,
    "uid": "myapp-overview",
    "title": "My Application Overview",
    "tags": ["myapp", "production"],
    "timezone": "browser",
    "refresh": "30s",
    "schemaVersion": 38,
    "version": 1,
    "panels": [
      {
        "id": 1,
        "type": "stat",
        "title": "Request Rate",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m]))",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": null, "color": "green" },
                { "value": 100, "color": "yellow" },
                { "value": 500, "color": "red" }
              ]
            }
          }
        }
      }
    ],
    "templating": {
      "list": [
        {
          "name": "instance",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(up, instance)",
          "refresh": 1,
          "multi": true,
          "includeAll": true
        }
      ]
    }
  },
  "overwrite": true
}
```

---

## Building a Complete Dashboard

### Application Overview Dashboard

```json
{
  "dashboard": {
    "title": "Application Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 4, "x": 0, "y": 0 },
        "targets": [{
          "expr": "sum(rate(http_requests_total{job=\"$job\"}[5m]))"
        }],
        "fieldConfig": {
          "defaults": { "unit": "reqps" }
        }
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 4, "x": 4, "y": 0 },
        "targets": [{
          "expr": "sum(rate(http_requests_total{job=\"$job\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{job=\"$job\"}[5m])) * 100"
        }],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                { "value": null, "color": "green" },
                { "value": 1, "color": "yellow" },
                { "value": 5, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "title": "P99 Latency",
        "type": "stat",
        "gridPos": { "h": 4, "w": 4, "x": 8, "y": 0 },
        "targets": [{
          "expr": "histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"$job\"}[5m])))"
        }],
        "fieldConfig": {
          "defaults": { "unit": "s" }
        }
      },
      {
        "title": "Request Rate Over Time",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
        "targets": [{
          "expr": "sum by (status) (rate(http_requests_total{job=\"$job\"}[5m]))",
          "legendFormat": "{{status}}"
        }]
      },
      {
        "title": "Latency Percentiles",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 4 },
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"$job\"}[5m])))",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.90, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"$job\"}[5m])))",
            "legendFormat": "p90"
          },
          {
            "expr": "histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"$job\"}[5m])))",
            "legendFormat": "p99"
          }
        ]
      }
    ],
    "templating": {
      "list": [{
        "name": "job",
        "type": "query",
        "query": "label_values(http_requests_total, job)"
      }]
    }
  }
}
```

---

## Annotations

### Query-Based Annotations

```yaml
Name: Deployments
Data source: Prometheus
Query: changes(deployment_timestamp[5m]) > 0
Title: Deployment
Text: New deployment
Tags: deployment
```

### Manual Annotations

```
Right-click on graph → Add annotation
Enter title and description
Click Save
```

---

## Dashboard Best Practices

### Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Row 1: Key Metrics (Stats/Gauges)                         │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐        │
│  │ Req/ │  │Error │  │ P99  │  │ CPU  │  │ Mem  │        │
│  │  sec │  │ Rate │  │Latency│ │Usage │  │Usage │        │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘        │
├─────────────────────────────────────────────────────────────┤
│  Row 2: Trends (Time Series)                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐ │
│  │    Request Rate         │  │    Error Rate           │ │
│  │    [Graph over time]    │  │    [Graph over time]    │ │
│  └─────────────────────────┘  └─────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Row 3: Details (Tables/Heatmaps)                          │
│  ┌─────────────────────────────────────────────────────┐  │
│  │    Top Endpoints by Request Count                    │  │
│  │    [Table with endpoint, count, error rate]          │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Naming Conventions

```
Dashboard names:
- [Service] - Overview
- [Service] - Details
- [Service] - [Component]

Panel titles:
- Use clear, descriptive names
- Include units: "CPU Usage (%)"
- Specify time range if not obvious: "Errors (last 24h)"
```

### Colors

```yaml
Standard Colors:
- Green: Good / OK
- Yellow: Warning
- Red: Critical / Error
- Blue: Information

Use consistent colors across dashboards
```

---

## API Examples

### Create Dashboard

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  http://localhost:3000/api/dashboards/db \
  -d '{
    "dashboard": {
      "title": "New Dashboard",
      "panels": [],
      "schemaVersion": 38
    },
    "overwrite": false
  }'
```

### Get Dashboard

```bash
curl -u admin:admin123 \
  http://localhost:3000/api/dashboards/uid/abc123
```

### Search Dashboards

```bash
curl -u admin:admin123 \
  "http://localhost:3000/api/search?query=myapp"
```

---

## Summary

| Feature | Purpose |
|---------|---------|
| **Variables** | Dynamic filtering |
| **Panels** | Visualizations |
| **Rows** | Organize panels |
| **Annotations** | Mark events |
| **Provisioning** | GitOps dashboards |
| **Import** | Use community dashboards |
