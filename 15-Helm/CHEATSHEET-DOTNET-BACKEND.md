# Helm Cheat Sheet for .NET Backend Engineers

## Quick Reference - Helm for .NET Apps

### Create .NET Helm Chart
```bash
# Create chart structure
helm create myapi

# Customize for .NET
# - Update values.yaml with .NET defaults
# - Configure health checks
# - Add ConfigMap for appsettings
```

---

## Complete .NET Helm Chart

### Chart.yaml
```yaml
apiVersion: v2
name: myapi
description: A Helm chart for .NET Web API
type: application
version: 1.0.0
appVersion: "1.0.0"

maintainers:
  - name: Development Team
    email: dev@example.com
```

### values.yaml
```yaml
replicaCount: 2

image:
  repository: myregistry/myapi
  tag: "latest"
  pullPolicy: IfNotPresent

imagePullSecrets: []

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.example.com

# .NET specific configuration
dotnet:
  environment: Production
  urls: "http://+:8080"

# Application settings (non-sensitive)
appSettings:
  Logging__LogLevel__Default: "Information"
  AllowedHosts: "*"

# Secrets reference
secrets:
  existingSecret: ""  # Use existing secret
  connectionString: ""  # Or set directly (not recommended)
  jwtSecret: ""

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

healthCheck:
  enabled: true
  livenessPath: /health/live
  readinessPath: /health/ready
  initialDelaySeconds: 15
  periodSeconds: 10

nodeSelector: {}
tolerations: []
affinity: {}
```

### templates/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapi.fullname" . }}
  labels:
    {{- include "myapi.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapi.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "myapi.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: {{ .Values.dotnet.environment | quote }}
            - name: ASPNETCORE_URLS
              value: {{ .Values.dotnet.urls | quote }}
          envFrom:
            - configMapRef:
                name: {{ include "myapi.fullname" . }}-config
            {{- if or .Values.secrets.existingSecret .Values.secrets.connectionString }}
            - secretRef:
                name: {{ .Values.secrets.existingSecret | default (printf "%s-secrets" (include "myapi.fullname" .)) }}
            {{- end }}
          {{- if .Values.healthCheck.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.livenessPath }}
              port: http
            initialDelaySeconds: {{ .Values.healthCheck.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.readinessPath }}
              port: http
            initialDelaySeconds: 5
            periodSeconds: {{ .Values.healthCheck.periodSeconds }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### templates/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapi.fullname" . }}-config
  labels:
    {{- include "myapi.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.appSettings }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

### templates/secret.yaml
```yaml
{{- if and (not .Values.secrets.existingSecret) (or .Values.secrets.connectionString .Values.secrets.jwtSecret) }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapi.fullname" . }}-secrets
  labels:
    {{- include "myapi.labels" . | nindent 4 }}
type: Opaque
stringData:
  {{- if .Values.secrets.connectionString }}
  ConnectionStrings__DefaultConnection: {{ .Values.secrets.connectionString | quote }}
  {{- end }}
  {{- if .Values.secrets.jwtSecret }}
  JWT__Secret: {{ .Values.secrets.jwtSecret | quote }}
  {{- end }}
{{- end }}
```

### templates/hpa.yaml
```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapi.fullname" . }}
  labels:
    {{- include "myapi.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "myapi.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

---

## Installation Examples

### Basic Installation
```bash
helm install myapi ./charts/myapi \
  --namespace production \
  --create-namespace
```

### With Custom Values
```bash
helm install myapi ./charts/myapi \
  --namespace production \
  --set image.tag=v1.2.3 \
  --set replicaCount=5 \
  -f values-production.yaml
```

### Environment-Specific Values Files
```yaml
# values-production.yaml
replicaCount: 5

dotnet:
  environment: Production

appSettings:
  Logging__LogLevel__Default: "Warning"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20

resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi

ingress:
  hosts:
    - host: api.production.example.com
```

```yaml
# values-staging.yaml
replicaCount: 2

dotnet:
  environment: Staging

appSettings:
  Logging__LogLevel__Default: "Debug"

autoscaling:
  enabled: false

ingress:
  hosts:
    - host: api.staging.example.com
```

---

## Handling Secrets

### Using Existing Secrets
```bash
# Create secret separately
kubectl create secret generic myapi-secrets \
  --from-literal=ConnectionStrings__DefaultConnection="Server=..." \
  --from-literal=JWT__Secret="my-secret" \
  -n production

# Reference in values
helm install myapi ./charts/myapi \
  --set secrets.existingSecret=myapi-secrets
```

### Using External Secrets
```yaml
# templates/external-secret.yaml
{{- if .Values.externalSecrets.enabled }}
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "myapi.fullname" . }}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: {{ .Values.externalSecrets.secretStore }}
    kind: ClusterSecretStore
  target:
    name: {{ include "myapi.fullname" . }}-secrets
  data:
    - secretKey: ConnectionStrings__DefaultConnection
      remoteRef:
        key: {{ .Values.externalSecrets.connectionStringKey }}
    - secretKey: JWT__Secret
      remoteRef:
        key: {{ .Values.externalSecrets.jwtSecretKey }}
{{- end }}
```

---

## CI/CD Integration

### GitHub Actions
```yaml
name: Deploy .NET API with Helm

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Set up Helm
        uses: azure/setup-helm@v3

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapi ./charts/myapi \
            --namespace production \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            -f ./charts/myapi/values-production.yaml \
            --wait --timeout 5m
```

---

## Interview Q&A

### Q1: How do you deploy a .NET application using Helm?
**A:**
1. Create Helm chart with templates for Deployment, Service, ConfigMap
2. Configure values.yaml with .NET-specific settings (environment, URLs)
3. Use ConfigMap for appsettings, Secrets for sensitive data
4. Install with `helm install myapi ./charts/myapi -f values-prod.yaml`

### Q2: How do you manage appsettings in Helm?
**A:**
- **ConfigMap** for non-sensitive settings
- **Secrets** for connection strings, API keys
- Environment variables override appsettings.json
```yaml
envFrom:
  - configMapRef:
      name: myapi-config
  - secretRef:
      name: myapi-secrets
```

### Q3: How do you handle different environments?
**A:**
```bash
# Separate values files
helm install myapi ./myapi -f values-dev.yaml
helm install myapi ./myapi -f values-production.yaml

# Or use --set for overrides
helm install myapi ./myapi --set dotnet.environment=Production
```

### Q4: How do you update image tag during deployment?
**A:**
```bash
helm upgrade myapi ./myapi --set image.tag=v2.0.0

# Or in CI/CD
helm upgrade myapi ./myapi --set image.tag=${{ github.sha }}
```

### Q5: How do you handle health checks in Helm?
**A:**
```yaml
# values.yaml
healthCheck:
  livenessPath: /health/live
  readinessPath: /health/ready

# deployment.yaml
livenessProbe:
  httpGet:
    path: {{ .Values.healthCheck.livenessPath }}
    port: http
```

### Q6: How do you rollback a .NET deployment with Helm?
**A:**
```bash
# View history
helm history myapi

# Rollback
helm rollback myapi 2
```

---

## Database Migrations with Helm

### Migration Job Hook
```yaml
# templates/migration-job.yaml
{{- if .Values.migrations.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapi.fullname" . }}-migrations
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-1"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrations
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["dotnet", "ef", "database", "update"]
          envFrom:
            - secretRef:
                name: {{ include "myapi.fullname" . }}-secrets
{{- end }}
```

---

## Best Practices for .NET Helm Charts

1. **Use ConfigMaps for config** - appsettings via environment variables
2. **Externalize secrets** - Never in values.yaml
3. **Configure health checks** - /health/live and /health/ready
4. **Enable HPA** - Auto-scale based on load
5. **Use resource limits** - Prevent resource starvation
6. **Version your charts** - Match with app versions
7. **Use hooks for migrations** - pre-upgrade hooks
8. **Template common labels** - Consistent metadata
