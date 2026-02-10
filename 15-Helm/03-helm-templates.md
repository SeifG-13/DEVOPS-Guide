# Helm Templates

## Template Basics

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM TEMPLATE SYNTAX                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Template Actions:                                                  │
│  {{ .Values.key }}        - Access values                          │
│  {{ .Release.Name }}      - Built-in objects                       │
│  {{ include "name" . }}   - Include template                       │
│  {{ toYaml . }}           - Convert to YAML                        │
│                                                                     │
│  Control Structures:                                                │
│  {{- if condition }}      - Conditional                            │
│  {{- range .list }}       - Loop                                   │
│  {{- with .object }}      - Scope change                           │
│                                                                     │
│  Whitespace Control:                                                │
│  {{-  (left trim)         - Remove left whitespace                 │
│  -}}  (right trim)        - Remove right whitespace                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Built-in Objects

```yaml
# Available objects in templates

# Release object
{{ .Release.Name }}        # Release name
{{ .Release.Namespace }}   # Release namespace
{{ .Release.IsUpgrade }}   # Is this an upgrade?
{{ .Release.IsInstall }}   # Is this an install?
{{ .Release.Revision }}    # Revision number
{{ .Release.Service }}     # Rendering service (Helm)

# Chart object
{{ .Chart.Name }}          # Chart name
{{ .Chart.Version }}       # Chart version
{{ .Chart.AppVersion }}    # Application version
{{ .Chart.Description }}   # Chart description

# Values object
{{ .Values.key }}          # Access values
{{ .Values.nested.key }}   # Nested values

# Capabilities object
{{ .Capabilities.APIVersions }}           # Available API versions
{{ .Capabilities.KubeVersion.Version }}   # Kubernetes version
{{ .Capabilities.KubeVersion.Major }}     # Major version
{{ .Capabilities.KubeVersion.Minor }}     # Minor version

# Template object
{{ .Template.Name }}       # Current template file
{{ .Template.BasePath }}   # Templates directory

# Files object
{{ .Files.Get "config.json" }}    # Get file content
{{ .Files.GetBytes "image.png" }} # Get as bytes
{{ .Files.Glob "*.yaml" }}        # Glob files
```

---

## Control Structures

### Conditionals

```yaml
# Simple if
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# If-else
{{- if .Values.persistence.enabled }}
  volumeClaimTemplates:
    # ...
{{- else }}
  volumes:
    - name: data
      emptyDir: {}
{{- end }}

# If-else if-else
{{- if eq .Values.service.type "ClusterIP" }}
  # ClusterIP specific config
{{- else if eq .Values.service.type "NodePort" }}
  # NodePort specific config
{{- else if eq .Values.service.type "LoadBalancer" }}
  # LoadBalancer specific config
{{- else }}
  # Default config
{{- end }}

# Boolean operators
{{- if and .Values.ingress.enabled .Values.ingress.tls }}
  tls:
    # ...
{{- end }}

{{- if or .Values.postgresql.enabled .Values.externalDatabase.host }}
  # Database configuration
{{- end }}

{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

### Loops (range)

```yaml
# Loop over list
env:
{{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}

# Loop over map
labels:
{{- range $key, $value := .Values.labels }}
  {{ $key }}: {{ $value | quote }}
{{- end }}

# Loop with index
{{- range $index, $host := .Values.ingress.hosts }}
  - host: {{ $host.host }}
    # ...
{{- end }}

# Loop over files
{{- range $path, $_ := .Files.Glob "config/*.yaml" }}
  {{ $path | base }}: |
{{ $.Files.Get $path | indent 4 }}
{{- end }}
```

### With (scope change)

```yaml
# Change scope
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}

# With and else
{{- with .Values.tolerations }}
tolerations:
  {{- toYaml . | nindent 2 }}
{{- else }}
tolerations: []
{{- end }}

# Accessing parent scope with $
{{- with .Values.ingress }}
  {{- range .hosts }}
    host: {{ .host }}
    service: {{ $.Values.service.name }}  # Access root context
  {{- end }}
{{- end }}
```

---

## Template Functions

### String Functions

```yaml
# Case functions
{{ .Values.name | upper }}         # UPPERCASE
{{ .Values.name | lower }}         # lowercase
{{ .Values.name | title }}         # Title Case
{{ .Values.name | untitle }}       # unTitle Case

# Trimming
{{ .Values.name | trim }}          # Trim whitespace
{{ .Values.name | trimPrefix "-" }}
{{ .Values.name | trimSuffix "-" }}
{{ .Values.name | trimAll "-" }}

# String manipulation
{{ .Values.name | trunc 63 }}      # Truncate
{{ .Values.name | replace "-" "_" }}
{{ .Values.name | quote }}         # Add quotes
{{ .Values.name | squote }}        # Single quotes
{{ .Values.name | nospace }}       # Remove spaces

# Substring
{{ substr 0 5 .Values.name }}

# String checks
{{ hasPrefix "v" .Values.version }}
{{ hasSuffix ".yaml" .Values.file }}
{{ contains "test" .Values.env }}

# Default value
{{ .Values.name | default "default-name" }}
{{ .Values.tag | default .Chart.AppVersion }}

# Coalesce (first non-empty)
{{ coalesce .Values.name .Values.defaultName "fallback" }}

# printf
{{ printf "%s-%s" .Release.Name .Chart.Name }}
```

### Type Conversion

```yaml
# Type conversions
{{ .Values.port | int }}           # To integer
{{ .Values.count | int64 }}
{{ .Values.ratio | float64 }}
{{ .Values.enabled | toString }}   # To string
{{ .Values.list | toJson }}        # To JSON
{{ .Values.map | toYaml }}         # To YAML
{{ .Values.data | toPrettyJson }}  # Pretty JSON

# Parsing
{{ fromJson .Values.jsonString }}
{{ fromYaml .Values.yamlString }}
```

### List Functions

```yaml
# List operations
{{ list "a" "b" "c" }}             # Create list
{{ first .Values.list }}           # First element
{{ last .Values.list }}            # Last element
{{ rest .Values.list }}            # All but first
{{ initial .Values.list }}         # All but last
{{ append .Values.list "new" }}    # Append
{{ prepend .Values.list "new" }}   # Prepend
{{ concat .Values.list1 .Values.list2 }}  # Concatenate
{{ reverse .Values.list }}         # Reverse
{{ uniq .Values.list }}            # Unique values
{{ without .Values.list "remove" }} # Remove item

# List checks
{{ has "item" .Values.list }}      # Contains check
{{ len .Values.list }}             # Length

# Sort
{{ .Values.list | sortAlpha }}     # Sort strings
```

### Dict Functions

```yaml
# Dictionary operations
{{ dict "key" "value" }}           # Create dict
{{ get .Values.map "key" }}        # Get value
{{ set .Values.map "key" "value" }} # Set value
{{ unset .Values.map "key" }}      # Remove key
{{ hasKey .Values.map "key" }}     # Check key exists
{{ keys .Values.map }}             # Get keys
{{ values .Values.map }}           # Get values
{{ merge .Values.map1 .Values.map2 }} # Merge dicts
{{ pick .Values.map "key1" "key2" }}  # Pick keys
{{ omit .Values.map "key1" }}      # Omit keys
{{ pluck "key" .Values.list }}     # Extract from list of maps
```

### Math Functions

```yaml
# Math operations
{{ add 1 2 }}                      # Addition
{{ sub 5 2 }}                      # Subtraction
{{ mul 2 3 }}                      # Multiplication
{{ div 10 2 }}                     # Division
{{ mod 10 3 }}                     # Modulo
{{ max 1 2 3 }}                    # Maximum
{{ min 1 2 3 }}                    # Minimum
{{ ceil 1.5 }}                     # Ceiling
{{ floor 1.5 }}                    # Floor
{{ round 1.5 }}                    # Round
```

### Cryptographic Functions

```yaml
# Crypto functions
{{ sha256sum .Values.data }}       # SHA256 hash
{{ sha1sum .Values.data }}         # SHA1 hash
{{ b64enc .Values.data }}          # Base64 encode
{{ b64dec .Values.encoded }}       # Base64 decode
{{ randAlphaNum 10 }}              # Random string
{{ randAlpha 10 }}                 # Random letters
{{ randNumeric 10 }}               # Random numbers
{{ randAscii 10 }}                 # Random ASCII
{{ uuidv4 }}                       # Generate UUID

# Generate secrets
{{ randAlphaNum 32 | b64enc }}
```

---

## Whitespace Control

```yaml
# Without whitespace control (produces extra blank lines)
metadata:
  labels:
{{ if .Values.extraLabels }}
{{ toYaml .Values.extraLabels | indent 4 }}
{{ end }}

# With whitespace control (clean output)
metadata:
  labels:
{{- if .Values.extraLabels }}
{{ toYaml .Values.extraLabels | indent 4 }}
{{- end }}

# Indentation functions
{{ .Values.data | nindent 4 }}     # Newline + indent
{{ .Values.data | indent 4 }}      # Indent only

# Example
spec:
  containers:
    - name: app
      env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
```

---

## Named Templates

```yaml
# templates/_helpers.tpl

{{/*
Create a fully qualified name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Create labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels (subset for pod selector)
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

# Using named templates
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
```

### Template with Arguments

```yaml
# Define template with arguments
{{- define "myapp.env" -}}
{{- range $key, $value := . }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
{{- end }}

# Use with arguments
env:
  {{- include "myapp.env" .Values.environment | nindent 2 }}
```

---

## Working with Files

```yaml
# Get file content
data:
  config.json: |
{{ .Files.Get "files/config.json" | indent 4 }}

# Glob multiple files
{{- range $path, $_ := .Files.Glob "configs/*.yaml" }}
  {{ base $path }}: |
{{ $.Files.Get $path | indent 4 }}
{{- end }}

# As ConfigMap data
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
{{ (.Files.Glob "configs/*").AsConfig | indent 2 }}

# As Secret data (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-secret
type: Opaque
data:
{{ (.Files.Glob "secrets/*").AsSecrets | indent 2 }}

# Lines function
data:
  hosts: |
{{- range .Files.Lines "files/hosts.txt" }}
    {{ . }}
{{- end }}
```

---

## Advanced Patterns

### Conditional Resources

```yaml
# Only create if enabled
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  # ...
{{- end }}
```

### Multiple Documents

```yaml
# templates/resources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  key: value
---
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-secret
type: Opaque
data:
  password: {{ .Values.password | b64enc }}
```

### Lookup Function

```yaml
# Look up existing resources
{{- $secret := lookup "v1" "Secret" .Release.Namespace "existing-secret" }}
{{- if $secret }}
  # Secret exists, use it
  password: {{ index $secret.data "password" }}
{{- else }}
  # Create new password
  password: {{ randAlphaNum 16 | b64enc }}
{{- end }}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Template Actions:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ {{ .Values.x }}   │ Access values                           │   │
│  │ {{ if }}...{{end}}│ Conditionals                            │   │
│  │ {{ range }}       │ Loops                                   │   │
│  │ {{ with }}        │ Scope change                            │   │
│  │ {{ include }}     │ Named templates                         │   │
│  │ {{ define }}      │ Define templates                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Common Functions:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ quote, default    │ String handling                         │   │
│  │ toYaml, indent    │ YAML formatting                         │   │
│  │ b64enc, sha256sum │ Encoding/hashing                        │   │
│  │ dict, list        │ Data structures                         │   │
│  │ lookup            │ Query existing resources                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Best Practices:                                                    │
│  • Use {{- -}} for whitespace control                              │
│  • Create helper templates in _helpers.tpl                         │
│  • Use nindent for proper indentation                              │
│  • Quote string values                                             │
│  • Provide sensible defaults                                       │
│                                                                     │
│  Next: Learn about Helm hooks                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
