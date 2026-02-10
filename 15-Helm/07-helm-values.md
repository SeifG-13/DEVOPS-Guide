# Helm Values and Configuration

## Values Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM VALUES                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What are Values?                                                   │
│  • Configuration parameters for charts                             │
│  • Customize chart behavior without modifying templates            │
│  • Can be set via files, command line, or environment              │
│                                                                     │
│  Value Sources (in order of precedence):                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. --set flags (highest)                                    │   │
│  │ 2. -f or --values files (later files override)              │   │
│  │ 3. Parent chart values                                      │   │
│  │ 4. Subchart values.yaml                                     │   │
│  │ 5. Chart's values.yaml (lowest)                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Setting Values

### Command Line

```bash
# Simple value
helm install my-release bitnami/nginx --set replicaCount=3

# Nested value
helm install my-release bitnami/nginx --set service.type=LoadBalancer

# Multiple values
helm install my-release bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer \
  --set resources.requests.memory=256Mi

# String value (force string)
helm install my-release bitnami/nginx --set image.tag="1.21"

# Array/List
helm install my-release bitnami/nginx \
  --set ingress.hosts[0].host=example.com \
  --set ingress.hosts[0].paths[0].path=/

# Complex values with --set-string
helm install my-release bitnami/nginx --set-string podAnnotations."prometheus\.io/scrape"=true

# Set from file content
helm install my-release bitnami/nginx --set-file config=./nginx.conf

# JSON values
helm install my-release bitnami/nginx --set-json 'resources={"limits":{"cpu":"200m"}}'
```

### Values Files

```bash
# Single values file
helm install my-release bitnami/nginx -f values.yaml

# Multiple values files (later overrides earlier)
helm install my-release bitnami/nginx \
  -f values.yaml \
  -f values-production.yaml

# Combined with --set
helm install my-release bitnami/nginx \
  -f values.yaml \
  --set replicaCount=5

# Remote values file
helm install my-release bitnami/nginx \
  -f https://example.com/values.yaml
```

---

## Values File Structure

### Basic Structure

```yaml
# values.yaml
# Global settings
global:
  imageRegistry: docker.io
  storageClass: standard

# Application settings
replicaCount: 1

image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

# Service configuration
service:
  type: ClusterIP
  port: 80

# Resource limits
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

### Environment-Specific Values

```yaml
# values-dev.yaml
replicaCount: 1

resources:
  limits:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: false

postgresql:
  enabled: true
  auth:
    database: app_dev
```

```yaml
# values-staging.yaml
replicaCount: 2

resources:
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  hosts:
    - host: staging.example.com
      paths:
        - path: /

postgresql:
  enabled: true
  primary:
    persistence:
      size: 10Gi
```

```yaml
# values-production.yaml
replicaCount: 5

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  hosts:
    - host: app.example.com
      paths:
        - path: /
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com

postgresql:
  enabled: true
  primary:
    persistence:
      size: 100Gi
  readReplicas:
    replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
```

---

## Accessing Values in Templates

### Basic Access

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

### With Defaults

```yaml
# Using default function
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"

# Nested defaults
port: {{ .Values.service.port | default 8080 }}

# Coalesce (first non-empty)
storageClass: {{ coalesce .Values.persistence.storageClass .Values.global.storageClass "standard" }}
```

### Conditional Values

```yaml
# Check if value exists
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# Check for non-empty
{{- if .Values.extraEnv }}
env:
  {{- toYaml .Values.extraEnv | nindent 2 }}
{{- end }}

# Required values
{{ required "image.repository is required" .Values.image.repository }}
```

---

## Global Values

```yaml
# Parent chart values.yaml
global:
  imageRegistry: myregistry.io
  imagePullSecrets:
    - name: regcred
  storageClass: fast-ssd
  env: production

# Subchart can access
postgresql:
  image:
    registry: "{{ .Values.global.imageRegistry }}"
```

```yaml
# In subchart templates
image: {{ .Values.global.imageRegistry }}/postgres:15
```

---

## Values Schema Validation

```json
// values.schema.json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "default": 1,
      "description": "Number of pod replicas"
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "description": "Image repository"
        },
        "tag": {
          "type": "string",
          "description": "Image tag"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"],
          "default": "IfNotPresent"
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"],
          "default": "ClusterIP"
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535,
          "default": 80
        }
      }
    },
    "resources": {
      "type": "object",
      "properties": {
        "limits": {
          "type": "object",
          "properties": {
            "cpu": { "type": "string" },
            "memory": { "type": "string" }
          }
        },
        "requests": {
          "type": "object",
          "properties": {
            "cpu": { "type": "string" },
            "memory": { "type": "string" }
          }
        }
      }
    }
  }
}
```

---

## Inspecting Values

```bash
# Show chart's default values
helm show values bitnami/nginx

# Save to file
helm show values bitnami/nginx > values.yaml

# Get values from release
helm get values my-release

# Get all values (including defaults)
helm get values my-release --all

# Get values as JSON
helm get values my-release -o json

# Debug template rendering
helm template my-release ./my-chart --debug
```

---

## Values Patterns

### Feature Toggles

```yaml
# values.yaml
features:
  metrics:
    enabled: true
    port: 9090
  tracing:
    enabled: false
    endpoint: ""
  cache:
    enabled: true
    type: redis
```

```yaml
# templates/deployment.yaml
{{- if .Values.features.metrics.enabled }}
- name: metrics
  containerPort: {{ .Values.features.metrics.port }}
{{- end }}
```

### Configuration Maps

```yaml
# values.yaml
config:
  app:
    logLevel: info
    maxConnections: 100
    timeout: 30s
  database:
    host: localhost
    port: 5432
```

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  config.yaml: |
    {{- toYaml .Values.config | nindent 4 }}
```

### Dynamic Environment Variables

```yaml
# values.yaml
env:
  - name: LOG_LEVEL
    value: info
  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: host

extraEnv: {}
```

```yaml
# templates/deployment.yaml
env:
  {{- range .Values.env }}
  - {{- toYaml . | nindent 4 }}
  {{- end }}
  {{- with .Values.extraEnv }}
  {{- toYaml . | nindent 2 }}
  {{- end }}
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VALUES BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Structure:                                                         │
│  ✓ Use flat structures when possible                               │
│  ✓ Group related values together                                   │
│  ✓ Document all values with comments                               │
│  ✓ Provide sensible defaults                                       │
│                                                                     │
│  Naming:                                                            │
│  ✓ Use camelCase for value names                                   │
│  ✓ Be consistent with naming conventions                           │
│  ✓ Use descriptive names                                           │
│  ✓ Avoid abbreviations                                             │
│                                                                     │
│  Security:                                                          │
│  ✓ Never commit secrets in values files                            │
│  ✓ Use external secrets management                                 │
│  ✓ Validate sensitive values                                       │
│                                                                     │
│  Maintenance:                                                       │
│  ✓ Use schema validation                                           │
│  ✓ Keep environment files separate                                 │
│  ✓ Version control all values files                                │
│  ✓ Review values on upgrade                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Setting Values:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ --set key=value    │ Command line                           │   │
│  │ -f values.yaml     │ Values file                            │   │
│  │ --set-file         │ Set from file content                  │   │
│  │ --set-json         │ JSON values                            │   │
│  │ --set-string       │ Force string type                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Precedence (highest to lowest):                                   │
│  1. --set flags                                                    │
│  2. Values files (-f)                                              │
│  3. Parent chart values                                            │
│  4. Chart's values.yaml                                            │
│                                                                     │
│  Key Patterns:                                                      │
│  • Environment-specific files (values-prod.yaml)                   │
│  • Global values for subcharts                                     │
│  • Schema validation                                               │
│  • Feature toggles                                                 │
│                                                                     │
│  Next: Learn about Helm upgrades and rollbacks                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
