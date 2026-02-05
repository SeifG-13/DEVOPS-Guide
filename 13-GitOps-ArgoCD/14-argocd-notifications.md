# ArgoCD Notifications

## Overview

ArgoCD Notifications sends alerts about application events to various channels.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NOTIFICATIONS FLOW                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ArgoCD Events                             │   │
│  │  • Sync started/completed/failed                            │   │
│  │  • Health status changed                                    │   │
│  │  • Application created/deleted                              │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │                                       │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Notifications Controller                        │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │   │
│  │  │ Triggers │  │Templates │  │ Services │                  │   │
│  │  │(when)    │  │(what)    │  │(where)   │                  │   │
│  │  └──────────┘  └──────────┘  └──────────┘                  │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │                                       │
│         ┌───────────────────┼───────────────────┐                  │
│         │                   │                   │                  │
│         ▼                   ▼                   ▼                  │
│   ┌──────────┐       ┌──────────┐       ┌──────────┐             │
│   │  Slack   │       │  Email   │       │ Webhook  │             │
│   └──────────┘       └──────────┘       └──────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Installation

Notifications controller is included in ArgoCD since v2.3.

```bash
# Verify notifications controller is running
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-notifications-controller
```

---

## Configuration Components

### 1. Services (Where to send)

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack service
  service.slack: |
    token: $slack-token

  # Email service
  service.email: |
    host: smtp.gmail.com
    port: 587
    from: $email-username
    username: $email-username
    password: $email-password

  # Webhook service
  service.webhook.deployment-status: |
    url: https://api.example.com/deployments
    headers:
      - name: Authorization
        value: Bearer $webhook-token
```

### 2. Templates (What to send)

```yaml
data:
  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} has been deployed!
      Sync Status: {{.app.status.sync.status}}
      Health Status: {{.app.status.health.status}}
      Revision: {{.app.status.sync.revision}}

  template.app-sync-failed: |
    message: |
      :x: Application {{.app.metadata.name}} sync failed!
      Error: {{.app.status.operationState.message}}

  template.app-health-degraded: |
    message: |
      :warning: Application {{.app.metadata.name}} health is degraded!
      Health Status: {{.app.status.health.status}}
```

### 3. Triggers (When to send)

```yaml
data:
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [app-deployed]

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]

  trigger.on-sync-running: |
    - when: app.status.operationState.phase in ['Running']
      send: [app-sync-running]

  trigger.on-sync-status-unknown: |
    - when: app.status.sync.status == 'Unknown'
      send: [app-sync-status-unknown]
```

---

## Slack Integration

### Create Slack App and Get Token

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SLACK SETUP                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Go to https://api.slack.com/apps                               │
│  2. Create New App → From Scratch                                  │
│  3. Name: "ArgoCD Notifications"                                   │
│  4. OAuth & Permissions → Scopes → Add:                            │
│     • chat:write                                                   │
│     • chat:write.public (for public channels)                      │
│  5. Install App to Workspace                                       │
│  6. Copy Bot User OAuth Token (xoxb-...)                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Configure Slack Service

```yaml
# argocd-notifications-secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  slack-token: xoxb-your-slack-bot-token
```

```yaml
# argocd-notifications-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
    signingSecret: $slack-signing-secret  # Optional for slash commands

  # Slack-specific templates
  template.app-deployed: |
    slack:
      attachments: |
        [{
          "color": "#18be52",
          "title": "{{ .app.metadata.name }} deployed successfully",
          "fields": [
            {"title": "Sync Status", "value": "{{.app.status.sync.status}}", "short": true},
            {"title": "Health", "value": "{{.app.status.health.status}}", "short": true},
            {"title": "Revision", "value": "{{.app.status.sync.revision | trunc 7}}", "short": true},
            {"title": "Environment", "value": "{{index .app.metadata.labels \"environment\"}}", "short": true}
          ]
        }]

  template.app-sync-failed: |
    slack:
      attachments: |
        [{
          "color": "#E96D76",
          "title": ":x: {{ .app.metadata.name }} sync failed",
          "text": "{{.app.status.operationState.message}}",
          "fields": [
            {"title": "Application", "value": "{{.app.metadata.name}}", "short": true},
            {"title": "Revision", "value": "{{.app.status.sync.revision | trunc 7}}", "short": true}
          ]
        }]

  # Triggers
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [app-deployed]

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
```

### Subscribe Applications to Slack

```yaml
# Add annotation to Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: deployments-channel
    notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts-channel
```

---

## Microsoft Teams Integration

### Configure Teams Webhook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.teams: |
    recipientUrls:
      deployments: $teams-deployments-webhook
      alerts: $teams-alerts-webhook

  template.app-deployed: |
    teams:
      title: "Application Deployed"
      text: "{{.app.metadata.name}} has been deployed successfully"
      facts: |
        [{
          "name": "Sync Status",
          "value": "{{.app.status.sync.status}}"
        },{
          "name": "Health Status",
          "value": "{{.app.status.health.status}}"
        },{
          "name": "Revision",
          "value": "{{.app.status.sync.revision | trunc 7}}"
        }]
      potentialAction: |
        [{
          "@type": "OpenUri",
          "name": "Open in ArgoCD",
          "targets": [{"os": "default", "uri": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"}]
        }]

  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]
```

### Subscribe to Teams

```yaml
annotations:
  notifications.argoproj.io/subscribe.on-deployed.teams: deployments
  notifications.argoproj.io/subscribe.on-sync-failed.teams: alerts
```

---

## Email Notifications

### Configure Email Service

```yaml
# Secret for credentials
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  email-username: argocd@example.com
  email-password: app-password-here
```

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.email: |
    host: smtp.gmail.com
    port: 587
    from: $email-username
    username: $email-username
    password: $email-password

  template.app-deployed: |
    email:
      subject: "ArgoCD: {{.app.metadata.name}} deployed"
      body: |
        <h2>Application Deployed</h2>
        <p><strong>Application:</strong> {{.app.metadata.name}}</p>
        <p><strong>Sync Status:</strong> {{.app.status.sync.status}}</p>
        <p><strong>Health Status:</strong> {{.app.status.health.status}}</p>
        <p><strong>Revision:</strong> {{.app.status.sync.revision}}</p>
        <p><a href="{{.context.argocdUrl}}/applications/{{.app.metadata.name}}">View in ArgoCD</a></p>

  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]
```

### Subscribe to Email

```yaml
annotations:
  notifications.argoproj.io/subscribe.on-deployed.email: team@example.com
  notifications.argoproj.io/subscribe.on-sync-failed.email: alerts@example.com
```

---

## Webhook Integration

### Generic Webhook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.webhook.deployment-tracker: |
    url: https://api.example.com/deployments
    headers:
      - name: Content-Type
        value: application/json
      - name: Authorization
        value: Bearer $webhook-token

  template.app-deployed: |
    webhook:
      deployment-tracker:
        method: POST
        body: |
          {
            "application": "{{.app.metadata.name}}",
            "status": "{{.app.status.sync.status}}",
            "health": "{{.app.status.health.status}}",
            "revision": "{{.app.status.sync.revision}}",
            "timestamp": "{{.app.status.operationState.finishedAt}}"
          }
```

### GitHub Status

```yaml
data:
  service.github: |
    appID: 12345
    installationID: 67890
    privateKey: $github-private-key

  template.app-deployed: |
    github:
      repoURLPath: "{{.app.spec.source.repoURL}}"
      revisionPath: "{{.app.status.sync.revision}}"
      status:
        state: success
        label: "ArgoCD"
        targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
        description: "Application synced successfully"
```

---

## Built-in Triggers

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BUILT-IN TRIGGERS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Trigger Name              │ Condition                              │
│  ───────────────────────────────────────────────────────────────   │
│  on-created                │ Application created                   │
│  on-deleted                │ Application deleted                   │
│  on-deployed               │ Sync succeeded & healthy              │
│  on-health-degraded        │ Health became Degraded                │
│  on-sync-failed            │ Sync operation failed                 │
│  on-sync-running           │ Sync in progress                      │
│  on-sync-status-unknown    │ Sync status unknown                   │
│  on-sync-succeeded         │ Sync completed (any health)           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Custom Triggers

### Time-Based Conditions

```yaml
trigger.on-sync-stalled: |
  - when: time.Now().Sub(time.Parse(app.status.operationState.startedAt)).Minutes() > 10
    send: [sync-stalled-warning]
```

### Label-Based Triggers

```yaml
trigger.on-production-deployed: |
  - when: app.status.operationState.phase == 'Succeeded' and app.metadata.labels.environment == 'production'
    send: [production-deployed]
```

### Health-Based Triggers

```yaml
trigger.on-any-unhealthy: |
  - when: app.status.health.status != 'Healthy'
    send: [app-unhealthy]
    oncePer: app.status.health.status
```

---

## Template Functions

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TEMPLATE FUNCTIONS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  String Functions:                                                  │
│  • trunc 7          - Truncate string                              │
│  • trim             - Remove whitespace                            │
│  • upper / lower    - Change case                                  │
│  • replace          - Replace string                               │
│                                                                     │
│  Time Functions:                                                    │
│  • time.Now()       - Current time                                 │
│  • time.Parse()     - Parse time string                            │
│                                                                     │
│  Available Context:                                                 │
│  • .app             - Application resource                         │
│  • .context.argocdUrl - ArgoCD URL                                 │
│  • .serviceType     - Notification service type                    │
│                                                                     │
│  Application Fields:                                                │
│  • .app.metadata.name                                              │
│  • .app.metadata.labels                                            │
│  • .app.metadata.annotations                                       │
│  • .app.spec.source.repoURL                                        │
│  • .app.spec.destination.namespace                                 │
│  • .app.status.sync.status                                         │
│  • .app.status.health.status                                       │
│  • .app.status.sync.revision                                       │
│  • .app.status.operationState.phase                                │
│  • .app.status.operationState.message                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Complete Configuration Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Context
  context: |
    argocdUrl: https://argocd.example.com

  # Services
  service.slack: |
    token: $slack-token

  service.email: |
    host: smtp.gmail.com
    port: 587
    from: $email-username
    username: $email-username
    password: $email-password

  # Templates
  template.app-deployed: |
    message: ":white_check_mark: *{{.app.metadata.name}}* has been deployed!"
    slack:
      attachments: |
        [{
          "color": "#18be52",
          "fields": [
            {"title": "Application", "value": "{{.app.metadata.name}}", "short": true},
            {"title": "Sync", "value": "{{.app.status.sync.status}}", "short": true},
            {"title": "Health", "value": "{{.app.status.health.status}}", "short": true},
            {"title": "Revision", "value": "{{.app.status.sync.revision | trunc 7}}", "short": true}
          ]
        }]
    email:
      subject: "[ArgoCD] {{.app.metadata.name}} deployed"
      body: |
        Application {{.app.metadata.name}} has been deployed successfully.
        Revision: {{.app.status.sync.revision}}

  template.app-sync-failed: |
    message: ":x: *{{.app.metadata.name}}* sync failed!"
    slack:
      attachments: |
        [{
          "color": "#E96D76",
          "title": "Sync Failed",
          "text": "{{.app.status.operationState.message}}"
        }]
    email:
      subject: "[ArgoCD] ALERT: {{.app.metadata.name}} sync failed"
      body: |
        Application {{.app.metadata.name}} sync failed!
        Error: {{.app.status.operationState.message}}

  # Triggers
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [app-deployed]

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-deployed]

  # Default subscriptions (applied to all apps)
  subscriptions: |
    - recipients:
        - slack:general-deployments
      triggers:
        - on-deployed
    - recipients:
        - slack:alerts
        - email:oncall@example.com
      triggers:
        - on-sync-failed
        - on-health-degraded
```

---

## Troubleshooting

```bash
# Check notifications controller logs
kubectl logs -n argocd deployment/argocd-notifications-controller

# Test notification
argocd admin notifications template notify \
  app-deployed my-app \
  --recipient slack:my-channel

# List configured services
argocd admin notifications service list

# List templates
argocd admin notifications template list
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Notification Components:                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Services   │ Where: Slack, Teams, Email, Webhook            │   │
│  │ Templates  │ What: Message format and content               │   │
│  │ Triggers   │ When: Event conditions                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Supported Services:                                                │
│  • Slack, Microsoft Teams, Discord                                 │
│  • Email (SMTP)                                                    │
│  • Generic Webhooks                                                │
│  • GitHub, GitLab status                                           │
│  • PagerDuty, OpsGenie                                             │
│  • Grafana, Telegram, and more                                     │
│                                                                     │
│  Key Triggers:                                                      │
│  • on-deployed: Successful sync + healthy                          │
│  • on-sync-failed: Sync error                                      │
│  • on-health-degraded: Health issues                               │
│                                                                     │
│  Next: Learn ArgoCD best practices                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
