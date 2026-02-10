# Helm Best Practices

## Chart Development Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CHART DEVELOPMENT                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Structure:                                                         │
│  ✓ Follow standard chart layout                                    │
│  ✓ Keep templates organized and focused                            │
│  ✓ Use _helpers.tpl for reusable functions                         │
│  ✓ Include comprehensive README.md                                 │
│  ✓ Provide NOTES.txt for post-install guidance                     │
│                                                                     │
│  Naming:                                                            │
│  ✓ Use lowercase chart names                                       │
│  ✓ Use hyphens, not underscores                                    │
│  ✓ Resource names: {{ include "chart.fullname" . }}                │
│  ✓ Truncate names to 63 characters                                 │
│                                                                     │
│  Versioning:                                                        │
│  ✓ Use SemVer for chart versions                                   │
│  ✓ Separate chart version from appVersion                          │
│  ✓ Document changes in CHANGELOG.md                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Values Best Practices

### Values Structure

```yaml
# values.yaml - Well-structured example

# Global settings
global:
  imageRegistry: ""
  imagePullSecrets: []

# Application image
image:
  repository: myapp
  tag: ""  # Defaults to appVersion
  pullPolicy: IfNotPresent

# Replicas (simple, not nested unnecessarily)
replicaCount: 1

# Service configuration
service:
  type: ClusterIP
  port: 80
  annotations: {}

# Resource management
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

# Feature flags (clear naming)
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10

# External dependencies
postgresql:
  enabled: true
  # Subchart values here
```

### Values Guidelines

```yaml
# ✓ DO: Use flat structures when possible
replicaCount: 3
serviceType: ClusterIP

# ✗ DON'T: Over-nest values
# deployment:
#   spec:
#     replicas: 3
#     service:
#       type: ClusterIP

# ✓ DO: Use clear boolean names
autoscaling:
  enabled: false
persistence:
  enabled: true

# ✗ DON'T: Use confusing names
# noAutoscaling: true
# enablePersistence: true

# ✓ DO: Provide comments
# Number of replicas for the deployment
replicaCount: 1

# ✓ DO: Use empty objects for optional nested values
podAnnotations: {}
nodeSelector: {}
tolerations: []
affinity: {}
```

---

## Template Best Practices

### Labels and Selectors

```yaml
# templates/_helpers.tpl

# Standard labels
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

# Selector labels (immutable, keep minimal)
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Defaults and Required Values

```yaml
# Use defaults wisely
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"

# Require critical values
{{ required "A valid image.repository is required" .Values.image.repository }}

# Coalesce for multiple fallbacks
storageClass: {{ coalesce .Values.storageClass .Values.global.storageClass "standard" }}
```

### Conditional Resources

```yaml
# Only create if enabled
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# Optional annotations
{{- with .Values.podAnnotations }}
annotations:
  {{- toYaml . | nindent 4 }}
{{- end }}
```

---

## Security Best Practices

```yaml
# templates/deployment.yaml

# Always set security context
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          # Always set resource limits
          resources:
            {{- toYaml .Values.resources | nindent 12 }}

          # Use non-root ports
          ports:
            - containerPort: 8080

          # Mount tmp as writable
          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}
```

---

## Dependency Management

```yaml
# Chart.yaml - Dependencies

dependencies:
  # Pin to major version
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

  # Always specify condition
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled

  # Use aliases for multiple instances
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: session-redis
    condition: sessionRedis.enabled
```

```bash
# Always update before packaging
helm dependency update ./my-chart

# Commit Chart.lock
git add Chart.lock
git commit -m "Update dependencies"
```

---

## Release Management

### Installation

```bash
# Always use --namespace
helm install my-release ./my-chart -n production

# Create namespace if needed
helm install my-release ./my-chart -n production --create-namespace

# Use meaningful release names
helm install myapp-v2 ./my-chart  # ✓ Good
helm install release1 ./my-chart  # ✗ Bad

# Preview with dry-run
helm install my-release ./my-chart --dry-run --debug
```

### Upgrades

```bash
# Use --atomic in production
helm upgrade my-release ./my-chart --atomic --timeout 10m

# Preview changes
helm diff upgrade my-release ./my-chart -f values.yaml

# Keep history reasonable
helm upgrade my-release ./my-chart --history-max 10
```

### Rollbacks

```bash
# Know your rollback targets
helm history my-release

# Test rollback in staging
helm rollback my-release 5 --dry-run

# Keep enough history
helm upgrade my-release ./my-chart --history-max 10
```

---

## CI/CD Best Practices

```yaml
# GitHub Actions example
name: Helm CI/CD

on:
  push:
    branches: [main]
    paths:
      - 'charts/**'

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      # Always lint
      - uses: azure/setup-helm@v3
      - run: helm lint charts/my-app --strict

      # Template test
      - run: helm template my-app charts/my-app > manifests.yaml

      # Validate manifests
      - run: kubeconform manifests.yaml

  deploy:
    needs: lint-test
    runs-on: ubuntu-latest
    steps:
      # Use helm diff before upgrade
      - run: helm diff upgrade my-app charts/my-app

      # Atomic deployment
      - run: |
          helm upgrade --install my-app charts/my-app \
            --namespace production \
            --atomic \
            --timeout 10m \
            --set image.tag=${{ github.sha }}
```

---

## Documentation Best Practices

### README.md

```markdown
# My Application Chart

## Introduction
Brief description of what this chart deploys.

## Prerequisites
- Kubernetes 1.23+
- Helm 3.x
- PV provisioner (if persistence enabled)

## Installing the Chart

```bash
helm install my-release ./my-app
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| replicaCount | Number of replicas | 1 |
| image.repository | Image repository | myapp |
| image.tag | Image tag | Chart appVersion |

## Upgrading

### From 1.x to 2.x
Breaking changes and migration steps.
```

### NOTES.txt

```yaml
# templates/NOTES.txt
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To get the application URL:
{{- if .Values.ingress.enabled }}
  http{{ if .Values.ingress.tls }}s{{ end }}://{{ .Values.ingress.host }}
{{- else if eq .Values.service.type "LoadBalancer" }}
  export SERVICE_IP=$(kubectl get svc {{ include "myapp.fullname" . }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else }}
  kubectl port-forward svc/{{ include "myapp.fullname" . }} {{ .Values.service.port }}:{{ .Values.service.port }}
  echo http://127.0.0.1:{{ .Values.service.port }}
{{- end }}
```

---

## Testing Best Practices

### Helm Test

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: curlimages/curl
      command: ['curl', '-f', '{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/health']
```

### Unit Testing

```yaml
# tests/deployment_test.yaml
suite: deployment tests
templates:
  - deployment.yaml
tests:
  - it: should set correct replicas
    set:
      replicaCount: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3

  - it: should require resources
    asserts:
      - isNotNull:
          path: spec.template.spec.containers[0].resources
```

---

## Checklist Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM BEST PRACTICES CHECKLIST                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Chart Development:                                                 │
│  □ Standard chart structure                                        │
│  □ Comprehensive README and NOTES.txt                              │
│  □ Values documented with comments                                 │
│  □ Sensible defaults provided                                      │
│  □ Schema validation (values.schema.json)                          │
│                                                                     │
│  Templates:                                                         │
│  □ Standard labels on all resources                                │
│  □ Resource limits defined                                         │
│  □ Security contexts set                                           │
│  □ Probes configured                                               │
│  □ Network policies included                                       │
│                                                                     │
│  Testing:                                                           │
│  □ Lint passes (helm lint --strict)                                │
│  □ Template renders correctly                                      │
│  □ Unit tests written                                              │
│  □ Integration tests pass                                          │
│                                                                     │
│  Operations:                                                        │
│  □ Version pinned in CI/CD                                         │
│  □ Atomic upgrades configured                                      │
│  □ Rollback tested                                                 │
│  □ Monitoring in place                                             │
│                                                                     │
│  Security:                                                          │
│  □ No hardcoded secrets                                            │
│  □ Charts signed (production)                                      │
│  □ Images scanned                                                  │
│  □ RBAC properly scoped                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Anti-Patterns to Avoid

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM ANTI-PATTERNS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✗ Hardcoding values in templates                                  │
│  ✗ Skipping security contexts                                      │
│  ✗ Not setting resource limits                                     │
│  ✗ Unpinned dependency versions                                    │
│  ✗ Missing health probes                                           │
│  ✗ Storing secrets in values.yaml                                  │
│  ✗ Overly complex templates                                        │
│  ✗ Inconsistent naming conventions                                 │
│  ✗ Missing documentation                                           │
│  ✗ Not testing before release                                      │
│  ✗ Ignoring deprecation warnings                                   │
│  ✗ Using latest tags                                               │
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
│  Core Principles:                                                   │
│  • Keep charts simple and focused                                  │
│  • Document everything                                             │
│  • Test before release                                             │
│  • Use atomic upgrades                                             │
│  • Prioritize security                                             │
│                                                                     │
│  Key Practices:                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Values       │ Flat, documented, sensible defaults          │   │
│  │ Templates    │ Standard labels, security contexts           │   │
│  │ Testing      │ Lint, unit tests, integration tests          │   │
│  │ CI/CD        │ Diff before upgrade, atomic deploys          │   │
│  │ Security     │ No secrets in values, signed charts          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Remember:                                                          │
│  • Charts are APIs - design them carefully                         │
│  • Breaking changes need migration paths                           │
│  • Production deployments need safety nets                         │
│  • Good documentation saves time                                   │
│                                                                     │
│  This completes the Helm section!                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
