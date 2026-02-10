# Helm Library Charts

## Library Charts Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LIBRARY CHARTS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What are Library Charts?                                          │
│  • Charts that provide reusable templates and helpers              │
│  • Cannot be installed directly                                    │
│  • Used as dependencies by application charts                      │
│  • Enable code sharing across multiple charts                      │
│                                                                     │
│  Use Cases:                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Common Labels     │ Standardized labeling across charts     │   │
│  │ Helper Functions  │ Shared template logic                   │   │
│  │ Base Resources    │ Standard deployment patterns            │   │
│  │ Security Defaults │ Common security contexts                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Creating a Library Chart

### Chart Structure

```
common/
├── Chart.yaml
├── templates/
│   ├── _labels.tpl
│   ├── _names.tpl
│   ├── _resources.tpl
│   ├── _security.tpl
│   └── _validation.tpl
└── values.yaml
```

### Chart.yaml

```yaml
# common/Chart.yaml
apiVersion: v2
name: common
version: 1.0.0
type: library  # This makes it a library chart
description: Common templates for Helm charts
keywords:
  - common
  - library
  - helpers
maintainers:
  - name: Platform Team
    email: platform@example.com
```

### Common Labels Template

```yaml
# common/templates/_labels.tpl

{{/*
Common labels for all resources
*/}}
{{- define "common.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
app.kubernetes.io/name: {{ include "common.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | default "unknown" | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/part-of: {{ .Values.global.partOf | default .Chart.Name }}
{{- if .Values.commonLabels }}
{{ toYaml .Values.commonLabels }}
{{- end }}
{{- end }}

{{/*
Selector labels (subset for pod selector)
*/}}
{{- define "common.selectorLabels" -}}
app.kubernetes.io/name: {{ include "common.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Match labels for services
*/}}
{{- define "common.matchLabels" -}}
{{ include "common.selectorLabels" . }}
{{- end }}
```

### Common Names Template

```yaml
# common/templates/_names.tpl

{{/*
Expand the name of the chart.
*/}}
{{- define "common.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a fully qualified app name.
*/}}
{{- define "common.fullname" -}}
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
Create chart name and version for chart label
*/}}
{{- define "common.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Service account name
*/}}
{{- define "common.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "common.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Common Resources Template

```yaml
# common/templates/_resources.tpl

{{/*
Resource limits and requests
*/}}
{{- define "common.resources" -}}
{{- if .Values.resources }}
resources:
  {{- toYaml .Values.resources | nindent 2 }}
{{- end }}
{{- end }}

{{/*
Container ports
*/}}
{{- define "common.containerPorts" -}}
ports:
{{- range .Values.ports }}
  - name: {{ .name }}
    containerPort: {{ .containerPort }}
    protocol: {{ .protocol | default "TCP" }}
{{- end }}
{{- end }}

{{/*
Environment variables
*/}}
{{- define "common.env" -}}
{{- if or .Values.env .Values.envFrom }}
{{- if .Values.env }}
env:
  {{- toYaml .Values.env | nindent 2 }}
{{- end }}
{{- if .Values.envFrom }}
envFrom:
  {{- toYaml .Values.envFrom | nindent 2 }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Volume mounts
*/}}
{{- define "common.volumeMounts" -}}
{{- if .Values.volumeMounts }}
volumeMounts:
  {{- toYaml .Values.volumeMounts | nindent 2 }}
{{- end }}
{{- end }}

{{/*
Volumes
*/}}
{{- define "common.volumes" -}}
{{- if .Values.volumes }}
volumes:
  {{- toYaml .Values.volumes | nindent 2 }}
{{- end }}
{{- end }}
```

### Security Context Template

```yaml
# common/templates/_security.tpl

{{/*
Pod security context
*/}}
{{- define "common.podSecurityContext" -}}
{{- with .Values.podSecurityContext }}
securityContext:
  {{- toYaml . | nindent 2 }}
{{- else }}
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault
{{- end }}
{{- end }}

{{/*
Container security context
*/}}
{{- define "common.containerSecurityContext" -}}
{{- with .Values.containerSecurityContext }}
securityContext:
  {{- toYaml . | nindent 2 }}
{{- else }}
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
{{- end }}
{{- end }}

{{/*
Network policy default deny
*/}}
{{- define "common.networkPolicyDenyAll" -}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "common.fullname" . }}-deny-all
  labels:
    {{- include "common.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "common.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
{{- end }}
```

### Validation Template

```yaml
# common/templates/_validation.tpl

{{/*
Validate required values
*/}}
{{- define "common.validateRequired" -}}
{{- if not .Values.image.repository }}
{{- fail "image.repository is required" }}
{{- end }}
{{- end }}

{{/*
Validate resource limits
*/}}
{{- define "common.validateResources" -}}
{{- if not .Values.resources }}
{{- fail "resources.limits and resources.requests are required" }}
{{- end }}
{{- if not .Values.resources.limits }}
{{- fail "resources.limits is required" }}
{{- end }}
{{- if not .Values.resources.requests }}
{{- fail "resources.requests is required" }}
{{- end }}
{{- end }}
```

---

## Using Library Charts

### Adding as Dependency

```yaml
# my-app/Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
dependencies:
  - name: common
    version: "1.x.x"
    repository: https://charts.example.com
```

```bash
# Update dependencies
helm dependency update ./my-app
```

### Using Library Templates

```yaml
# my-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "common.fullname" . }}
  labels:
    {{- include "common.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "common.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "common.labels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "common.serviceAccountName" . }}
      {{- include "common.podSecurityContext" . | nindent 6 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          {{- include "common.containerSecurityContext" . | nindent 10 }}
          {{- include "common.containerPorts" . | nindent 10 }}
          {{- include "common.resources" . | nindent 10 }}
          {{- include "common.env" . | nindent 10 }}
          {{- include "common.volumeMounts" . | nindent 10 }}
      {{- include "common.volumes" . | nindent 6 }}
```

### Using Validation

```yaml
# my-app/templates/_helpers.tpl
{{- include "common.validateRequired" . -}}
{{- include "common.validateResources" . -}}
```

---

## Advanced Library Patterns

### Composable Deployment Template

```yaml
# common/templates/_deployment.tpl
{{- define "common.deployment" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "common.fullname" . }}
  labels:
    {{- include "common.labels" . | nindent 4 }}
  {{- with .Values.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "common.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "common.labels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "common.serviceAccountName" . }}
      {{- include "common.podSecurityContext" . | nindent 6 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          {{- include "common.containerSecurityContext" . | nindent 10 }}
          {{- include "common.containerPorts" . | nindent 10 }}
          {{- include "common.resources" . | nindent 10 }}
          {{- include "common.env" . | nindent 10 }}
          {{- if .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          {{- end }}
          {{- if .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- end }}
```

### Using Composable Template

```yaml
# my-app/templates/deployment.yaml
{{ include "common.deployment" . }}
```

### Extending with Additional Containers

```yaml
# my-app/templates/deployment.yaml
{{- $deployment := include "common.deployment" . | fromYaml }}
{{- $_ := set $deployment.spec.template.spec "initContainers" .Values.initContainers }}
{{- $_ := set $deployment.spec.template.spec.containers (append $deployment.spec.template.spec.containers .Values.sidecars) }}
{{ toYaml $deployment }}
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                 LIBRARY CHART BEST PRACTICES                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Design:                                                            │
│  ✓ Keep templates focused and single-purpose                       │
│  ✓ Use consistent naming conventions                               │
│  ✓ Document all templates                                          │
│  ✓ Provide sensible defaults                                       │
│                                                                     │
│  Versioning:                                                        │
│  ✓ Use semantic versioning                                         │
│  ✓ Document breaking changes                                       │
│  ✓ Maintain backward compatibility                                 │
│  ✓ Version constraints in dependencies                             │
│                                                                     │
│  Testing:                                                           │
│  ✓ Create test charts that use the library                         │
│  ✓ Test all template variations                                    │
│  ✓ Validate output with kubeval/kubeconform                        │
│                                                                     │
│  Organization:                                                      │
│  ✓ Group related templates                                         │
│  ✓ Separate concerns (labels, security, etc.)                      │
│  ✓ Keep values.yaml minimal                                        │
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
│  Library Chart Key Points:                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Type          │ type: library in Chart.yaml                 │   │
│  │ Templates     │ Only _*.tpl files with define blocks        │   │
│  │ Installation  │ Cannot be installed directly                │   │
│  │ Usage         │ Include as dependency                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Common Templates:                                                  │
│  • Labels and selectors                                            │
│  • Name generation                                                 │
│  • Security contexts                                               │
│  • Resource definitions                                            │
│  • Validation helpers                                              │
│                                                                     │
│  Benefits:                                                          │
│  • Code reuse across charts                                        │
│  • Consistent standards                                            │
│  • Easier maintenance                                              │
│  • Reduced duplication                                             │
│                                                                     │
│  Next: Learn about Helm best practices                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
