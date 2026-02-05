# Kibana Dashboards

## Kibana Overview

Kibana is the visualization and management UI for Elasticsearch.

```
┌─────────────────────────────────────────────────────────────┐
│                    KIBANA FEATURES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Analytics                                                  │
│  ├── Discover    - Search and explore logs                 │
│  ├── Dashboard   - Combine visualizations                  │
│  ├── Visualize   - Create charts and graphs               │
│  └── Lens        - Drag-and-drop visualization            │
│                                                             │
│  Management                                                 │
│  ├── Stack Management - Index patterns, ILM               │
│  ├── Dev Tools       - Console for ES queries             │
│  └── Alerts          - Rule-based alerting                │
│                                                             │
│  Observability                                              │
│  ├── Logs        - Log explorer                            │
│  ├── Metrics     - Infrastructure monitoring              │
│  └── APM         - Application performance                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Initial Setup

### Create Data View (Index Pattern)

```
1. Go to Stack Management → Data Views
2. Click "Create data view"
3. Enter:
   - Name: logs-*
   - Index pattern: logs-*
   - Timestamp field: @timestamp
4. Click "Save data view"
```

### Via API

```bash
# Create data view
POST /api/data_views/data_view
{
  "data_view": {
    "title": "logs-*",
    "name": "Application Logs",
    "timeFieldName": "@timestamp"
  }
}
```

---

## Discover

Search and explore logs in real-time.

### Basic Search

```
# Kibana Query Language (KQL)
level: ERROR
service: "auth-service"
message: "login failed"
response_time > 1000
@timestamp >= "2024-01-15" and @timestamp < "2024-01-16"
```

### KQL Syntax

```
# Field exists
level: *

# Exact match
level: "ERROR"

# Wildcard
message: error*
service: auth-*

# Boolean operators
level: ERROR and service: api
level: ERROR or level: FATAL
not level: DEBUG

# Range queries
response_time >= 1000
@timestamp > "2024-01-15"

# Nested fields
user.email: "john@example.com"

# Combine
(level: ERROR or level: WARN) and service: api and response_time > 500
```

### Lucene Query Syntax

```
# Alternative to KQL
level:ERROR AND service:api
message:"connection refused"
response_time:[1000 TO *]
message:/error.*/
```

### Saved Searches

```
1. Run a search in Discover
2. Click "Save" button
3. Enter name: "Production Errors"
4. Click Save
5. Access from Discover → Open
```

---

## Visualizations

### Types of Visualizations

| Type | Use Case |
|------|----------|
| **Lens** | Drag-and-drop (recommended) |
| **Area** | Trends over time |
| **Bar** | Comparisons |
| **Line** | Time series |
| **Pie** | Proportions |
| **Metric** | Single values |
| **Data Table** | Tabular data |
| **Heat Map** | Distribution |
| **Tag Cloud** | Word frequency |

### Create with Lens

```
1. Go to Visualize → Create visualization
2. Select "Lens"
3. Choose data view: logs-*
4. Drag fields to build:
   - X-axis: @timestamp
   - Y-axis: Count
   - Break down by: level
5. Customize options
6. Save visualization
```

### Common Visualizations

#### Logs Over Time (Area Chart)

```
Type: Area
X-axis: @timestamp (Date Histogram, interval: auto)
Y-axis: Count
Split series: level
```

#### Error Count by Service (Bar Chart)

```
Type: Bar (horizontal)
Y-axis: service (Terms)
X-axis: Count
Filter: level: ERROR
```

#### Response Time Percentiles (Line)

```
Type: Line
X-axis: @timestamp (Date Histogram)
Y-axis: Percentiles (field: response_time, percentiles: 50, 90, 99)
```

#### Log Level Distribution (Pie)

```
Type: Pie
Slice by: level (Terms)
Metric: Count
```

#### Error Rate Metric

```
Type: Metric
Metric: Count
Filter: level: ERROR
Compare to previous period: enabled
```

---

## Dashboards

### Create Dashboard

```
1. Go to Dashboard → Create dashboard
2. Click "Add panel"
3. Select existing visualization or create new
4. Arrange panels (drag and resize)
5. Click "Save"
6. Enter name: "Application Overview"
```

### Dashboard Layout Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│  Row 1: Key Metrics                                         │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │
│  │ Total  │ │ Errors │ │ Warn   │ │ P99    │ │ Error  │  │
│  │ Logs   │ │ Count  │ │ Count  │ │ Latency│ │ Rate % │  │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Row 2: Trends                                              │
│  ┌─────────────────────────────┐ ┌─────────────────────────┐│
│  │     Logs Over Time          │ │   Error Rate Trend     ││
│  │     (by level)              │ │                         ││
│  └─────────────────────────────┘ └─────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│  Row 3: Details                                             │
│  ┌─────────────────────────────┐ ┌─────────────────────────┐│
│  │   Errors by Service         │ │   Top Error Messages   ││
│  │   (bar chart)               │ │   (table)              ││
│  └─────────────────────────────┘ └─────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│  Row 4: Log Stream                                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │   Recent Logs (Saved Search embed)                      ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Dashboard Controls

```
1. Add filter controls:
   - Click "Controls" → "Add control"
   - Select field: service
   - Type: Options list
2. Add time filter control
3. Users can filter dashboard dynamically
```

---

## TSVB (Time Series Visual Builder)

Advanced time series visualization.

```
1. Visualize → Create → TSVB
2. Configure:
   - Panel type: Time Series
   - Index pattern: logs-*
   - Metrics: Count, Percentile, etc.
   - Group by: Terms
3. Add multiple series
4. Configure annotations for events
```

### TSVB Error Rate Example

```
Series 1: Error Count
- Aggregation: Count
- Filter: level: ERROR

Series 2: Total Count
- Aggregation: Count

Panel Options:
- Use pipeline aggregation for ratio
```

---

## Saved Objects

### Export Dashboards

```bash
# Via API
GET /api/saved_objects/_export
{
  "type": ["dashboard"],
  "objects": [
    { "type": "dashboard", "id": "dashboard-id" }
  ],
  "includeReferencesDeep": true
}

# Via UI
Stack Management → Saved Objects → Select → Export
```

### Import Dashboards

```bash
# Via API
POST /api/saved_objects/_import
# Upload NDJSON file

# Via UI
Stack Management → Saved Objects → Import
```

### Sharing Dashboards

```
1. Open dashboard
2. Click "Share" button
3. Options:
   - Get link (with/without time range)
   - Embed code
   - PDF/PNG report (requires subscription)
```

---

## Dev Tools (Console)

Run Elasticsearch queries directly.

### Common Queries

```bash
# Cluster health
GET _cluster/health

# List indices
GET _cat/indices?v

# Search
GET logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "level": "ERROR" } }
      ],
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "size": 100
}

# Aggregation
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "by_service": {
      "terms": { "field": "service", "size": 10 }
    }
  }
}

# Index mapping
GET logs-*/_mapping
```

---

## Index Lifecycle Management (ILM)

Manage index lifecycle from Kibana.

### Create ILM Policy

```
1. Stack Management → Index Lifecycle Policies
2. Create policy:
   Name: logs-policy

   Hot phase:
   - Rollover: 50GB or 30 days
   - Priority: 100

   Warm phase (after 7 days):
   - Move to warm nodes
   - Force merge: 1 segment
   - Priority: 50

   Cold phase (after 30 days):
   - Move to cold nodes
   - Priority: 0

   Delete phase (after 90 days):
   - Delete index
```

### Apply to Index Template

```
Stack Management → Index Management → Index Templates
Edit template → Add ILM policy
```

---

## Spaces

Organize content by team or environment.

```
1. Stack Management → Spaces
2. Create space:
   - Name: Production
   - Features: Discover, Dashboard, Visualize
3. Assign dashboards to spaces
4. Control user access per space
```

---

## Dashboard Examples

### Operations Dashboard

```json
// Panels to include:
{
  "panels": [
    "Total Log Count (Metric)",
    "Error Count (Metric)",
    "Logs by Level Over Time (Area)",
    "Top Services by Error Count (Bar)",
    "Response Time P99 (Line)",
    "Recent Errors (Saved Search)"
  ]
}
```

### Service Health Dashboard

```json
{
  "panels": [
    "Service Status (Traffic Light)",
    "Request Rate by Service (Line)",
    "Error Rate by Service (Line)",
    "Latency by Service (Heatmap)",
    "Service Dependencies (Table)"
  ]
}
```

---

## Reporting

### On-Demand Reports

```
1. Open dashboard
2. Share → PDF Reports / PNG
3. Configure: layout, time range
4. Generate
```

### Scheduled Reports

```
1. Stack Management → Reporting
2. Create scheduled report
3. Set schedule (daily, weekly)
4. Configure recipients
```

---

## Summary

| Feature | Purpose |
|---------|---------|
| **Discover** | Search and explore |
| **Lens** | Drag-and-drop viz |
| **Dashboard** | Combine panels |
| **Dev Tools** | ES queries |
| **ILM** | Index lifecycle |

### Quick Access URLs

```
Discover:   /app/discover
Dashboard:  /app/dashboards
Visualize:  /app/visualize
Dev Tools:  /app/dev_tools
Management: /app/management
```
