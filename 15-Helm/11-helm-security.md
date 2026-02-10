# Helm Security

## Security Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM SECURITY                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Security Concerns:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Chart Integrity  │ Verify charts are not tampered           │   │
│  │ Secrets          │ Protect sensitive data in values         │   │
│  │ RBAC             │ Control who can deploy what              │   │
│  │ Supply Chain     │ Trust chart sources                      │   │
│  │ Runtime          │ Secure generated resources               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Helm v3 Improvements:                                              │
│  • No Tiller (removed attack surface)                              │
│  • Uses Kubernetes RBAC directly                                   │
│  • Releases stored as secrets                                      │
│  • Improved chart provenance                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Chart Signing and Verification

### Creating a GPG Key

```bash
# Generate GPG key
gpg --full-generate-key

# List keys
gpg --list-keys

# Export public key
gpg --export -a "Your Name" > pubkey.asc

# Export to keyring
gpg --export > ~/.gnupg/pubring.gpg
```

### Signing Charts

```bash
# Package and sign chart
helm package --sign --key "Your Name" --keyring ~/.gnupg/pubring.gpg ./my-chart

# Output:
# Successfully packaged chart and saved it to: my-chart-1.0.0.tgz
# Successfully signed chart and saved it to: my-chart-1.0.0.tgz.prov

# The .prov file contains:
# - Chart metadata hash
# - GPG signature
```

### Verifying Charts

```bash
# Verify chart signature
helm verify my-chart-1.0.0.tgz --keyring ~/.gnupg/pubring.gpg

# Install with verification
helm install my-release my-chart-1.0.0.tgz --verify --keyring ~/.gnupg/pubring.gpg

# Verify from repository
helm install my-release myrepo/my-chart --verify --keyring ~/.gnupg/pubring.gpg
```

---

## Secrets Management

### Encrypted Values with SOPS

```bash
# Install helm-secrets plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Encrypt values file
sops -e secrets.yaml > secrets.enc.yaml

# Use encrypted values
helm secrets install my-release ./my-chart -f secrets.enc.yaml
```

### External Secrets Operator

```yaml
# ExternalSecret resource in chart
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "myapp.fullname" . }}-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: {{ include "myapp.fullname" . }}-secrets
  data:
    - secretKey: database-password
      remoteRef:
        key: myapp/production
        property: db_password
```

### Sealed Secrets

```yaml
# templates/sealed-secret.yaml
{{- if .Values.sealedSecrets.enabled }}
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: {{ include "myapp.fullname" . }}-secrets
  namespace: {{ .Release.Namespace }}
spec:
  encryptedData:
    password: {{ .Values.sealedSecrets.encryptedPassword }}
{{- end }}
```

### Don't Do This

```yaml
# ❌ BAD: Secrets in values.yaml
database:
  password: "mysecretpassword"  # Never do this!

# ❌ BAD: Secrets in templates
env:
  - name: DB_PASSWORD
    value: "hardcoded-password"  # Never do this!
```

### Do This Instead

```yaml
# ✓ GOOD: Reference external secrets
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Values.existingSecret | default (include "myapp.fullname" .) }}
        key: password

# ✓ GOOD: Use secretKeyRef in values
database:
  existingSecret: my-db-secret
  secretKey: password
```

---

## RBAC Configuration

### Helm Service Account

```yaml
# Service account for Helm operations
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: helm-deployer
  namespace: production
rules:
  - apiGroups: ["", "apps", "networking.k8s.io"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "create", "update", "delete"]
    resourceNames: ["sh.helm.release.*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: helm-deployer
  namespace: production
subjects:
  - kind: ServiceAccount
    name: helm-deployer
    namespace: production
roleRef:
  kind: Role
  name: helm-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Namespace-Scoped Deployment

```bash
# Deploy with specific service account
helm install my-release ./my-chart \
  --namespace production \
  --service-account helm-deployer
```

---

## Secure Chart Templates

### Pod Security

```yaml
# templates/deployment.yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: {{ .Values.securityContext.runAsUser | default 1000 }}
        runAsGroup: {{ .Values.securityContext.runAsGroup | default 1000 }}
        fsGroup: {{ .Values.securityContext.fsGroup | default 1000 }}
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
          # ...
```

### Network Policies

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  podSelector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              {{- toYaml .Values.networkPolicy.allowedPods | nindent 14 }}
      ports:
        - protocol: TCP
          port: {{ .Values.service.port }}
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
{{- end }}
```

### Resource Limits

```yaml
# Enforce resource limits
containers:
  - name: app
    resources:
      limits:
        cpu: {{ .Values.resources.limits.cpu | default "500m" }}
        memory: {{ .Values.resources.limits.memory | default "512Mi" }}
      requests:
        cpu: {{ .Values.resources.requests.cpu | default "100m" }}
        memory: {{ .Values.resources.requests.memory | default "128Mi" }}
```

---

## Supply Chain Security

### Verifying Chart Sources

```bash
# Only use trusted repositories
helm repo add bitnami https://charts.bitnami.com/bitnami

# Verify repository authenticity
helm show chart bitnami/nginx

# Check chart provenance
helm pull bitnami/nginx --prov
helm verify nginx-*.tgz
```

### SBOM for Charts

```yaml
# Generate SBOM for chart
# Using syft
syft dir:./my-chart -o spdx-json > chart-sbom.json

# Attach SBOM to OCI artifact
cosign attach sbom --sbom chart-sbom.json \
  ghcr.io/myorg/charts/my-chart:1.0.0
```

### Image Verification in Charts

```yaml
# templates/deployment.yaml
# Use digests instead of tags
containers:
  - name: app
    {{- if .Values.image.digest }}
    image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
    {{- else }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    {{- end }}
```

```yaml
# values.yaml
image:
  repository: myapp
  # Use digest for immutability
  digest: sha256:abc123def456...
  # Or tag (less secure)
  tag: "1.0.0"
```

---

## Security Scanning

### Scanning Charts

```bash
# Trivy for chart scanning
trivy config ./my-chart

# Checkov for Helm charts
checkov -d ./my-chart --framework helm

# Kubesec for generated manifests
helm template my-release ./my-chart | kubesec scan -

# Polaris for best practices
helm template my-release ./my-chart | polaris audit --audit-path -
```

### CI/CD Integration

```yaml
# .github/workflows/security.yaml
name: Security Scan

on:
  push:
    paths:
      - 'charts/**'

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trivy config scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: './charts/my-app'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Checkov scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: ./charts/my-app
          framework: helm

      - name: Template and scan
        run: |
          helm template my-app ./charts/my-app > manifests.yaml
          kubesec scan manifests.yaml
```

---

## Security Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM SECURITY CHECKLIST                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Chart Development:                                                 │
│  □ No hardcoded secrets in values or templates                    │
│  □ Security contexts defined                                       │
│  □ Resource limits set                                             │
│  □ Network policies included                                       │
│  □ Read-only root filesystem                                       │
│  □ Non-root user specified                                         │
│                                                                     │
│  Chart Distribution:                                                │
│  □ Charts are signed                                               │
│  □ Using trusted repositories                                      │
│  □ SBOM generated                                                  │
│  □ Provenance verified                                             │
│                                                                     │
│  Deployment:                                                        │
│  □ RBAC properly configured                                        │
│  □ Secrets managed externally                                      │
│  □ Image digests used                                              │
│  □ Namespace isolation                                             │
│                                                                     │
│  Ongoing:                                                           │
│  □ Regular security scans                                          │
│  □ Dependency updates                                              │
│  □ Audit logging enabled                                           │
│  □ Access reviews                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                SECURITY BEST PRACTICES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Secrets:                                                           │
│  ✓ Never store secrets in values.yaml                              │
│  ✓ Use external secrets management                                 │
│  ✓ Encrypt sensitive values with SOPS                              │
│  ✓ Rotate secrets regularly                                        │
│                                                                     │
│  Charts:                                                            │
│  ✓ Sign charts with GPG                                            │
│  ✓ Verify signatures before install                                │
│  ✓ Scan charts for vulnerabilities                                 │
│  ✓ Use minimal base images                                         │
│                                                                     │
│  Runtime:                                                           │
│  ✓ Apply Pod Security Standards                                    │
│  ✓ Use network policies                                            │
│  ✓ Set resource limits                                             │
│  ✓ Enable audit logging                                            │
│                                                                     │
│  Access:                                                            │
│  ✓ Use namespace-scoped RBAC                                       │
│  ✓ Principle of least privilege                                    │
│  ✓ Regular access reviews                                          │
│  ✓ Separate dev/staging/prod                                       │
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
│  Security Layers:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Chart Integrity  │ Signing, verification, scanning          │   │
│  │ Secrets          │ External management, encryption          │   │
│  │ RBAC             │ Namespace-scoped, least privilege        │   │
│  │ Runtime          │ Pod security, network policies           │   │
│  │ Supply Chain     │ Trusted sources, SBOM, provenance        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Commands:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm package --sign  │ Sign chart                           │   │
│  │ helm verify          │ Verify signature                     │   │
│  │ helm secrets         │ Encrypted values                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Essential Tools:                                                   │
│  • SOPS/helm-secrets for encrypted values                          │
│  • Trivy/Checkov for scanning                                      │
│  • GPG for chart signing                                           │
│  • External Secrets Operator for secrets                           │
│                                                                     │
│  Next: Learn about Helm troubleshooting                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
