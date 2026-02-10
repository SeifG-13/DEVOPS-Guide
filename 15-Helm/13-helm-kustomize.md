# Helm with Kustomize

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM + KUSTOMIZE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Why Combine Helm and Kustomize?                                   │
│  • Use Helm for packaging and distribution                         │
│  • Use Kustomize for environment-specific overlays                 │
│  • Best of both worlds                                             │
│                                                                     │
│  Common Patterns:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Helm → Kustomize │ Generate base, customize with overlays   │   │
│  │ Kustomize → Helm │ Use Helm charts as Kustomize bases       │   │
│  │ Post-rendering   │ Helm generates, Kustomize patches        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Helm Post-Rendering with Kustomize

### Basic Setup

```
my-project/
├── chart/                    # Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── kustomize/
│   ├── base/
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   └── kustomization.yaml
│       ├── staging/
│       │   └── kustomization.yaml
│       └── production/
│           └── kustomization.yaml
└── post-renderer.sh
```

### Post-Renderer Script

```bash
#!/bin/bash
# post-renderer.sh

# Read stdin (Helm output), apply Kustomize
cat > /tmp/helm-output.yaml
kustomize build ./kustomize/overlays/${ENVIRONMENT:-dev} < /tmp/helm-output.yaml
```

```bash
# Make executable
chmod +x post-renderer.sh

# Use with Helm
ENVIRONMENT=production helm install my-release ./chart \
  --post-renderer ./post-renderer.sh
```

### Kustomization for Post-Rendering

```yaml
# kustomize/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Read from stdin (Helm output)
resources:
  - /dev/stdin

# Add labels
commonLabels:
  environment: production

# Add annotations
commonAnnotations:
  team: platform

# Patches
patches:
  # Strategic merge patch
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-app
      spec:
        replicas: 5

  # JSON patch
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      - op: add
        path: /spec/template/metadata/annotations
        value:
          prometheus.io/scrape: "true"
```

---

## Kustomize with Helm Charts

### Helm Chart as Kustomize Base

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Use Helm chart generator
helmCharts:
  - name: nginx
    repo: https://charts.bitnami.com/bitnami
    version: 15.0.0
    releaseName: my-nginx
    namespace: production
    valuesFile: values.yaml
    includeCRDs: true

# Additional resources
resources:
  - additional-configmap.yaml

# Patches to apply on top
patches:
  - path: patches/deployment-patch.yaml
```

### Values File for Helm Generator

```yaml
# values.yaml
replicaCount: 3

service:
  type: LoadBalancer

resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

### Build and Apply

```bash
# Build with Kustomize
kustomize build --enable-helm .

# Apply directly
kustomize build --enable-helm . | kubectl apply -f -

# Or use kubectl
kubectl apply -k . --enable-helm
```

---

## ArgoCD Integration

### Helm + Kustomize in ArgoCD

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/my-app
    targetRevision: HEAD
    path: kustomize/overlays/production

    # Enable Helm chart support in Kustomize
    kustomize:
      version: v5.0.0

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Directory Structure for ArgoCD

```
my-app/
├── chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── kustomize/
    ├── base/
    │   ├── kustomization.yaml
    │   └── helm-values.yaml
    └── overlays/
        └── production/
            ├── kustomization.yaml
            └── patches/
```

---

## Advanced Patterns

### Environment-Specific Values

```yaml
# kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: my-app
    repo: https://charts.example.com
    version: 1.0.0
    releaseName: my-app
    valuesFile: values.yaml
```

```yaml
# kustomize/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

# Override Helm values for production
helmCharts:
  - name: my-app
    repo: https://charts.example.com
    version: 1.0.0
    releaseName: my-app
    valuesFile: values-production.yaml
    valuesInline:
      replicaCount: 5
      autoscaling:
        enabled: true

# Additional patches
patches:
  - path: patches/hpa.yaml
```

### Multi-Environment Setup

```
environments/
├── base/
│   ├── kustomization.yaml
│   └── values.yaml
├── dev/
│   ├── kustomization.yaml
│   ├── values.yaml
│   └── patches/
│       └── replicas.yaml
├── staging/
│   ├── kustomization.yaml
│   ├── values.yaml
│   └── patches/
│       └── ingress.yaml
└── production/
    ├── kustomization.yaml
    ├── values.yaml
    └── patches/
        ├── replicas.yaml
        ├── hpa.yaml
        └── pdb.yaml
```

### ConfigMap/Secret Generation

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: my-app
    repo: https://charts.example.com
    version: 1.0.0
    releaseName: my-app
    valuesFile: values.yaml

# Generate ConfigMap from file
configMapGenerator:
  - name: app-config
    files:
      - config.yaml
    options:
      labels:
        app: my-app

# Generate Secret
secretGenerator:
  - name: app-secrets
    envs:
      - secrets.env
    options:
      disableNameSuffixHash: true
```

---

## Comparison: Helm vs Kustomize

```
┌─────────────────────────────────────────────────────────────────────┐
│                 HELM VS KUSTOMIZE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Helm Strengths:                                                    │
│  • Templating with logic                                           │
│  • Package management                                              │
│  • Lifecycle hooks                                                 │
│  • Release management                                              │
│  • Chart dependencies                                              │
│                                                                     │
│  Kustomize Strengths:                                               │
│  • No templating (pure YAML)                                       │
│  • Overlay-based customization                                     │
│  • Built into kubectl                                              │
│  • Strategic merge patches                                         │
│  • Native Kubernetes                                               │
│                                                                     │
│  Combined Benefits:                                                 │
│  • Helm for base + Kustomize for environment patches               │
│  • Separation of concerns                                          │
│  • Best practices from both                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Flux Integration

```yaml
# flux/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: my-app
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: my-app
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: my-charts
        namespace: flux-system
  values:
    replicaCount: 3

  # Post-build with Kustomize
  postRenderers:
    - kustomize:
        patchesStrategicMerge:
          - apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: my-app
            spec:
              template:
                metadata:
                  annotations:
                    sidecar.istio.io/inject: "true"
        images:
          - name: my-app
            newName: myregistry/my-app
            newTag: production
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BEST PRACTICES                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Architecture:                                                      │
│  ✓ Use Helm for application packaging                              │
│  ✓ Use Kustomize for environment-specific patches                  │
│  ✓ Keep base manifests clean and minimal                           │
│  ✓ Layer customizations logically                                  │
│                                                                     │
│  Organization:                                                      │
│  ✓ Clear directory structure                                       │
│  ✓ Separate base from overlays                                     │
│  ✓ Environment-specific values files                               │
│  ✓ Document patch purposes                                         │
│                                                                     │
│  Operations:                                                        │
│  ✓ Test builds locally before deploy                               │
│  ✓ Version control everything                                      │
│  ✓ Use GitOps for deployment                                       │
│  ✓ Keep patches small and focused                                  │
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
│  Integration Methods:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Post-Rendering     │ Helm output → Kustomize patches        │   │
│  │ Helm Generator     │ Kustomize uses Helm as source          │   │
│  │ GitOps Integration │ ArgoCD/Flux with both tools            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Commands:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm --post-renderer    │ Apply post-processing             │   │
│  │ kustomize --enable-helm │ Use Helm charts in Kustomize      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Use Cases:                                                         │
│  • Environment-specific customization                              │
│  • Adding labels/annotations across resources                      │
│  • Strategic patches without modifying charts                      │
│  • Managing multiple environments from single chart                │
│                                                                     │
│  Next: Learn about Helm library charts                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
