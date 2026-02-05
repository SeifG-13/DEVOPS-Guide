# ArgoCD Applications

## What is an Application?

An Application in ArgoCD is a Kubernetes Custom Resource (CRD) that defines the source, destination, and sync settings for a deployment.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APPLICATION CONCEPT                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  An ArgoCD Application connects:                                    │
│                                                                     │
│  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐  │
│  │   SOURCE    │         │  APPLICATION │         │ DESTINATION │  │
│  │             │────────▶│             │────────▶│             │  │
│  │ Git repo    │         │ ArgoCD CRD  │         │ K8s cluster │  │
│  │ Helm chart  │         │             │         │ Namespace   │  │
│  │ Kustomize   │         │ Sync policy │         │             │  │
│  └─────────────┘         └─────────────┘         └─────────────┘  │
│                                                                     │
│  Application defines:                                               │
│  • WHERE to get manifests (source)                                 │
│  • WHERE to deploy (destination)                                   │
│  • HOW to sync (sync policy)                                       │
│  • WHAT project it belongs to (RBAC)                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Application CRD Structure

### Complete Application Spec

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-application
  namespace: argocd
  # Optional: Add finalizer for cascade deletion
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  labels:
    app.kubernetes.io/name: my-application
    environment: production
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: my-channel
spec:
  # Project the application belongs to
  project: default

  # Source of the application manifests
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: apps/my-app

  # Destination cluster and namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Sync policy
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

  # Ignore differences (optional)
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

---

## Source Configuration

### Git Repository Source

```yaml
spec:
  source:
    # Repository URL (HTTPS or SSH)
    repoURL: https://github.com/org/repo.git

    # Git revision: branch, tag, or commit SHA
    targetRevision: HEAD           # Latest commit on default branch
    # targetRevision: main         # Specific branch
    # targetRevision: v1.0.0       # Tag
    # targetRevision: abc1234      # Commit SHA

    # Path to manifests within repository
    path: kubernetes/overlays/production
```

### Multiple Sources (ArgoCD 2.6+)

```yaml
spec:
  sources:
    # Main application manifests
    - repoURL: https://github.com/org/app-config.git
      targetRevision: main
      path: apps/my-app

    # Shared values from another repo
    - repoURL: https://github.com/org/shared-config.git
      targetRevision: main
      ref: values  # Create reference for use in other sources

    # Helm chart with values from multiple sources
    - repoURL: https://github.com/org/helm-charts.git
      targetRevision: main
      path: charts/my-app
      helm:
        valueFiles:
          - $values/common/values.yaml
          - $values/production/values.yaml
```

### Helm Chart Source

```yaml
spec:
  source:
    # Helm repository
    repoURL: https://charts.helm.sh/stable
    chart: nginx
    targetRevision: 1.2.3

    # Helm-specific configuration
    helm:
      # Release name (defaults to application name)
      releaseName: my-nginx

      # Values files
      valueFiles:
        - values.yaml
        - values-production.yaml

      # Inline values
      values: |
        replicaCount: 3
        image:
          tag: latest

      # Individual parameters
      parameters:
        - name: image.tag
          value: v1.0.0
        - name: service.type
          value: LoadBalancer

      # Force string values
      fileParameters:
        - name: config
          path: files/config.json
```

### Helm from Git Repository

```yaml
spec:
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: main
    path: charts/my-app

    helm:
      releaseName: my-app
      valueFiles:
        - values.yaml
        - ../config/values-production.yaml
      parameters:
        - name: image.tag
          value: v2.0.0
```

### Kustomize Source

```yaml
spec:
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: main
    path: kustomize/overlays/production

    kustomize:
      # Kustomize version
      version: v5.0.0

      # Name prefix/suffix
      namePrefix: prod-
      nameSuffix: -v2

      # Image overrides
      images:
        - myapp=myregistry/myapp:v2.0.0
        - nginx=nginx:1.25

      # Common labels
      commonLabels:
        environment: production

      # Common annotations
      commonAnnotations:
        owner: team-a

      # Replicas override
      replicas:
        - name: my-deployment
          count: 5
```

### Directory Source (Plain YAML)

```yaml
spec:
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: main
    path: manifests/production

    directory:
      # Recursive directory scan
      recurse: true

      # Include specific files
      include: '*.yaml'

      # Exclude specific files
      exclude: '*-test.yaml'

      # Jsonnet support
      jsonnet:
        tlas:
          - name: environment
            value: production
        extVars:
          - name: revision
            value: v1.0.0
```

---

## Destination Configuration

### Destination Options

```yaml
spec:
  destination:
    # Target cluster (URL or name)
    server: https://kubernetes.default.svc
    # OR
    # name: production-cluster  # Cluster name registered in ArgoCD

    # Target namespace
    namespace: production
```

### Common Destination Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DESTINATION PATTERNS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Same Cluster (default):                                           │
│  server: https://kubernetes.default.svc                            │
│                                                                     │
│  External Cluster (by URL):                                        │
│  server: https://prod-cluster.example.com:6443                     │
│                                                                     │
│  External Cluster (by name):                                       │
│  name: production-cluster                                          │
│                                                                     │
│  Dynamic Namespace (with sync option):                             │
│  namespace: my-app                                                 │
│  syncPolicy:                                                       │
│    syncOptions:                                                    │
│      - CreateNamespace=true                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Creating Applications

### Method 1: Using the Web UI

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CREATE VIA UI                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Click "NEW APP" button                                         │
│  2. Fill in application details:                                   │
│     • Application Name: my-app                                     │
│     • Project: default                                             │
│     • Sync Policy: Manual or Automatic                             │
│  3. Configure Source:                                              │
│     • Repository URL: https://github.com/org/repo.git              │
│     • Revision: HEAD                                               │
│     • Path: kubernetes/                                            │
│  4. Configure Destination:                                         │
│     • Cluster: https://kubernetes.default.svc                      │
│     • Namespace: default                                           │
│  5. Click "CREATE"                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Method 2: Using the CLI

```bash
# Basic application
argocd app create my-app \
  --repo https://github.com/org/repo.git \
  --path kubernetes/ \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# With auto-sync
argocd app create my-app \
  --repo https://github.com/org/repo.git \
  --path kubernetes/ \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Helm application
argocd app create my-helm-app \
  --repo https://github.com/org/repo.git \
  --path charts/my-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --helm-set image.tag=v1.0.0 \
  --values values-prod.yaml

# Kustomize application
argocd app create my-kustomize-app \
  --repo https://github.com/org/repo.git \
  --path kustomize/overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --kustomize-image myapp=myregistry/myapp:v2.0.0
```

### Method 3: Declarative YAML (Recommended)

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: main
    path: kubernetes/
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
# Apply the application
kubectl apply -f application.yaml

# Or using ArgoCD CLI
argocd app create -f application.yaml
```

---

## Application Examples

### Example 1: Simple Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

### Example 2: Helm Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus
  namespace: argocd
spec:
  project: monitoring
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 45.7.1
    helm:
      releaseName: prometheus
      values: |
        grafana:
          enabled: true
          adminPassword: admin123
        prometheus:
          prometheusSpec:
            retention: 15d
            storageSpec:
              volumeClaimTemplate:
                spec:
                  storageClassName: standard
                  resources:
                    requests:
                      storage: 50Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Example 3: Kustomize with Overlays

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-production
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/org/frontend.git
    targetRevision: main
    path: deploy/overlays/production
    kustomize:
      images:
        - frontend=gcr.io/my-project/frontend:v2.5.0
      commonLabels:
        environment: production
        team: frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: frontend-production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Example 4: Multi-Environment Application

```yaml
# development.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
spec:
  project: development
  source:
    repoURL: https://github.com/org/myapp.git
    targetRevision: develop
    path: deploy/overlays/development
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
# staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
spec:
  project: staging
  source:
    repoURL: https://github.com/org/myapp.git
    targetRevision: release
    path: deploy/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-staging
  syncPolicy:
    automated:
      prune: true
---
# production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/org/myapp.git
    targetRevision: main
    path: deploy/overlays/production
  destination:
    server: https://production-cluster.example.com
    namespace: myapp
  syncPolicy:
    automated:
      prune: false  # Manual prune in production
      selfHeal: true
```

---

## Ignore Differences

### Common Ignore Patterns

```yaml
spec:
  ignoreDifferences:
    # Ignore replicas (managed by HPA)
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

    # Ignore specific annotation
    - group: apps
      kind: Deployment
      jsonPointers:
        - /metadata/annotations/kubectl.kubernetes.io~1last-applied-configuration

    # Ignore all annotations
    - group: "*"
      kind: "*"
      jsonPointers:
        - /metadata/annotations

    # Ignore using JQ path expression
    - group: apps
      kind: Deployment
      jqPathExpressions:
        - .spec.template.spec.containers[].resources

    # Ignore specific resource by name
    - group: ""
      kind: Secret
      name: my-secret
      jsonPointers:
        - /data
```

### Managed Fields Ignore

```yaml
spec:
  ignoreDifferences:
    # Ignore managedFields (useful for server-side apply)
    - group: "*"
      kind: "*"
      managedFieldsManagers:
        - kube-controller-manager
        - cluster-autoscaler
```

---

## Application Health

### Custom Health Checks

```yaml
# In argocd-cm ConfigMap
data:
  resource.customizations.health.certmanager.io_Certificate: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Ready" and condition.status == "False" then
            hs.status = "Degraded"
            hs.message = condition.message
            return hs
          end
          if condition.type == "Ready" and condition.status == "True" then
            hs.status = "Healthy"
            hs.message = condition.message
            return hs
          end
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for certificate"
    return hs
```

---

## Application Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APPLICATION LIFECYCLE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐         │
│  │ Create  │───▶│  Sync   │───▶│ Healthy │───▶│ Update  │         │
│  └─────────┘    └─────────┘    └─────────┘    └────┬────┘         │
│                      ▲                              │               │
│                      │                              │               │
│                      └──────────────────────────────┘               │
│                           (continuous loop)                         │
│                                                                     │
│  States:                                                            │
│  • OutOfSync: Git differs from cluster                             │
│  • Syncing: Applying changes                                       │
│  • Synced: Git matches cluster                                     │
│  • Healthy/Degraded/Progressing: Resource health                   │
│                                                                     │
│  Termination:                                                       │
│  • Delete app (keep resources): argocd app delete myapp            │
│  • Delete app (cascade): argocd app delete myapp --cascade         │
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
│  Application Components:                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ source      │ Where to get manifests (Git, Helm, Kustomize) │   │
│  │ destination │ Where to deploy (cluster, namespace)          │   │
│  │ project     │ RBAC boundary                                 │   │
│  │ syncPolicy  │ How to sync (manual, automated)               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Source Types:                                                      │
│  • Git repository (YAML, Kustomize, Helm, Jsonnet)                 │
│  • Helm repository (chart + values)                                │
│  • OCI registry (Helm charts as OCI artifacts)                     │
│                                                                     │
│  Best Practices:                                                    │
│  • Use declarative YAML for applications                           │
│  • Store application manifests in Git (app-of-apps pattern)        │
│  • Use projects for RBAC boundaries                                │
│  • Configure ignoreDifferences for dynamic fields                  │
│                                                                     │
│  Next: Learn about sync strategies and policies                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
