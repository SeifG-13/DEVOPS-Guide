# Helm Dependencies

## Dependencies Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM DEPENDENCIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What are Chart Dependencies?                                       │
│  • Charts that your chart requires to function                     │
│  • Managed through Chart.yaml                                      │
│  • Downloaded to charts/ directory                                 │
│  • Can be conditionally enabled/disabled                           │
│                                                                     │
│  Example: Web Application Dependencies                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    My Web App                                │   │
│  │                        │                                     │   │
│  │         ┌──────────────┼──────────────┐                     │   │
│  │         ▼              ▼              ▼                     │   │
│  │    ┌────────┐    ┌────────┐    ┌────────┐                   │   │
│  │    │PostgreSQL│  │  Redis │    │ Nginx  │                   │   │
│  │    └────────┘    └────────┘    └────────┘                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Defining Dependencies

### Chart.yaml Dependencies

```yaml
# Chart.yaml
apiVersion: v2
name: my-webapp
version: 1.0.0
appVersion: "2.0.0"

dependencies:
  # Basic dependency
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami

  # Dependency with condition
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled

  # Dependency with alias
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: session-cache
    condition: sessionCache.enabled

  # Dependency with tags
  - name: elasticsearch
    version: "19.x.x"
    repository: https://charts.bitnami.com/bitnami
    tags:
      - logging
      - monitoring

  # Local dependency
  - name: common
    version: "1.x.x"
    repository: "file://../common"

  # OCI registry
  - name: nginx
    version: "15.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts
```

---

## Managing Dependencies

```bash
# Add repository for dependencies
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Download dependencies
helm dependency update ./my-webapp
# or
helm dep up ./my-webapp

# List dependencies
helm dependency list ./my-webapp

# Build dependencies (download + package)
helm dependency build ./my-webapp

# Output:
# NAME          VERSION   REPOSITORY                              STATUS
# postgresql    12.x.x    https://charts.bitnami.com/bitnami      ok
# redis         17.x.x    https://charts.bitnami.com/bitnami      ok
```

### Chart.lock

```yaml
# Chart.lock - Auto-generated, do not edit manually
dependencies:
  - name: postgresql
    repository: https://charts.bitnami.com/bitnami
    version: 12.1.5
  - name: redis
    repository: https://charts.bitnami.com/bitnami
    version: 17.3.2
digest: sha256:abc123...
generated: "2024-01-15T10:30:00Z"
```

---

## Conditional Dependencies

### Using Conditions

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled,cache.enabled
```

```yaml
# values.yaml
postgresql:
  enabled: true

redis:
  enabled: false

cache:
  enabled: true  # This would enable redis via second condition
```

### Using Tags

```yaml
# Chart.yaml
dependencies:
  - name: elasticsearch
    version: "19.x.x"
    repository: https://charts.bitnami.com/bitnami
    tags:
      - logging

  - name: kibana
    version: "10.x.x"
    repository: https://charts.bitnami.com/bitnami
    tags:
      - logging

  - name: prometheus
    version: "23.x.x"
    repository: https://charts.bitnami.com/bitnami
    tags:
      - monitoring

  - name: grafana
    version: "9.x.x"
    repository: https://charts.bitnami.com/bitnami
    tags:
      - monitoring
```

```yaml
# values.yaml
tags:
  logging: true      # Enables elasticsearch and kibana
  monitoring: false  # Disables prometheus and grafana
```

---

## Configuring Dependencies

### Passing Values to Dependencies

```yaml
# values.yaml

# Configure postgresql dependency
postgresql:
  enabled: true
  auth:
    username: myapp
    password: secretpassword
    database: myapp_db
  primary:
    persistence:
      enabled: true
      size: 10Gi
  resources:
    requests:
      memory: 256Mi
      cpu: 250m

# Configure redis dependency
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    password: redispassword
  master:
    persistence:
      enabled: false
```

### Global Values

```yaml
# values.yaml

# Global values available to all charts
global:
  imageRegistry: myregistry.io
  imagePullSecrets:
    - name: regcred
  storageClass: fast-ssd

# These are automatically available in dependencies
# Access in templates: .Values.global.imageRegistry
```

### Import Values

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    import-values:
      # Import child values to parent
      - child: primary.service
        parent: database

      # Import with data transformation
      - child: auth
        parent: db.credentials
```

```yaml
# Result in parent values
database:
  port: 5432
  type: ClusterIP

db:
  credentials:
    username: postgres
    password: ""
```

---

## Dependency Aliases

```yaml
# Chart.yaml - Multiple instances of same chart
dependencies:
  # Primary database
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: primary-db
    condition: primaryDb.enabled

  # Read replica database
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: replica-db
    condition: replicaDb.enabled

  # Application cache
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: app-cache
    condition: appCache.enabled

  # Session cache
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: session-cache
    condition: sessionCache.enabled
```

```yaml
# values.yaml
primary-db:
  enabled: true
  auth:
    database: primary

replica-db:
  enabled: true
  auth:
    database: replica

app-cache:
  enabled: true
  architecture: standalone

session-cache:
  enabled: true
  architecture: replication
```

---

## Local Dependencies

### File-based Dependencies

```
my-project/
├── charts/
│   └── common/
│       ├── Chart.yaml
│       └── templates/
│           └── _helpers.tpl
├── my-webapp/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       └── deployment.yaml
```

```yaml
# my-webapp/Chart.yaml
apiVersion: v2
name: my-webapp
version: 1.0.0

dependencies:
  - name: common
    version: "1.x.x"
    repository: "file://../charts/common"
```

### Library Charts

```yaml
# common/Chart.yaml
apiVersion: v2
name: common
version: 1.0.0
type: library  # This is a library chart
```

```yaml
# common/templates/_helpers.tpl
{{- define "common.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "common.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "common.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
```

```yaml
# my-webapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "common.fullname" . }}
  labels:
    {{- include "common.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "common.selectorLabels" . | nindent 6 }}
```

---

## Accessing Dependency Resources

### In Templates

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            # Reference postgresql subchart service
            - name: DATABASE_HOST
              value: {{ .Release.Name }}-postgresql

            # Or construct from values
            - name: DATABASE_HOST
              value: "{{ .Release.Name }}-{{ .Values.postgresql.nameOverride | default \"postgresql\" }}"

            - name: DATABASE_PORT
              value: "{{ .Values.postgresql.primary.service.ports.postgresql }}"

            - name: DATABASE_USER
              value: {{ .Values.postgresql.auth.username }}

            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-postgresql
                  key: password

            # Reference redis subchart
            - name: REDIS_HOST
              value: {{ .Release.Name }}-redis-master

            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-redis
                  key: redis-password
```

### Init Container for Dependencies

```yaml
# templates/deployment.yaml
spec:
  template:
    spec:
      initContainers:
        {{- if .Values.postgresql.enabled }}
        - name: wait-for-postgresql
          image: busybox
          command: ['sh', '-c']
          args:
            - |
              until nc -z {{ .Release.Name }}-postgresql {{ .Values.postgresql.primary.service.ports.postgresql }}; do
                echo "Waiting for PostgreSQL..."
                sleep 2
              done
        {{- end }}

        {{- if .Values.redis.enabled }}
        - name: wait-for-redis
          image: busybox
          command: ['sh', '-c']
          args:
            - |
              until nc -z {{ .Release.Name }}-redis-master 6379; do
                echo "Waiting for Redis..."
                sleep 2
              done
        {{- end }}
```

---

## Dependency Version Constraints

```yaml
# Chart.yaml - Version constraints
dependencies:
  # Exact version
  - name: postgresql
    version: "12.1.5"
    repository: https://charts.bitnami.com/bitnami

  # Patch version range (12.1.x)
  - name: redis
    version: "17.3.x"
    repository: https://charts.bitnami.com/bitnami

  # Minor version range (12.x.x)
  - name: mongodb
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami

  # Range operators
  - name: kafka
    version: ">=22.0.0 <23.0.0"
    repository: https://charts.bitnami.com/bitnami

  # Caret range (^1.2.3 = >=1.2.3 <2.0.0)
  - name: nginx
    version: "^15.0.0"
    repository: https://charts.bitnami.com/bitnami

  # Tilde range (~1.2.3 = >=1.2.3 <1.3.0)
  - name: apache
    version: "~10.0.0"
    repository: https://charts.bitnami.com/bitnami
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                DEPENDENCY BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Version Management:                                                │
│  ✓ Pin major versions in production                                │
│  ✓ Commit Chart.lock to version control                            │
│  ✓ Test dependency updates in staging first                        │
│  ✓ Document breaking changes                                       │
│                                                                     │
│  Configuration:                                                     │
│  ✓ Use conditions for optional dependencies                        │
│  ✓ Document required values for dependencies                       │
│  ✓ Use aliases for multiple instances                              │
│  ✓ Leverage global values wisely                                   │
│                                                                     │
│  Structure:                                                         │
│  ✓ Use library charts for shared code                              │
│  ✓ Keep dependency charts small                                    │
│  ✓ Prefer condition over tags                                      │
│  ✓ Test with dependencies enabled/disabled                         │
│                                                                     │
│  Security:                                                          │
│  ✓ Verify dependency sources                                       │
│  ✓ Review dependency values for secrets                            │
│  ✓ Keep dependencies updated                                       │
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
│  Dependency Commands:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm dep update   │ Download dependencies                   │   │
│  │ helm dep list     │ List dependencies                       │   │
│  │ helm dep build    │ Build dependency packages               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Concepts:                                                      │
│  • condition - Enable/disable based on values                      │
│  • tags - Group dependencies together                              │
│  • alias - Multiple instances of same chart                        │
│  • import-values - Share values between charts                     │
│  • global - Values available to all charts                         │
│                                                                     │
│  Files:                                                             │
│  • Chart.yaml - Define dependencies                                │
│  • Chart.lock - Lock versions (auto-generated)                     │
│  • charts/ - Downloaded dependency charts                          │
│                                                                     │
│  Next: Learn about Helm repositories                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
