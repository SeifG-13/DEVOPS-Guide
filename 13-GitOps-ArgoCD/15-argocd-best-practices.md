# ArgoCD Best Practices

## Repository Structure

### Mono-Repo vs Multi-Repo

```
┌─────────────────────────────────────────────────────────────────────┐
│                    REPOSITORY STRATEGIES                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MONO-REPO (Single repository for all configs)                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ gitops-repo/                                                │   │
│  │ ├── apps/                                                   │   │
│  │ │   ├── frontend/                                          │   │
│  │ │   ├── backend/                                           │   │
│  │ │   └── database/                                          │   │
│  │ ├── infrastructure/                                         │   │
│  │ │   ├── monitoring/                                        │   │
│  │ │   └── ingress/                                           │   │
│  │ └── clusters/                                               │   │
│  │     ├── dev/                                               │   │
│  │     ├── staging/                                           │   │
│  │     └── production/                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ✓ Single source of truth                                          │
│  ✓ Atomic changes across apps                                      │
│  ✗ Large repo, complex permissions                                 │
│                                                                     │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                     │
│  MULTI-REPO (Separate repos per app/team)                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ frontend-config/      backend-config/      infra-config/    │   │
│  │ ├── base/            ├── base/            ├── monitoring/   │   │
│  │ └── overlays/        └── overlays/        └── ingress/      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ✓ Team autonomy                                                   │
│  ✓ Simpler permissions                                             │
│  ✗ Harder to track cross-app changes                               │
│                                                                     │
│  RECOMMENDATION: Start with mono-repo, split as needed             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Recommended Directory Structure

```
gitops-repo/
├── apps/                          # Application configs
│   ├── base/                      # Base configs (shared)
│   │   ├── frontend/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── backend/
│   └── overlays/                  # Environment-specific
│       ├── development/
│       │   ├── frontend/
│       │   │   ├── kustomization.yaml
│       │   │   └── replicas-patch.yaml
│       │   └── backend/
│       ├── staging/
│       └── production/
│
├── infrastructure/                # Platform components
│   ├── monitoring/
│   ├── logging/
│   ├── ingress/
│   └── cert-manager/
│
├── argocd/                        # ArgoCD configs
│   ├── applications/              # Application CRDs
│   ├── applicationsets/           # ApplicationSet CRDs
│   └── projects/                  # AppProject CRDs
│
└── clusters/                      # Cluster-specific configs
    ├── dev-cluster/
    ├── staging-cluster/
    └── prod-cluster/
```

---

## App-of-Apps Pattern

### Root Application

```yaml
# argocd/applications/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: HEAD
    path: argocd/applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Child Applications

```yaml
# argocd/applications/frontend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: HEAD
    path: apps/overlays/production/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APP-OF-APPS PATTERN                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                       ┌───────────────┐                            │
│                       │   Root App    │                            │
│                       │               │                            │
│                       │ Manages all   │                            │
│                       │ Application   │                            │
│                       │ CRDs          │                            │
│                       └───────┬───────┘                            │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐              │
│         │                     │                     │              │
│         ▼                     ▼                     ▼              │
│  ┌────────────┐       ┌────────────┐       ┌────────────┐         │
│  │  Frontend  │       │  Backend   │       │  Database  │         │
│  │    App     │       │    App     │       │    App     │         │
│  └────────────┘       └────────────┘       └────────────┘         │
│         │                     │                     │              │
│         ▼                     ▼                     ▼              │
│  ┌────────────┐       ┌────────────┐       ┌────────────┐         │
│  │ Kubernetes │       │ Kubernetes │       │ Kubernetes │         │
│  │ Resources  │       │ Resources  │       │ Resources  │         │
│  └────────────┘       └────────────┘       └────────────┘         │
│                                                                     │
│  Benefits:                                                          │
│  • Declarative management of applications                          │
│  • Self-healing ArgoCD configuration                               │
│  • GitOps for GitOps                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Environment Promotion

### GitOps Promotion Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PROMOTION FLOW                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Developer merges PR to main branch                             │
│  2. CI builds and pushes image (e.g., v1.2.3)                      │
│  3. CI opens PR to update dev overlay                              │
│  4. Auto-merge to dev → ArgoCD syncs                               │
│  5. After testing, PR to update staging overlay                    │
│  6. After approval, PR to update production overlay                │
│                                                                     │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │  Code   │───▶│  Build  │───▶│   Dev   │───▶│ Staging │───▶ Prod│
│  │  Merge  │    │  Image  │    │         │    │         │        │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘        │
│                      │               │              │              │
│                      ▼               ▼              ▼              │
│                  registry     kustomization  kustomization        │
│                  v1.2.3       image: v1.2.3  image: v1.2.3        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Image Updater Integration

```yaml
# Application with image updater annotation
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-dev
  annotations:
    argocd-image-updater.argoproj.io/image-list: frontend=myregistry/frontend
    argocd-image-updater.argoproj.io/frontend.update-strategy: semver
    argocd-image-updater.argoproj.io/write-back-method: git
spec:
  source:
    repoURL: https://github.com/org/gitops-repo.git
    path: apps/overlays/development/frontend
```

---

## Security Best Practices

### RBAC Configuration

```yaml
# Least privilege project
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
spec:
  # Restrict sources
  sourceRepos:
    - https://github.com/org/frontend-*

  # Restrict destinations
  destinations:
    - namespace: frontend-*
      server: https://kubernetes.default.svc

  # Restrict cluster resources
  clusterResourceWhitelist: []  # No cluster resources

  # Namespace resources allowed
  namespaceResourceWhitelist:
    - group: ''
      kind: ConfigMap
    - group: ''
      kind: Secret
    - group: apps
      kind: Deployment
    - group: ''
      kind: Service

  # RBAC roles
  roles:
    - name: developer
      policies:
        - p, proj:team-frontend:developer, applications, get, team-frontend/*, allow
        - p, proj:team-frontend:developer, applications, sync, team-frontend/*, allow
      groups:
        - frontend-developers
```

### Secret Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRETS BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DO:                                                                │
│  ✓ Use Sealed Secrets, SOPS, or External Secrets                   │
│  ✓ Rotate secrets regularly                                        │
│  ✓ Use different secrets per environment                           │
│  ✓ Audit secret access                                             │
│  ✓ Use short-lived tokens where possible                           │
│                                                                     │
│  DON'T:                                                             │
│  ✗ Commit plain secrets to Git                                     │
│  ✗ Share secrets across environments                               │
│  ✗ Use long-lived credentials                                      │
│  ✗ Store secrets in ConfigMaps                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Network Security

```yaml
# Restrict ArgoCD to specific namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: argocd-server
  namespace: argocd
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8080
        - port: 8083
```

---

## High Availability

### HA Configuration

```yaml
# values.yaml for Helm installation
controller:
  replicas: 2
  env:
    - name: ARGOCD_CONTROLLER_REPLICAS
      value: "2"

server:
  replicas: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5

repoServer:
  replicas: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5

redis-ha:
  enabled: true
  replicas: 3
```

### Resource Limits

```yaml
controller:
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

server:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

repoServer:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
```

---

## Performance Optimization

### Application Controller Tuning

```yaml
# argocd-cmd-params-cm ConfigMap
data:
  # Increase status processors
  controller.status.processors: "50"

  # Increase operation processors
  controller.operation.processors: "25"

  # Reduce reconciliation frequency
  controller.self.heal.timeout.seconds: "5"

  # Repo server parallelism
  reposerver.parallelism.limit: "10"
```

### Sharding for Large Scale

```yaml
# Shard applications across controller replicas
controller:
  replicas: 3
  env:
    - name: ARGOCD_CONTROLLER_REPLICAS
      value: "3"
    # Each controller handles specific clusters
```

---

## Disaster Recovery

### Backup Strategy

```bash
# Backup all ArgoCD resources
kubectl get applications -n argocd -o yaml > applications-backup.yaml
kubectl get appprojects -n argocd -o yaml > projects-backup.yaml
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository -o yaml > repos-backup.yaml
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster -o yaml > clusters-backup.yaml

# Backup ConfigMaps
kubectl get configmap argocd-cm -n argocd -o yaml > argocd-cm-backup.yaml
kubectl get configmap argocd-rbac-cm -n argocd -o yaml > argocd-rbac-cm-backup.yaml
```

### Recovery Procedure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DISASTER RECOVERY                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Install ArgoCD on new cluster                                  │
│  2. Restore ConfigMaps                                             │
│  3. Restore repository secrets                                     │
│  4. Restore cluster secrets                                        │
│  5. Restore AppProjects                                            │
│  6. Restore Applications (or let root-app recreate them)           │
│                                                                     │
│  Best Practice: Store all ArgoCD config in Git (GitOps for GitOps) │
│  Recovery = Install ArgoCD + Apply root-app                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Monitoring ArgoCD

### Key Metrics

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  endpoints:
    - port: metrics
```

### Important Metrics to Monitor

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KEY METRICS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Application Metrics:                                               │
│  • argocd_app_info                    - App state                  │
│  • argocd_app_sync_total              - Sync count                 │
│  • argocd_app_health_status           - Health status              │
│                                                                     │
│  Controller Metrics:                                                │
│  • argocd_app_reconcile_count         - Reconciliation count       │
│  • argocd_app_reconcile_bucket        - Reconciliation duration    │
│  • workqueue_depth                    - Queue depth                │
│                                                                     │
│  Repo Server Metrics:                                               │
│  • argocd_git_request_total           - Git operations             │
│  • argocd_git_request_duration_seconds- Git latency                │
│                                                                     │
│  Alerts to Configure:                                               │
│  • Application OutOfSync > 30 minutes                              │
│  • Application Degraded                                            │
│  • Sync failures                                                   │
│  • High reconciliation queue depth                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Anti-Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ANTI-PATTERNS TO AVOID                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✗ Manual kubectl changes                                          │
│    → Always use Git, let ArgoCD sync                               │
│                                                                     │
│  ✗ Disabling self-heal in production                               │
│    → Enable self-heal, use ignoreDifferences for exceptions        │
│                                                                     │
│  ✗ Using default project for everything                            │
│    → Create specific projects with restrictions                    │
│                                                                     │
│  ✗ Hardcoding environment-specific values                          │
│    → Use Kustomize overlays or Helm values                         │
│                                                                     │
│  ✗ Mixing app code and deployment config                           │
│    → Separate repositories                                         │
│                                                                     │
│  ✗ Not validating manifests before commit                          │
│    → Use CI to validate YAML, run kubeval/kubeconform              │
│                                                                     │
│  ✗ Ignoring sync failures                                          │
│    → Set up notifications, monitor metrics                         │
│                                                                     │
│  ✗ Overly complex ApplicationSets                                  │
│    → Start simple, add complexity when needed                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ARGOCD IMPLEMENTATION CHECKLIST                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Setup:                                                             │
│  □ Install ArgoCD (HA for production)                              │
│  □ Configure SSO authentication                                    │
│  □ Set up RBAC policies                                            │
│  □ Configure TLS/ingress                                           │
│  □ Add repository credentials                                      │
│                                                                     │
│  Configuration:                                                     │
│  □ Create AppProjects per team/environment                         │
│  □ Set up notifications                                            │
│  □ Configure sync windows (if needed)                              │
│  □ Set up secrets management                                       │
│                                                                     │
│  Operations:                                                        │
│  □ Monitor ArgoCD metrics                                          │
│  □ Set up alerts for sync failures                                 │
│  □ Document backup/recovery procedures                             │
│  □ Test disaster recovery                                          │
│                                                                     │
│  Best Practices:                                                    │
│  □ Use declarative Application definitions                         │
│  □ Implement app-of-apps pattern                                   │
│  □ Validate manifests in CI                                        │
│  □ Use Kustomize/Helm for environment differences                  │
│  □ Never commit plain secrets                                      │
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
│  Key Best Practices:                                                │
│                                                                     │
│  Repository:                                                        │
│  • Use mono-repo for simplicity or multi-repo for autonomy         │
│  • Implement app-of-apps pattern                                   │
│  • Separate base configs from environment overlays                 │
│                                                                     │
│  Security:                                                          │
│  • Create restrictive AppProjects                                  │
│  • Use external secrets management                                 │
│  • Enable SSO, disable local admin                                 │
│                                                                     │
│  Operations:                                                        │
│  • Enable HA for production                                        │
│  • Monitor metrics and set up alerts                               │
│  • Have documented DR procedures                                   │
│                                                                     │
│  Development:                                                       │
│  • Validate manifests in CI                                        │
│  • Use GitOps promotion workflow                                   │
│  • Enable self-heal for drift correction                           │
│                                                                     │
│  Remember: Everything in Git, Git is the source of truth!          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
