# Chart Development

## Creating a New Chart

```bash
# Create a new chart
helm create my-app

# Chart structure created:
# my-app/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# ├── templates/
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── serviceaccount.yaml
# │   ├── ingress.yaml
# │   ├── hpa.yaml
# │   ├── _helpers.tpl
# │   ├── NOTES.txt
# │   └── tests/
# │       └── test-connection.yaml
# └── .helmignore
```

---

## Chart.yaml Configuration

```yaml
# Chart.yaml - Complete example
apiVersion: v2

# Chart name (required)
name: my-app

# Chart description (recommended)
description: A production-ready web application chart

# Chart type: application or library
type: application

# Chart version - follows SemVer
version: 1.2.3

# Application version
appVersion: "2.1.0"

# Kubernetes version constraint
kubeVersion: ">=1.23.0-0"

# Chart keywords for searching
keywords:
  - web
  - application
  - microservice

# Project home page
home: https://example.com/my-app

# Source code locations
sources:
  - https://github.com/example/my-app

# Chart maintainers
maintainers:
  - name: DevOps Team
    email: devops@example.com
    url: https://example.com/team

# Icon URL
icon: https://example.com/icon.png

# Deprecation flag
deprecated: false

# Annotations
annotations:
  artifacthub.io/license: Apache-2.0
  artifacthub.io/links: |
    - name: Documentation
      url: https://docs.example.com
  artifacthub.io/maintainers: |
    - name: DevOps Team
      email: devops@example.com

# Dependencies
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
    tags:
      - database
    import-values:
      - child: primary
        parent: postgresql

  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
    alias: cache
```

---

## Values Schema

```yaml
# values.yaml - Comprehensive example

# Global values (available to all subcharts)
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: ""

# Application image
image:
  registry: docker.io
  repository: mycompany/my-app
  tag: ""  # Defaults to appVersion
  pullPolicy: IfNotPresent
  pullSecrets: []

# Replica configuration
replicaCount: 1

# Pod labels and annotations
podLabels: {}
podAnnotations: {}

# Service account
serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

# Security contexts
podSecurityContext:
  fsGroup: 1000
  runAsUser: 1000
  runAsGroup: 1000
  runAsNonRoot: true

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

# Service configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  nodePort: ""
  annotations: {}

# Ingress configuration
ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

# Resource limits
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Probes
livenessProbe:
  enabled: true
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  enabled: true
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

# Pod disruption budget
podDisruptionBudget:
  enabled: false
  minAvailable: 1
  maxUnavailable: ""

# Node selection
nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []

# Environment variables
env: []
envFrom: []

# Extra volumes
extraVolumes: []
extraVolumeMounts: []

# ConfigMap
configMap:
  create: false
  data: {}

# Secrets
secrets:
  create: false
  data: {}

# Database dependency
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
```

---

## Values Schema Validation

```yaml
# values.schema.json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "description": "Number of replicas"
    },
    "image": {
      "type": "object",
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
          "description": "Image pull policy"
        }
      },
      "required": ["repository"]
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"],
          "description": "Service type"
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535,
          "description": "Service port"
        }
      }
    },
    "resources": {
      "type": "object",
      "properties": {
        "limits": {
          "type": "object"
        },
        "requests": {
          "type": "object"
        }
      }
    }
  },
  "required": ["replicaCount", "image"]
}
```

---

## Template Development

### Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "my-app.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.containerSecurityContext | nindent 12 }}
          image: "{{ .Values.image.registry }}/{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- if .Values.livenessProbe.enabled }}
          livenessProbe:
            {{- omit .Values.livenessProbe "enabled" | toYaml | nindent 12 }}
          {{- end }}
          {{- if .Values.readinessProbe.enabled }}
          readinessProbe:
            {{- omit .Values.readinessProbe "enabled" | toYaml | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            {{- with .Values.extraVolumeMounts }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
      volumes:
        - name: tmp
          emptyDir: {}
        {{- with .Values.extraVolumes }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Service Template

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  {{- if and (eq .Values.service.type "LoadBalancer") .Values.service.loadBalancerIP }}
  loadBalancerIP: {{ .Values.service.loadBalancerIP }}
  {{- end }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
```

### Ingress Template

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "my-app.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

---

## Helper Templates

```yaml
# templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
Return the proper image name
*/}}
{{- define "my-app.image" -}}
{{- $registry := .Values.global.imageRegistry | default .Values.image.registry -}}
{{- $repository := .Values.image.repository -}}
{{- $tag := .Values.image.tag | default .Chart.AppVersion -}}
{{- printf "%s/%s:%s" $registry $repository $tag -}}
{{- end }}
```

---

## Chart Testing

```bash
# Lint chart
helm lint ./my-app

# Lint with strict mode
helm lint ./my-app --strict

# Template locally (dry-run)
helm template my-release ./my-app

# Template with debug
helm template my-release ./my-app --debug

# Template with specific values
helm template my-release ./my-app -f values-prod.yaml

# Install with dry-run
helm install my-release ./my-app --dry-run

# Validate templates
helm template my-release ./my-app | kubectl apply --dry-run=client -f -
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                CHART DEVELOPMENT BEST PRACTICES                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Structure:                                                         │
│  ✓ Follow standard chart structure                                 │
│  ✓ Include comprehensive README                                    │
│  ✓ Document all values                                             │
│  ✓ Provide sensible defaults                                       │
│                                                                     │
│  Templates:                                                         │
│  ✓ Use helper templates for common patterns                        │
│  ✓ Follow Kubernetes naming conventions                            │
│  ✓ Include proper labels and annotations                           │
│  ✓ Support resource customization                                  │
│                                                                     │
│  Values:                                                            │
│  ✓ Use nested structure for complex configs                        │
│  ✓ Provide schema validation                                       │
│  ✓ Include commented examples                                      │
│  ✓ Keep values file organized                                      │
│                                                                     │
│  Testing:                                                           │
│  ✓ Lint charts before release                                      │
│  ✓ Test with different value combinations                          │
│  ✓ Include test templates                                          │
│  ✓ Use dry-run for validation                                      │
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
│  Chart Components:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Chart.yaml    │ Metadata and dependencies                   │   │
│  │ values.yaml   │ Default configuration                       │   │
│  │ templates/    │ Kubernetes manifest templates               │   │
│  │ _helpers.tpl  │ Reusable template functions                 │   │
│  │ NOTES.txt     │ Post-installation instructions              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Development Workflow:                                              │
│  1. Create chart with helm create                                  │
│  2. Configure Chart.yaml                                           │
│  3. Define values.yaml                                             │
│  4. Write templates                                                │
│  5. Create helper functions                                        │
│  6. Test and validate                                              │
│  7. Package and publish                                            │
│                                                                     │
│  Key Commands:                                                      │
│  • helm create - Create new chart                                  │
│  • helm lint - Validate chart                                      │
│  • helm template - Render templates locally                        │
│  • helm package - Package chart                                    │
│                                                                     │
│  Next: Learn about Helm templates and functions                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
