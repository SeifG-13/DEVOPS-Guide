# Helm Cheat Sheet for DevOps Engineers

## Quick Reference Commands

### Repository Management
```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# Update repositories
helm repo update

# List repositories
helm repo list

# Search charts
helm search repo nginx
helm search hub nginx          # Search Artifact Hub
helm search repo bitnami/nginx --versions
```

### Chart Installation
```bash
# Install chart
helm install myrelease bitnami/nginx

# Install with namespace
helm install myrelease bitnami/nginx -n mynamespace --create-namespace

# Install with custom values
helm install myrelease bitnami/nginx -f values.yaml
helm install myrelease bitnami/nginx --set replicaCount=3

# Install specific version
helm install myrelease bitnami/nginx --version 15.0.0

# Dry run (preview)
helm install myrelease bitnami/nginx --dry-run

# Generate manifests only
helm template myrelease bitnami/nginx > manifests.yaml
```

### Release Management
```bash
# List releases
helm list
helm list -A                   # All namespaces
helm list --pending            # Pending releases

# Get release info
helm status myrelease
helm get values myrelease
helm get manifest myrelease
helm get all myrelease

# Upgrade release
helm upgrade myrelease bitnami/nginx
helm upgrade myrelease bitnami/nginx -f values.yaml
helm upgrade myrelease bitnami/nginx --set replicaCount=5
helm upgrade --install myrelease bitnami/nginx  # Install if not exists

# Rollback
helm rollback myrelease 1      # Rollback to revision 1
helm history myrelease         # View history

# Uninstall
helm uninstall myrelease
helm uninstall myrelease --keep-history
```

### Chart Development
```bash
# Create new chart
helm create mychart

# Lint chart
helm lint mychart

# Package chart
helm package mychart

# Install local chart
helm install myrelease ./mychart

# Show chart info
helm show chart bitnami/nginx
helm show values bitnami/nginx
helm show readme bitnami/nginx
```

---

## Chart Structure

```
mychart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── charts/                 # Dependencies
├── templates/              # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl        # Template helpers
│   ├── NOTES.txt           # Post-install notes
│   └── tests/
│       └── test-connection.yaml
├── .helmignore             # Files to ignore
└── README.md
```

### Chart.yaml
```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application
version: 1.0.0              # Chart version
appVersion: "1.0.0"         # Application version

dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

maintainers:
  - name: DevOps Team
    email: devops@example.com
```

### values.yaml
```yaml
replicaCount: 2

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

env:
  - name: ENVIRONMENT
    value: production

secrets:
  enabled: true
  data:
    API_KEY: ""  # Override in installation
```

---

## Template Syntax

### Basic Templates
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          {{- if .Values.env }}
          env:
            {{- toYaml .Values.env | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Helpers (_helpers.tpl)
```yaml
{{/*
Create chart name and version
*/}}
{{- define "mychart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create a default fully qualified app name
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
```

### Conditionals and Loops
```yaml
# Conditional
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# Loop
{{- range .Values.ingress.hosts }}
- host: {{ .host | quote }}
  http:
    paths:
    {{- range .paths }}
    - path: {{ .path }}
      pathType: {{ .pathType }}
    {{- end }}
{{- end }}

# With (change scope)
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

---

## Interview Q&A

### Q1: What is Helm?
**A:** Helm is the package manager for Kubernetes:
- Packages K8s resources into charts
- Manages releases (install, upgrade, rollback)
- Template engine for customization
- Dependency management

### Q2: What is the difference between helm install and helm upgrade?
**A:**
- **install**: Creates new release
- **upgrade**: Updates existing release
- **upgrade --install**: Install if not exists, otherwise upgrade (idempotent)

### Q3: What are Helm hooks?
**A:** Jobs that run at specific points in release lifecycle:
```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
```
Types: pre-install, post-install, pre-upgrade, post-upgrade, pre-delete, post-delete

### Q4: How do you pass values to a Helm chart?
**A:**
```bash
# Values file
helm install myrelease mychart -f values.yaml
helm install myrelease mychart -f values.yaml -f values-prod.yaml

# Command line
helm install myrelease mychart --set replicaCount=3
helm install myrelease mychart --set image.tag=v2.0.0

# Combined
helm install myrelease mychart -f values.yaml --set replicaCount=3
```
Priority: --set > -f (last) > -f (first) > defaults

### Q5: What is a Helm release?
**A:** An instance of a chart running in a cluster. You can:
- Have multiple releases of same chart
- Track history and rollback
- Upgrade independently

### Q6: How do you manage dependencies?
**A:**
```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```
```bash
helm dependency update mychart
helm dependency build mychart
```

### Q7: What is helm template?
**A:** Renders chart templates locally without installing:
```bash
helm template myrelease mychart > manifests.yaml
```
Useful for:
- Debugging templates
- GitOps workflows
- CI/CD validation

### Q8: How do you rollback a Helm release?
**A:**
```bash
# View history
helm history myrelease

# Rollback to previous
helm rollback myrelease

# Rollback to specific revision
helm rollback myrelease 2
```

### Q9: What is the difference between chart version and appVersion?
**A:**
- **version**: Chart version (incremented when chart changes)
- **appVersion**: Application version in the chart (informational)

### Q10: How do you test Helm charts?
**A:**
```bash
# Lint
helm lint mychart

# Dry run
helm install myrelease mychart --dry-run

# Test hooks
helm test myrelease

# Unit tests (helm-unittest plugin)
helm unittest mychart
```

---

## Helm in CI/CD

### GitHub Actions
```yaml
name: Helm Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v3

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}

      - name: Helm lint
        run: helm lint ./charts/myapp

      - name: Helm upgrade
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            -f ./charts/myapp/values-prod.yaml
```

---

## Helmfile (Multi-Chart Management)

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

releases:
  - name: nginx
    namespace: web
    chart: bitnami/nginx
    version: 15.0.0
    values:
      - values/nginx.yaml

  - name: redis
    namespace: cache
    chart: bitnami/redis
    version: 18.0.0
    values:
      - values/redis.yaml

environments:
  production:
    values:
      - values/production.yaml
  staging:
    values:
      - values/staging.yaml
```

```bash
helmfile sync
helmfile -e production sync
helmfile diff
```

---

## Best Practices

1. **Version your charts** - Semantic versioning
2. **Use values.yaml defaults** - Sensible defaults
3. **Lint before installing** - `helm lint`
4. **Use --dry-run** - Preview before applying
5. **Keep charts simple** - One app per chart
6. **Document values** - Comments in values.yaml
7. **Use helpers** - Avoid duplication
8. **Test your charts** - helm test, unit tests
