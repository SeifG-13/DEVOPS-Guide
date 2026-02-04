# Alertmanager

## What is Alertmanager?

Alertmanager handles alerts sent by Prometheus and routes them to the appropriate receivers.

```
┌─────────────────────────────────────────────────────────────┐
│                    ALERTMANAGER FLOW                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐                                          │
│  │  Prometheus  │                                          │
│  │              │───────────────┐                          │
│  │  Alert Rules │               │                          │
│  └──────────────┘               ▼                          │
│                        ┌──────────────────┐                │
│                        │   Alertmanager   │                │
│                        │                  │                │
│                        │ • Deduplicate    │                │
│                        │ • Group          │                │
│                        │ • Route          │                │
│                        │ • Silence        │                │
│                        │ • Inhibit        │                │
│                        └────────┬─────────┘                │
│                                 │                          │
│          ┌──────────────────────┼──────────────────────┐   │
│          ▼                      ▼                      ▼   │
│    ┌──────────┐          ┌──────────┐          ┌──────────┐│
│    │  Email   │          │  Slack   │          │ PagerDuty││
│    └──────────┘          └──────────┘          └──────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Installation

### Docker

```bash
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  -v $(pwd)/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager:latest \
  --config.file=/etc/alertmanager/alertmanager.yml
```

### Docker Compose

```yaml
services:
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://alertmanager.example.com'

volumes:
  alertmanager-data:
```

### Binary Installation

```bash
# Download
ALERTMANAGER_VERSION="0.26.0"
wget https://github.com/prometheus/alertmanager/releases/download/v${ALERTMANAGER_VERSION}/alertmanager-${ALERTMANAGER_VERSION}.linux-amd64.tar.gz

# Extract and install
tar xvfz alertmanager-${ALERTMANAGER_VERSION}.linux-amd64.tar.gz
sudo mv alertmanager-${ALERTMANAGER_VERSION}.linux-amd64/alertmanager /usr/local/bin/
sudo mv alertmanager-${ALERTMANAGER_VERSION}.linux-amd64/amtool /usr/local/bin/

# Create directories
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
```

---

## Configuration Structure

```yaml
# alertmanager.yml
global:
  # Global settings
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'

# Templates for notifications
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'

# Receivers (notification channels)
receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

# Inhibition rules
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

---

## Routing

### Basic Routing

```yaml
route:
  # Default receiver
  receiver: 'default'

  # Labels to group alerts by
  group_by: ['alertname', 'cluster', 'service']

  # Wait before sending initial notification
  group_wait: 30s

  # Wait before sending updated notification
  group_interval: 5m

  # Wait before resending notification
  repeat_interval: 4h
```

### Routing Tree

```yaml
route:
  receiver: 'default'
  group_by: ['alertname']
  routes:
    # Critical alerts go to PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: false  # Stop processing

    # Warning alerts go to Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'

    # Database alerts go to DBA team
    - match_re:
        alertname: '^(MySQL|PostgreSQL|Redis).*'
      receiver: 'dba-team'

    # Production alerts
    - match:
        environment: production
      receiver: 'ops-team'
      routes:
        # Nested route for critical prod alerts
        - match:
            severity: critical
          receiver: 'pagerduty'

    # Development alerts (during business hours only)
    - match:
        environment: development
      receiver: 'dev-team'
      mute_time_intervals:
        - offhours
```

### Match vs Match_re

```yaml
routes:
  # Exact match
  - match:
      severity: critical
      environment: production
    receiver: 'critical-prod'

  # Regex match
  - match_re:
      service: '^(api|web|worker)$'
      alertname: '.*Error.*'
    receiver: 'service-errors'
```

---

## Receivers

### Email

```yaml
receivers:
  - name: 'email-team'
    email_configs:
      - to: 'team@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'app-password'
        send_resolved: true
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}'
```

### Slack

```yaml
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#alerts'
        username: 'Alertmanager'
        icon_emoji: ':warning:'
        send_resolved: true
        title: '{{ .CommonAnnotations.summary }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Instance:* {{ .Labels.instance }}
          {{ end }}
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

### PagerDuty

```yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'your-pagerduty-service-key'
        severity: '{{ .CommonLabels.severity }}'
        description: '{{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
```

### Microsoft Teams

```yaml
receivers:
  - name: 'teams'
    webhook_configs:
      - url: 'https://outlook.office.com/webhook/xxx/IncomingWebhook/yyy/zzz'
        send_resolved: true
        http_config:
          bearer_token: ''
```

### Opsgenie

```yaml
receivers:
  - name: 'opsgenie'
    opsgenie_configs:
      - api_key: 'your-opsgenie-api-key'
        message: '{{ .CommonAnnotations.summary }}'
        priority: '{{ if eq .CommonLabels.severity "critical" }}P1{{ else }}P3{{ end }}'
        tags: 'prometheus,{{ .CommonLabels.alertname }}'
```

### Webhook (Custom)

```yaml
receivers:
  - name: 'webhook'
    webhook_configs:
      - url: 'http://my-webhook-handler:8080/alerts'
        send_resolved: true
        http_config:
          bearer_token: 'secret-token'
```

### Multiple Receivers

```yaml
receivers:
  - name: 'critical-alerts'
    email_configs:
      - to: 'oncall@example.com'
    slack_configs:
      - api_url: 'https://hooks.slack.com/...'
        channel: '#critical-alerts'
    pagerduty_configs:
      - service_key: 'xxx'
```

---

## Silences

Temporarily mute alerts.

### Create via Web UI

```
http://alertmanager:9093/#/silences/new
```

### Create via amtool

```bash
# Silence all alerts matching labels for 2 hours
amtool silence add alertname="HighCPU" instance="server1" \
  --comment="Maintenance window" \
  --author="admin" \
  --duration=2h

# Silence with regex
amtool silence add alertname=~".*CPU.*" \
  --comment="CPU maintenance" \
  --duration=1h

# List silences
amtool silence query

# Expire (remove) silence
amtool silence expire <silence-id>
```

### Create via API

```bash
curl -X POST http://alertmanager:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "alertname", "value": "HighCPU", "isRegex": false}
    ],
    "startsAt": "2024-01-15T10:00:00Z",
    "endsAt": "2024-01-15T12:00:00Z",
    "createdBy": "admin",
    "comment": "Planned maintenance"
  }'
```

---

## Inhibition Rules

Suppress alerts when other alerts are firing.

```yaml
inhibit_rules:
  # If critical alert fires, suppress warnings for same alertname
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']

  # If node is down, suppress all other alerts for that node
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '.+'
    equal: ['instance']

  # If cluster is down, suppress pod alerts
  - source_match:
      alertname: 'KubernetesClusterDown'
    target_match_re:
      alertname: 'KubernetesPod.*'
    equal: ['cluster']
```

---

## Time Intervals

### Mute During Off-Hours

```yaml
# Define time intervals
time_intervals:
  - name: offhours
    time_intervals:
      - weekdays: ['saturday', 'sunday']
      - times:
          - start_time: '00:00'
            end_time: '09:00'
          - start_time: '18:00'
            end_time: '24:00'
        weekdays: ['monday:friday']

  - name: businesshours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '18:00'
        weekdays: ['monday:friday']

# Use in routes
route:
  receiver: 'default'
  routes:
    - match:
        severity: warning
      receiver: 'slack-warnings'
      mute_time_intervals:
        - offhours  # Don't send warnings outside business hours

    - match:
        severity: critical
      receiver: 'pagerduty'
      # Critical alerts always sent (no mute)
```

---

## Templates

### Custom Slack Template

```yaml
# /etc/alertmanager/templates/slack.tmpl
{{ define "slack.myorg.title" -}}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
{{- end }}

{{ define "slack.myorg.text" -}}
{{ range .Alerts }}
*Alert:* {{ .Annotations.summary }}
*Description:* {{ .Annotations.description }}
*Severity:* `{{ .Labels.severity }}`
*Started:* {{ .StartsAt.Format "2006-01-02 15:04:05" }}
{{ if .Labels.instance }}*Instance:* {{ .Labels.instance }}{{ end }}
{{ end }}
{{- end }}
```

### Use Template

```yaml
templates:
  - '/etc/alertmanager/templates/*.tmpl'

receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/...'
        channel: '#alerts'
        title: '{{ template "slack.myorg.title" . }}'
        text: '{{ template "slack.myorg.text" . }}'
```

---

## High Availability

### Cluster Mode

```bash
# Node 1
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager2:9094 \
  --cluster.peer=alertmanager3:9094

# Node 2
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager1:9094 \
  --cluster.peer=alertmanager3:9094
```

### Prometheus Config for HA

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager1:9093
            - alertmanager2:9093
            - alertmanager3:9093
```

---

## amtool Commands

```bash
# Check config
amtool check-config alertmanager.yml

# Query alerts
amtool alert query

# Query silences
amtool silence query

# Add silence
amtool silence add alertname="Test" --duration=1h

# Expire silence
amtool silence expire <id>

# Test routing
amtool config routes test --config.file=alertmanager.yml \
  alertname=HighCPU severity=critical

# Show routing tree
amtool config routes show --config.file=alertmanager.yml
```

---

## Complete Example

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical production alerts
    - match:
        severity: critical
        environment: production
      receiver: 'pagerduty-critical'
      continue: true  # Also send to Slack

    # All critical alerts to Slack
    - match:
        severity: critical
      receiver: 'slack-critical'

    # Warning alerts
    - match:
        severity: warning
      receiver: 'slack-warnings'
      mute_time_intervals:
        - offhours

    # Database alerts
    - match_re:
        alertname: '^(MySQL|PostgreSQL|Redis).*'
      receiver: 'dba-team'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#critical-alerts'
        send_resolved: true
        color: 'danger'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#warnings'
        send_resolved: true
        color: 'warning'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'your-service-key'
        severity: 'critical'

  - name: 'dba-team'
    email_configs:
      - to: 'dba@example.com'
    slack_configs:
      - channel: '#dba-alerts'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']

  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '.+'
    equal: ['instance']

time_intervals:
  - name: offhours
    time_intervals:
      - weekdays: ['saturday', 'sunday']
      - times:
          - start_time: '00:00'
            end_time: '09:00'
          - start_time: '18:00'
            end_time: '24:00'
        weekdays: ['monday:friday']
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Routing** | Direct alerts to receivers |
| **Receivers** | Notification channels |
| **Silences** | Temporarily mute alerts |
| **Inhibition** | Suppress related alerts |
| **Templates** | Customize notifications |
| **Clustering** | High availability |
