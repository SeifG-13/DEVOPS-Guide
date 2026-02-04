# Grafana Alerting

## Grafana Unified Alerting

Grafana 9+ uses Unified Alerting, which combines Grafana-managed alerts with support for external alerting systems.

```
┌─────────────────────────────────────────────────────────────┐
│                 GRAFANA UNIFIED ALERTING                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Alert Rules                          │   │
│  │  ┌───────────────────┐  ┌───────────────────┐      │   │
│  │  │ Grafana-managed   │  │  Data source      │      │   │
│  │  │     alerts        │  │    alerts         │      │   │
│  │  │  (PromQL, SQL)    │  │  (Prometheus,     │      │   │
│  │  │                   │  │   Loki, etc.)     │      │   │
│  │  └─────────┬─────────┘  └─────────┬─────────┘      │   │
│  └────────────┼──────────────────────┼──────────────────┘   │
│               └──────────┬───────────┘                      │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Notification Policies                   │   │
│  │         (routing based on labels)                   │   │
│  └─────────────────────────┬───────────────────────────┘   │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Contact Points                         │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐   │   │
│  │  │ Email  │  │ Slack  │  │PagerDuty│ │ Teams  │   │   │
│  │  └────────┘  └────────┘  └────────┘  └────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Creating Alert Rules

### Via UI

```
1. Alerting → Alert rules → New alert rule
2. Configure:
   - Rule name
   - Query/condition
   - Evaluation behavior
   - Labels and annotations
3. Save rule
```

### Alert Rule Structure

```yaml
Rule name: High CPU Usage
Folder: Infrastructure
Group: Server Alerts
Evaluation interval: 1m

# Query and condition
Query A: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
Condition: A > 80

# Pending period (equivalent to Prometheus 'for')
Pending period: 5m

# Labels (for routing)
Labels:
  severity: warning
  team: ops

# Annotations (for notifications)
Annotations:
  summary: High CPU usage on {{ $labels.instance }}
  description: CPU usage is {{ $values.A }}%
  runbook_url: https://wiki.example.com/runbooks/high-cpu
```

---

## Alert Rule Examples

### Infrastructure Alerts

```yaml
# High Memory Usage
name: High Memory Usage
query: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
condition: > 85
pending: 5m
labels:
  severity: warning
annotations:
  summary: "High memory on {{ $labels.instance }}"
  description: "Memory usage is {{ printf \"%.1f\" $values.A }}%"

# Disk Space Low
name: Disk Space Low
query: (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) * 100
condition: > 85
pending: 5m
labels:
  severity: warning

# Node Down
name: Node Down
query: up{job="node-exporter"}
condition: == 0
pending: 1m
labels:
  severity: critical
```

### Application Alerts

```yaml
# High Error Rate
name: High Error Rate
query: |
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
  * 100
condition: > 5
pending: 5m
labels:
  severity: critical

# High Latency
name: High P99 Latency
query: |
  histogram_quantile(0.99,
    sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
  )
condition: > 2
pending: 5m
labels:
  severity: warning

# Service Down
name: Service Down
query: up{job="myapp"}
condition: == 0
pending: 1m
labels:
  severity: critical
```

---

## Contact Points

### Email

```yaml
Name: email-team
Type: Email
Addresses: team@example.com, oncall@example.com
Subject: [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
Message: |
  {{ range .Alerts }}
  Alert: {{ .Labels.alertname }}
  Instance: {{ .Labels.instance }}
  Summary: {{ .Annotations.summary }}
  {{ end }}
```

### Slack

```yaml
Name: slack-alerts
Type: Slack
Webhook URL: https://hooks.slack.com/services/XXX/YYY/ZZZ
Recipient: #alerts
Title: {{ .CommonLabels.alertname }}
Text: |
  {{ range .Alerts }}
  *Alert:* {{ .Annotations.summary }}
  *Severity:* {{ .Labels.severity }}
  *Instance:* {{ .Labels.instance }}
  {{ end }}
```

### PagerDuty

```yaml
Name: pagerduty-critical
Type: PagerDuty
Integration Key: your-pagerduty-integration-key
Severity: {{ .CommonLabels.severity }}
Class: {{ .CommonLabels.alertname }}
Component: {{ .CommonLabels.job }}
```

### Microsoft Teams

```yaml
Name: teams-alerts
Type: Microsoft Teams
Webhook URL: https://outlook.office.com/webhook/xxx
Title: Grafana Alert
Message: |
  **Alert:** {{ .CommonLabels.alertname }}
  **Status:** {{ .Status }}
  {{ range .Alerts }}
  - {{ .Annotations.summary }}
  {{ end }}
```

### Webhook (Custom)

```yaml
Name: custom-webhook
Type: Webhook
URL: https://my-service.example.com/alerts
HTTP Method: POST
Username: (optional)
Password: (optional)
Max Alerts: 10
```

---

## Notification Policies

### Default Policy

```yaml
# Root policy - catches all alerts
Default contact point: email-team
Group by: [alertname, grafana_folder]
Group wait: 30s
Group interval: 5m
Repeat interval: 4h
```

### Nested Policies (Routing)

```yaml
# Root policy
Default contact point: default
routes:
  # Critical alerts → PagerDuty
  - matcher: severity = critical
    contact_point: pagerduty-critical
    continue: true  # Also send to next matching route

  # All alerts also go to Slack
  - matcher: severity =~ "warning|critical"
    contact_point: slack-alerts

  # Database alerts → DBA team
  - matcher: alertname =~ "MySQL.*|PostgreSQL.*|Redis.*"
    contact_point: dba-slack

  # Specific team routing
  - matcher: team = backend
    contact_point: backend-slack

  - matcher: team = frontend
    contact_point: frontend-slack
```

### Matchers

```yaml
# Exact match
severity = critical

# Not equal
severity != info

# Regex match
alertname =~ ".*Error.*"

# Regex not match
instance !~ ".*test.*"
```

---

## Silences

### Create via UI

```
1. Alerting → Silences → New silence
2. Configure:
   - Duration (start/end time)
   - Matchers (which alerts to silence)
   - Comment
3. Create
```

### Silence Examples

```yaml
# Silence specific alert
Matcher: alertname = HighCPU
         instance = server1
Duration: 2 hours
Comment: Planned maintenance

# Silence all warnings
Matcher: severity = warning
Duration: 4 hours
Comment: Deployment window

# Silence alerts for namespace
Matcher: namespace = development
Duration: 1 hour
Comment: Dev environment testing
```

---

## Alert Rule Provisioning

### File-Based Provisioning

```yaml
# provisioning/alerting/alert-rules.yml
apiVersion: 1

groups:
  - orgId: 1
    name: Infrastructure Alerts
    folder: Infrastructure
    interval: 1m
    rules:
      - uid: high-cpu-alert
        title: High CPU Usage
        condition: A
        data:
          - refId: A
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: prometheus
            model:
              expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
              refId: A
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          summary: High CPU usage on {{ $labels.instance }}
          description: CPU usage is {{ printf "%.1f" $values.A }}%
        labels:
          severity: warning

      - uid: node-down-alert
        title: Node Down
        condition: A
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: up{job="node-exporter"} == 0
              refId: A
        for: 1m
        labels:
          severity: critical
```

### Contact Points Provisioning

```yaml
# provisioning/alerting/contact-points.yml
apiVersion: 1

contactPoints:
  - orgId: 1
    name: slack-alerts
    receivers:
      - uid: slack-receiver
        type: slack
        settings:
          url: https://hooks.slack.com/services/XXX/YYY/ZZZ
          recipient: "#alerts"
          title: '{{ template "slack.default.title" . }}'
          text: '{{ template "slack.default.text" . }}'

  - orgId: 1
    name: pagerduty-critical
    receivers:
      - uid: pagerduty-receiver
        type: pagerduty
        settings:
          integrationKey: your-key-here
          severity: '{{ .CommonLabels.severity }}'
```

### Notification Policies Provisioning

```yaml
# provisioning/alerting/notification-policies.yml
apiVersion: 1

policies:
  - orgId: 1
    receiver: slack-alerts
    group_by: ['alertname', 'grafana_folder']
    routes:
      - receiver: pagerduty-critical
        matchers:
          - severity = critical
        continue: true
      - receiver: dba-team
        matchers:
          - alertname =~ "MySQL.*|PostgreSQL.*"
```

---

## Templates

### Custom Message Templates

```yaml
# provisioning/alerting/templates.yml
apiVersion: 1

templates:
  - orgId: 1
    name: custom_slack_template
    template: |
      {{ define "custom.slack.title" }}
      [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
      {{ end }}

      {{ define "custom.slack.text" }}
      {{ range .Alerts }}
      *Alert:* {{ .Annotations.summary }}
      *Severity:* `{{ .Labels.severity }}`
      *Instance:* {{ .Labels.instance }}
      *Started:* {{ .StartsAt.Format "2006-01-02 15:04:05" }}
      {{ if .Annotations.runbook_url }}
      *Runbook:* {{ .Annotations.runbook_url }}
      {{ end }}
      ---
      {{ end }}
      {{ end }}
```

### Use Custom Template

```yaml
# In contact point configuration
Title: '{{ template "custom.slack.title" . }}'
Text: '{{ template "custom.slack.text" . }}'
```

---

## Alert States

```
┌────────────────────────────────────────────────────────────┐
│                    ALERT STATES                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Normal    → Condition false, no alert                    │
│  Pending   → Condition true, waiting for 'for' duration   │
│  Firing    → Condition true for full 'for' duration       │
│  Resolved  → Was firing, condition now false              │
│  NoData    → No data returned from query                  │
│  Error     → Query execution error                        │
│                                                            │
│  Timeline:                                                 │
│  ─────────────────────────────────────────────────────    │
│  Normal → Pending → Firing → Resolved → Normal            │
│           (for: 5m)                                        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Configuring No Data / Error Handling

```yaml
# In alert rule configuration
noDataState: NoData     # or Alerting, OK, KeepLast
execErrState: Error     # or Alerting, OK, KeepLast

# NoData options:
# - NoData: Sets state to NoData
# - Alerting: Treats as firing alert
# - OK: Treats as resolved
# - KeepLast: Keeps previous state

# Error options:
# - Error: Sets state to Error
# - Alerting: Treats as firing alert
# - OK: Treats as resolved
# - KeepLast: Keeps previous state
```

---

## Testing Alerts

### Send Test Notification

```
1. Alerting → Contact points
2. Click "Test" next to contact point
3. Verify notification received
```

### Evaluate Alert Rule

```
1. Alerting → Alert rules
2. Click on rule name
3. View "State history" tab
4. Check "Query and condition" evaluation
```

---

## Integration with Alertmanager

### Use External Alertmanager

```yaml
# grafana.ini
[unified_alerting]
enabled = true

[alerting]
enabled = false

# Point to external Alertmanager
[unified_alerting.alertmanager]
url = http://alertmanager:9093
```

### Configure Alertmanager Data Source

```yaml
# As data source for viewing Alertmanager alerts
Name: Alertmanager
Type: Alertmanager
URL: http://alertmanager:9093
```

---

## Best Practices

### DO

```
✓ Set appropriate 'for' duration (avoid flapping)
✓ Use meaningful labels for routing
✓ Include runbook URLs in annotations
✓ Test contact points before relying on them
✓ Use silences for planned maintenance
✓ Group related alerts
```

### DON'T

```
✗ Alert on every minor fluctuation
✗ Use 'for: 0' (causes alert storms)
✗ Ignore NoData states
✗ Create overlapping alert rules
✗ Skip annotations (hard to debug)
✗ Page for warnings
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Alert Rules** | Define conditions that trigger alerts |
| **Contact Points** | Where to send notifications |
| **Notification Policies** | How to route alerts |
| **Silences** | Temporarily mute alerts |
| **Templates** | Customize notification format |

### Quick Setup

```
1. Create contact point (Slack/Email/etc.)
2. Set default notification policy
3. Create alert rules
4. Test notifications
5. Add silences for maintenance
```
