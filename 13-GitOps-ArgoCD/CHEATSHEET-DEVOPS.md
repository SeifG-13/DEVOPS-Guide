# GitOps & ArgoCD Cheat Sheet for DevOps Engineers

## Quick Reference - GitOps Principles

### Core Principles
1. **Declarative**: Entire system described declaratively
2. **Versioned**: Canonical desired state versioned in Git
3. **Automated**: Approved changes auto-applied to system
4. **Reconciled**: Software agents ensure correctness and alert on drift

### GitOps Workflow
```
Developer → Git Push → Git Repository → ArgoCD → Kubernetes Cluster
                              ↓
                      Pull & Compare
                              ↓
                      Sync if Different
```

---

## ArgoCD CLI Commands

### Installation
```bash
# Install ArgoCD in Kubernetes
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Install CLI
brew install argocd  # macOS
# or download from GitHub releases

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login
argocd login localhost:8080
```

### Application Management
```bash
# Create application
argocd app create myapp \
  --repo https://github.com/org/repo.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# List applications
argocd app list

# Get application details
argocd app get myapp

# Sync application
argocd app sync myapp

# Force sync (override)
argocd app sync myapp --force

# Auto-sync enable
argocd app set myapp --sync-policy automated

# Rollback
argocd app rollback myapp <revision>

# Delete application
argocd app delete myapp

# Application diff
argocd app diff myapp
```

### Repository Management
```bash
# Add repository
argocd repo add https://github.com/org/repo.git --username user --password token

# Add private repo with SSH
argocd repo add git@github.com:org/repo.git --ssh-private-key-path ~/.ssh/id_rsa

# List repositories
argocd repo list
```

### Cluster Management
```bash
# Add external cluster
argocd cluster add my-cluster-context

# List clusters
argocd cluster list
```

---

## Application Manifests

### Basic Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application with Helm
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: helm/myapp
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "1.2.3"
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
```

### Application with Kustomize
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: kustomize/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
```

### ApplicationSet (Multi-Environment)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: myapp-dev
          - env: staging
            namespace: myapp-staging
          - env: prod
            namespace: myapp-prod
  template:
    metadata:
      name: 'myapp-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/repo.git
        targetRevision: HEAD
        path: 'k8s/overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Repository Structure

### Standard GitOps Repo Structure
```
gitops-repo/
├── apps/                    # ArgoCD Application definitions
│   ├── myapp.yaml
│   └── another-app.yaml
├── base/                    # Base Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/                # Environment-specific overrides
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    └── production/
        ├── kustomization.yaml
        └── patches/
```

### App-of-Apps Pattern
```yaml
# apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Interview Q&A

### Q1: What is GitOps?
**A:** GitOps is a way of implementing continuous deployment:
- Git as single source of truth
- Declarative infrastructure and applications
- Pull-based deployment (agents pull from Git)
- Continuous reconciliation (drift detection)

### Q2: What is the difference between push and pull deployment?
**A:**
- **Push (traditional CI/CD)**: CI system pushes to cluster (kubectl apply)
- **Pull (GitOps)**: Agent in cluster pulls from Git and applies
Pull is more secure (no cluster credentials in CI) and self-healing.

### Q3: What is ArgoCD sync?
**A:** Sync compares Git state with cluster state and applies differences:
- **In Sync**: Cluster matches Git
- **Out of Sync**: Cluster differs from Git
- **Sync**: Apply Git state to cluster

### Q4: What is self-healing in ArgoCD?
**A:** ArgoCD automatically reverts manual changes to match Git:
```yaml
syncPolicy:
  automated:
    selfHeal: true  # Revert manual changes
    prune: true     # Delete resources not in Git
```

### Q5: How do you handle secrets in GitOps?
**A:**
- **Sealed Secrets**: Encrypt secrets for Git
- **External Secrets Operator**: Pull from Vault/AWS SM
- **SOPS**: Encrypt files in Git
- **Vault Agent**: Inject at runtime

### Q6: What is an ApplicationSet?
**A:** Automates creating multiple Applications from templates:
- List generator: From defined list
- Git generator: From Git directory structure
- Cluster generator: For multi-cluster
- Matrix: Combine generators

### Q7: How do you promote changes across environments?
**A:**
1. **Branch-per-environment**: main→staging→prod branches
2. **Directory-per-environment**: overlays/dev, overlays/prod
3. **Repository-per-environment**: Separate repos
4. **PR-based promotion**: PR from dev to prod overlay

### Q8: What is the App of Apps pattern?
**A:** Root application that manages other applications:
- Single entry point
- Declarative app management
- Easy to add/remove apps
- Consistent configuration

### Q9: How do you handle rollbacks in GitOps?
**A:**
- Git revert: Create new commit reverting changes
- ArgoCD rollback: `argocd app rollback myapp <revision>`
- Git history is audit trail

### Q10: What are sync waves and hooks?
**A:**
- **Sync waves**: Order resource creation (databases before apps)
- **Hooks**: Run jobs at sync phases (PreSync, Sync, PostSync)

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # Runs first
    argocd.argoproj.io/hook: PreSync    # Run before sync
```

---

## Sync Policies

### Automated Sync
```yaml
syncPolicy:
  automated:
    prune: true        # Delete resources not in Git
    selfHeal: true     # Revert manual changes
    allowEmpty: false  # Don't sync empty commits
  syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

### Sync Waves
```yaml
# Database first (wave -1)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

# Application second (wave 0, default)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"

# Migration job last (wave 1)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

---

## Best Practices

1. **One repo per team/domain** - Not one giant monorepo
2. **Environment isolation** - Separate directories or branches
3. **Use Kustomize or Helm** - Reduce duplication
4. **Implement RBAC** - ArgoCD projects for isolation
5. **Seal your secrets** - Never plain text in Git
6. **Use sync waves** - Order deployments properly
7. **Enable notifications** - Slack/Teams for sync status
8. **Implement health checks** - Custom health for CRDs
