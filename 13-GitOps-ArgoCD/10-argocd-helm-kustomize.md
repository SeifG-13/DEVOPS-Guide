# ArgoCD with Helm and Kustomize

## Overview

ArgoCD natively supports both Helm and Kustomize for templating Kubernetes manifests.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MANIFEST GENERATION                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ArgoCD Repo Server                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │   │
│  │  │  Plain  │    │  Helm   │    │Kustomize│    │ Jsonnet │ │   │
│  │  │  YAML   │    │         │    │         │    │         │ │   │
│  │  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘ │   │
│  │       │              │              │              │       │   │
│  │       └──────────────┴──────────────┴──────────────┘       │   │
│  │                          │                                  │   │
│  │                          ▼                                  │   │
│  │              ┌─────────────────────┐                       │   │
│  │              │  Kubernetes YAML    │                       │   │
│  │              │     Manifests       │                       │   │
│  │              └─────────────────────┘                       │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Helm Integration

### Deploying from Helm Repository

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
  namespace: argocd
spec:
  project: default
  source:
    # Helm repository URL
    repoURL: https://kubernetes.github.io/ingress-nginx
    # Chart name
    chart: ingress-nginx
    # Chart version
    targetRevision: 4.7.1

    helm:
      # Release name (defaults to app name)
      releaseName: nginx-ingress

      # Values file (from same repo)
      valueFiles:
        - values.yaml

      # Inline values
      values: |
        controller:
          replicaCount: 2
          service:
            type: LoadBalancer

      # Individual parameters
      parameters:
        - name: controller.replicaCount
          value: "3"

  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx

  syncPolicy:
    automated:
      prune: true
    syncOptions:
      - CreateNamespace=true
```

### Deploying Helm Chart from Git

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/helm-charts.git
    targetRevision: main
    path: charts/my-app

    helm:
      releaseName: my-app

      # Multiple values files
      valueFiles:
        - values.yaml
        - values-production.yaml

      # Parameters override values files
      parameters:
        - name: image.tag
          value: v2.0.0
        - name: replicas
          value: "5"

      # Force string values
      parameters:
        - name: podAnnotations.prometheus\.io/scrape
          value: "true"
          forceString: true

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### Helm Values Precedence

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VALUES PRECEDENCE (lowest to highest)             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Chart's default values.yaml                                    │
│  2. valueFiles (in order listed)                                   │
│  3. values (inline YAML)                                           │
│  4. parameters (highest priority)                                  │
│                                                                     │
│  Example:                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Chart default:     replicas: 1                              │   │
│  │ values.yaml:       replicas: 2                              │   │
│  │ values-prod.yaml:  replicas: 3                              │   │
│  │ inline values:     replicas: 4                              │   │
│  │ parameters:        replicas: 5  ← This wins!                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Helm Options

```yaml
spec:
  source:
    helm:
      # Pass credentials to Helm for private registries
      passCredentials: true

      # Skip CRD installation
      skipCrds: false

      # Ignore missing value files
      ignoreMissingValueFiles: true

      # Helm version (v2 or v3)
      version: v3

      # File parameters (load from file)
      fileParameters:
        - name: config
          path: files/config.json

      # Value files from external repositories
      valueFiles:
        - $values/common/values.yaml

      # Release name
      releaseName: my-release

      # Kubernetes API versions to pass to Helm
      kubeVersion: "1.28.0"
```

### Multiple Value Sources

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  sources:
    # Values from separate repository
    - repoURL: https://github.com/myorg/values-repo.git
      targetRevision: main
      ref: values

    # Helm chart with external values
    - repoURL: https://github.com/myorg/helm-charts.git
      targetRevision: main
      path: charts/my-app
      helm:
        valueFiles:
          - $values/environments/production/values.yaml
          - $values/apps/my-app/values.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

---

## Kustomize Integration

### Basic Kustomize Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-kustomize-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/kustomize-repo.git
    targetRevision: main
    path: overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### Kustomize with Options

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  source:
    repoURL: https://github.com/myorg/repo.git
    path: kustomize/overlays/production

    kustomize:
      # Kustomize version
      version: v5.0.0

      # Add name prefix to all resources
      namePrefix: prod-

      # Add name suffix to all resources
      nameSuffix: -v2

      # Override images
      images:
        - myapp=gcr.io/myproject/myapp:v2.0.0
        - nginx=nginx:1.25-alpine

      # Add common labels
      commonLabels:
        environment: production
        team: platform

      # Add common annotations
      commonAnnotations:
        owner: platform-team
        contact: platform@example.com

      # Force common labels on all resources
      forceCommonLabels: true

      # Force common annotations on all resources
      forceCommonAnnotations: true

      # Replica count overrides
      replicas:
        - name: my-deployment
          count: 5
        - name: worker-deployment
          count: 3

      # Load .env file as ConfigMap generator
      # (requires kustomization.yaml support)

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### Kustomize Directory Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KUSTOMIZE STRUCTURE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  my-app/                                                            │
│  ├── base/                                                          │
│  │   ├── kustomization.yaml                                        │
│  │   ├── deployment.yaml                                           │
│  │   ├── service.yaml                                              │
│  │   └── configmap.yaml                                            │
│  └── overlays/                                                      │
│      ├── development/                                               │
│      │   ├── kustomization.yaml                                    │
│      │   └── replicas-patch.yaml                                   │
│      ├── staging/                                                   │
│      │   ├── kustomization.yaml                                    │
│      │   └── namespace.yaml                                        │
│      └── production/                                                │
│          ├── kustomization.yaml                                    │
│          ├── replicas-patch.yaml                                   │
│          └── hpa.yaml                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Example Kustomization Files

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: my-app
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base
  - hpa.yaml

patches:
  - path: replicas-patch.yaml

images:
  - name: my-app
    newName: gcr.io/myproject/my-app
    newTag: v2.0.0

configMapGenerator:
  - name: app-config
    envs:
      - config.env
```

```yaml
# overlays/production/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
```

---

## Combining Helm and Kustomize

### Helm with Kustomize Post-Rendering

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-with-kustomize
spec:
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.7.1

    helm:
      releaseName: nginx

    # Apply Kustomize after Helm rendering
    kustomize:
      images:
        - registry.k8s.io/ingress-nginx/controller=my-registry/ingress-nginx:v1.8.0

  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
```

### Kustomize + Helm Charts

```yaml
# kustomization.yaml with Helm chart
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.7.1
    releaseName: nginx
    namespace: ingress-nginx
    valuesFile: values.yaml

patches:
  - patch: |-
      - op: add
        path: /metadata/annotations/custom
        value: my-annotation
    target:
      kind: Deployment
```

---

## Environment-Specific Deployments

### Pattern: Directory per Environment

```yaml
# Application per environment
---
# dev-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
spec:
  source:
    repoURL: https://github.com/myorg/repo.git
    path: kustomize/overlays/development
  destination:
    namespace: dev
---
# staging-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
spec:
  source:
    repoURL: https://github.com/myorg/repo.git
    path: kustomize/overlays/staging
  destination:
    namespace: staging
---
# prod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
spec:
  source:
    repoURL: https://github.com/myorg/repo.git
    path: kustomize/overlays/production
  destination:
    namespace: production
```

### Pattern: Helm Values per Environment

```
repo/
├── charts/
│   └── my-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── environments/
│   ├── dev/
│   │   └── values.yaml
│   ├── staging/
│   │   └── values.yaml
│   └── production/
│       └── values.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
spec:
  source:
    repoURL: https://github.com/myorg/repo.git
    path: charts/my-app
    helm:
      valueFiles:
        - ../../environments/production/values.yaml
```

---

## CLI Operations

### Helm Commands

```bash
# Create Helm app
argocd app create my-helm-app \
  --repo https://charts.helm.sh/stable \
  --helm-chart nginx \
  --revision 1.2.3 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Set Helm values
argocd app set my-app --helm-set replicas=3
argocd app set my-app --helm-set-string version="1.0"
argocd app set my-app --values values-prod.yaml

# View Helm parameters
argocd app get my-app --show-params
```

### Kustomize Commands

```bash
# Create Kustomize app
argocd app create my-kustomize-app \
  --repo https://github.com/myorg/repo.git \
  --path kustomize/overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production

# Set Kustomize options
argocd app set my-app --kustomize-image myapp=myregistry/myapp:v2.0.0
argocd app set my-app --nameprefix prod-
argocd app set my-app --namesuffix -v2
```

### Preview Manifests

```bash
# Preview Helm manifests
argocd app manifests my-helm-app --source git

# Preview Kustomize manifests
argocd app manifests my-kustomize-app --source git

# Compare with live
argocd app diff my-app
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BEST PRACTICES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Helm Best Practices:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Pin chart versions (don't use latest)                     │   │
│  │ • Store values files in Git                                 │   │
│  │ • Use parameters for secrets/dynamic values                 │   │
│  │ • Test charts with helm template locally                    │   │
│  │ • Use multiple values files for layering                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Kustomize Best Practices:                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Use base + overlays pattern                               │   │
│  │ • Keep base generic, overlays specific                      │   │
│  │ • Use strategic merge patches over JSON patches             │   │
│  │ • Pin image tags in overlays                                │   │
│  │ • Use commonLabels for consistency                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  General:                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Validate manifests before commit                          │   │
│  │ • Use CI to test rendering                                  │   │
│  │ • Document values and their purposes                        │   │
│  │ • Keep environment differences minimal                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
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
│  Helm Integration:                                                  │
│  • Deploy from Helm repos or Git                                   │
│  • Use valueFiles, values, and parameters                          │
│  • Multiple value sources supported                                │
│  • Values precedence: files < inline < parameters                  │
│                                                                     │
│  Kustomize Integration:                                             │
│  • Native support for overlays                                     │
│  • Image overrides                                                 │
│  • Name prefix/suffix                                              │
│  • Common labels/annotations                                       │
│  • Replica overrides                                               │
│                                                                     │
│  Combining Both:                                                    │
│  • Kustomize post-rendering for Helm                               │
│  • Helm charts in Kustomization                                    │
│                                                                     │
│  Next: Learn about secrets management                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
