# ELK Alerting

## Alerting Options

```
┌─────────────────────────────────────────────────────────────┐
│                  ELK ALERTING OPTIONS                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Kibana Alerting (Recommended)                          │
│     └── Built-in, easy to configure via UI                 │
│     └── Available in Basic license (free)                  │
│                                                             │
│  2. Elasticsearch Watcher                                   │
│     └── Powerful, API-driven                               │
│     └── Requires Gold license or above                     │
│                                                             │
│  3. ElastAlert2 (Open Source)                              │
│     └── Python-based, flexible                             │
│     └── Community maintained                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Kibana Alerting

### Create Alert Rule via UI

```
1. Go to Stack Management → Rules and Connectors
2. Click "Create rule"
3. Select rule type:
   - Elasticsearch query
   - Index threshold
   - Log threshold
4. Configure conditions
5. Add actions (connectors)
6. Save rule
```

### Rule Types

| Type | Use Case |
|------|----------|
| **Elasticsearch query** | Custom ES queries |
| **Index threshold** | Count-based alerts |
| **Log threshold** | Log rate monitoring |

---

## Kibana Alert Examples

### High Error Rate Alert

```yaml
# Via Kibana UI
Name: High Error Rate
Rule type: Elasticsearch query
Index: logs-*

Query:
{
  "bool": {
    "must": [
      { "term": { "level": "ERROR" } },
      { "range": { "@timestamp": { "gte": "now-5m" } } }
    ]
  }
}

Threshold: > 100 (errors in 5 minutes)
Check every: 1 minute
Actions: Email, Slack
```

### Service Down Alert

```yaml
Name: Service Down
Rule type: Index threshold
Index: logs-*

Group by: service
Condition: count() IS BELOW 1
Time field: @timestamp
Time window: 5 minutes

# This alerts if no logs from a service in 5 minutes
```

### Log Spike Alert

```yaml
Name: Log Spike Detected
Rule type: Elasticsearch query
Index: logs-*

Query:
{
  "bool": {
    "must": [
      { "range": { "@timestamp": { "gte": "now-5m" } } }
    ]
  }
}

Threshold: > 10000 (logs in 5 minutes)
# Much higher than normal baseline
```

---

## Connectors (Actions)

### Email Connector

```
1. Stack Management → Rules and Connectors → Connectors
2. Create connector → Email
3. Configure:
   - Name: Email Alerts
   - Sender: alerts@example.com
   - Service: Gmail / Custom SMTP
   - Host: smtp.gmail.com
   - Port: 587
   - User/Password: ...
```

### Slack Connector

```
1. Create connector → Slack
2. Configure:
   - Name: Slack Alerts
   - Webhook URL: https://hooks.slack.com/services/XXX/YYY/ZZZ
```

### Webhook Connector

```
1. Create connector → Webhook
2. Configure:
   - Name: Custom Webhook
   - URL: https://my-service.example.com/alerts
   - Method: POST
   - Headers: Content-Type: application/json
```

### PagerDuty Connector

```
1. Create connector → PagerDuty
2. Configure:
   - Name: PagerDuty Critical
   - API URL: https://events.pagerduty.com/v2/enqueue
   - Routing Key: your-integration-key
```

---

## Elasticsearch Watcher

### Watcher Structure

```json
{
  "trigger": { ... },
  "input": { ... },
  "condition": { ... },
  "transform": { ... },
  "actions": { ... }
}
```

### Create Watch via API

```bash
PUT _watcher/watch/high_error_rate
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                { "term": { "level": "ERROR" } },
                { "range": { "@timestamp": { "gte": "now-5m" } } }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total.value": {
        "gte": 100
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": ["ops@example.com"],
        "subject": "High Error Rate Alert",
        "body": {
          "text": "{{ctx.payload.hits.total.value}} errors in the last 5 minutes"
        }
      }
    },
    "notify_slack": {
      "webhook": {
        "scheme": "https",
        "host": "hooks.slack.com",
        "port": 443,
        "method": "post",
        "path": "/services/XXX/YYY/ZZZ",
        "body": "{\"text\": \"High error rate: {{ctx.payload.hits.total.value}} errors\"}"
      }
    }
  }
}
```

### Watcher Examples

#### Disk Space Alert

```json
PUT _watcher/watch/disk_space_low
{
  "trigger": {
    "schedule": { "interval": "5m" }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["metricbeat-*"],
        "body": {
          "size": 1,
          "query": {
            "bool": {
              "must": [
                { "range": { "@timestamp": { "gte": "now-5m" } } },
                { "range": { "system.filesystem.used.pct": { "gte": 0.85 } } }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": { "ctx.payload.hits.total.value": { "gt": 0 } }
  },
  "actions": {
    "notify": {
      "webhook": {
        "method": "post",
        "url": "https://my-webhook.example.com/alerts",
        "body": "{\"alert\": \"Disk space low\", \"details\": \"{{ctx.payload.hits.hits.0._source}}\"}"
      }
    }
  }
}
```

#### No Logs Alert (Service Down)

```json
PUT _watcher/watch/service_no_logs
{
  "trigger": {
    "schedule": { "interval": "5m" }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                { "term": { "service": "critical-service" } },
                { "range": { "@timestamp": { "gte": "now-10m" } } }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": { "ctx.payload.hits.total.value": { "lt": 1 } }
  },
  "actions": {
    "pagerduty": {
      "webhook": {
        "method": "post",
        "url": "https://events.pagerduty.com/v2/enqueue",
        "headers": { "Content-Type": "application/json" },
        "body": "{\"routing_key\": \"xxx\", \"event_action\": \"trigger\", \"payload\": {\"summary\": \"No logs from critical-service\", \"severity\": \"critical\", \"source\": \"elasticsearch\"}}"
      }
    }
  }
}
```

### Manage Watches

```bash
# List all watches
GET _watcher/_stats

# Get watch details
GET _watcher/watch/high_error_rate

# Execute watch manually
POST _watcher/watch/high_error_rate/_execute

# Deactivate watch
PUT _watcher/watch/high_error_rate/_deactivate

# Activate watch
PUT _watcher/watch/high_error_rate/_activate

# Delete watch
DELETE _watcher/watch/high_error_rate
```

---

## ElastAlert2 (Open Source)

### Installation

```bash
# Docker
docker run -d \
  --name elastalert \
  -v $(pwd)/config.yaml:/opt/elastalert/config.yaml \
  -v $(pwd)/rules:/opt/elastalert/rules \
  jertel/elastalert2:latest

# Pip
pip install elastalert2
```

### Configuration

```yaml
# config.yaml
es_host: elasticsearch
es_port: 9200
es_username: elastic
es_password: changeme

rules_folder: /opt/elastalert/rules
run_every:
  minutes: 1
buffer_time:
  minutes: 15

alert_time_limit:
  days: 2
```

### Rule Example

```yaml
# rules/high_error_rate.yaml
name: High Error Rate
type: frequency
index: logs-*

num_events: 100
timeframe:
  minutes: 5

filter:
  - term:
      level: ERROR

alert:
  - slack

slack_webhook_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
slack_channel: "#alerts"
slack_username: "ElastAlert"
```

### Rule Types

| Type | Description |
|------|-------------|
| `frequency` | X events in Y time |
| `spike` | Sudden increase |
| `flatline` | No events for period |
| `blacklist` | Contains forbidden values |
| `whitelist` | Missing expected values |
| `any` | Any match triggers |
| `change` | Field value changes |

---

## Alert Best Practices

### DO

```
✓ Set appropriate thresholds based on baselines
✓ Use time windows that make sense (5-15 min)
✓ Include useful context in alerts
✓ Route critical alerts to PagerDuty
✓ Route warnings to Slack/Email
✓ Test alerts before enabling
```

### DON'T

```
✗ Alert on every single error
✗ Set thresholds too low (alert fatigue)
✗ Forget to include runbook links
✗ Create duplicate alerts
✗ Page for non-critical issues
```

### Alert Message Template

```
[SEVERITY] Alert Name

What: Brief description of the issue
When: Timestamp
Where: Service/Component affected

Details:
- Error count: X
- Threshold: Y
- Time window: Z minutes

Runbook: https://wiki.example.com/runbooks/alert-name

Query: (link to Kibana discover with relevant query)
```

---

## Summary

| Method | License | Best For |
|--------|---------|----------|
| **Kibana Alerting** | Basic (Free) | Simple alerts |
| **Watcher** | Gold+ | Complex conditions |
| **ElastAlert2** | Open Source | Flexible, free |

### Quick Setup

```bash
# Kibana Alerting (via UI)
Stack Management → Rules and Connectors → Create Rule

# Watcher (via API)
PUT _watcher/watch/my-watch { ... }

# ElastAlert2 (Docker)
docker run jertel/elastalert2 -v rules:/opt/elastalert/rules
```
