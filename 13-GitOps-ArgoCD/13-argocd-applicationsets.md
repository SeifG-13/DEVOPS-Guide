# ArgoCD ApplicationSets

## What is ApplicationSet?

ApplicationSet is a controller that automatically generates ArgoCD Applications from templates.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APPLICATIONSET CONCEPT                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Problem: Managing many similar applications is tedious             │
│                                                                     │
│  Without ApplicationSet:                                            │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐      │
│  │ app-dev    │ │ app-stage  │ │ app-prod   │ │ app-dr     │      │
│  │ (manual)   │ │ (manual)   │ │ (manual)   │ │ (manual)   │      │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘      │
│                                                                     │
│  With ApplicationSet:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ApplicationSet                            │   │
│  │  ┌──────────────┐    ┌────────────────────────────────────┐ │   │
│  │  │  Generator   │───▶│ Template                           │ │   │
│  │  │  (clusters,  │    │ (Application blueprint)            │ │   │
│  │  │   git dirs,  │    └────────────────────────────────────┘ │   │
│  │  │   list)      │                  │                        │   │
│  │  └──────────────┘                  │                        │   │
│  └────────────────────────────────────┼────────────────────────┘   │
│                                       │                             │
│            ┌──────────────────────────┼──────────────────────┐     │
│            │                          │                      │     │
│            ▼                          ▼                      ▼     │
│     ┌────────────┐           ┌────────────┐          ┌────────────┐│
│     │ app-dev    │           │ app-stage  │          │ app-prod   ││
│     │(generated) │           │(generated) │          │(generated) ││
│     └────────────┘           └────────────┘          └────────────┘│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ApplicationSet Structure

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-appset
  namespace: argocd
spec:
  # How to generate parameters
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://dev.example.com

  # Application template
  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/repo.git
        path: 'envs/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: myapp

  # Sync policy for ApplicationSet itself
  syncPolicy:
    preserveResourcesOnDeletion: false
```

---

## Generators

### List Generator

Manually define a list of elements to generate applications.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: environments
spec:
  generators:
    - list:
        elements:
          - cluster: development
            url: https://dev.example.com
            namespace: app-dev
            replicas: "1"
          - cluster: staging
            url: https://staging.example.com
            namespace: app-staging
            replicas: "2"
          - cluster: production
            url: https://prod.example.com
            namespace: app-prod
            replicas: "5"

  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      source:
        repoURL: https://github.com/org/repo.git
        path: 'overlays/{{cluster}}'
        helm:
          parameters:
            - name: replicas
              value: '{{replicas}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
```

### Cluster Generator

Generate applications for registered clusters.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
spec:
  generators:
    # All clusters
    - clusters: {}

    # Or filter by labels
    # - clusters:
    #     selector:
    #       matchLabels:
    #         environment: production

  template:
    metadata:
      name: 'addons-{{name}}'
    spec:
      source:
        repoURL: https://github.com/org/cluster-addons.git
        path: addons
      destination:
        server: '{{server}}'
        namespace: cluster-addons
```

### Cluster with Label Selector

```yaml
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            tier: production
          matchExpressions:
            - key: region
              operator: In
              values:
                - us-east-1
                - us-west-2

  template:
    metadata:
      name: 'monitoring-{{name}}'
      labels:
        cluster: '{{name}}'
        region: '{{metadata.labels.region}}'
```

### Git Directory Generator

Generate applications from directories in a Git repository.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-from-dirs
spec:
  generators:
    - git:
        repoURL: https://github.com/org/apps.git
        revision: HEAD
        directories:
          - path: apps/*
          # Exclude specific directories
          - path: apps/excluded-app
            exclude: true

  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      source:
        repoURL: https://github.com/org/apps.git
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

### Git File Generator

Generate applications from files (JSON/YAML) in Git.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-from-files
spec:
  generators:
    - git:
        repoURL: https://github.com/org/config.git
        revision: HEAD
        files:
          - path: "config/**/config.json"

  template:
    metadata:
      name: '{{app.name}}'
    spec:
      source:
        repoURL: '{{app.repo}}'
        path: '{{app.path}}'
        targetRevision: '{{app.revision}}'
      destination:
        server: '{{app.destination.server}}'
        namespace: '{{app.destination.namespace}}'
```

**config.json example:**
```json
{
  "app": {
    "name": "my-app",
    "repo": "https://github.com/org/my-app.git",
    "path": "kubernetes",
    "revision": "main",
    "destination": {
      "server": "https://kubernetes.default.svc",
      "namespace": "my-app"
    }
  }
}
```

### Matrix Generator

Combine multiple generators (Cartesian product).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: matrix-apps
spec:
  generators:
    - matrix:
        generators:
          # First generator: clusters
          - clusters:
              selector:
                matchLabels:
                  tier: production

          # Second generator: applications
          - list:
              elements:
                - app: frontend
                  path: apps/frontend
                - app: backend
                  path: apps/backend
                - app: database
                  path: apps/database

  # Generates: frontend on each prod cluster,
  #            backend on each prod cluster,
  #            database on each prod cluster
  template:
    metadata:
      name: '{{name}}-{{app}}'
    spec:
      source:
        repoURL: https://github.com/org/apps.git
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{app}}'
```

### Merge Generator

Combine generators with override capability.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: merge-example
spec:
  generators:
    - merge:
        mergeKeys:
          - cluster
        generators:
          # Base configuration
          - list:
              elements:
                - cluster: dev
                  replicas: "1"
                  url: https://dev.example.com
                - cluster: staging
                  replicas: "2"
                  url: https://staging.example.com
                - cluster: production
                  replicas: "5"
                  url: https://prod.example.com

          # Override for production
          - list:
              elements:
                - cluster: production
                  replicas: "10"  # Override replicas for prod

  template:
    metadata:
      name: 'app-{{cluster}}'
    spec:
      source:
        helm:
          parameters:
            - name: replicas
              value: '{{replicas}}'
```

### SCM Provider Generator

Auto-discover repositories from GitHub/GitLab organizations.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: github-org-apps
spec:
  generators:
    - scmProvider:
        github:
          organization: myorg
          tokenRef:
            secretName: github-token
            key: token
        filters:
          - repositoryMatch: ^app-.*
          - pathsExist:
              - kubernetes/

  template:
    metadata:
      name: '{{repository}}'
    spec:
      source:
        repoURL: '{{url}}'
        path: kubernetes/
        targetRevision: '{{branch}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{repository}}'
```

### Pull Request Generator

Create applications for pull requests (preview environments).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: pr-previews
spec:
  generators:
    - pullRequest:
        github:
          owner: myorg
          repo: my-app
          tokenRef:
            secretName: github-token
            key: token
        filters:
          - branchMatch: "^feature-.*"

  template:
    metadata:
      name: 'preview-{{number}}'
      labels:
        pr-number: '{{number}}'
    spec:
      source:
        repoURL: https://github.com/myorg/my-app.git
        targetRevision: '{{head_sha}}'
        path: kubernetes/
        helm:
          parameters:
            - name: ingress.host
              value: 'pr-{{number}}.preview.example.com'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'preview-{{number}}'
      syncPolicy:
        automated:
          prune: true
```

---

## Template Variables

### Available Variables by Generator

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TEMPLATE VARIABLES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  List Generator:                                                    │
│  • Any key defined in elements                                     │
│                                                                     │
│  Cluster Generator:                                                 │
│  • {{name}}     - Cluster name                                     │
│  • {{server}}   - Cluster API URL                                  │
│  • {{metadata.labels.<key>}} - Cluster labels                      │
│  • {{metadata.annotations.<key>}} - Cluster annotations            │
│                                                                     │
│  Git Directory Generator:                                          │
│  • {{path}}          - Full path                                   │
│  • {{path.basename}} - Directory name                              │
│  • {{path[n]}}       - nth path segment                            │
│                                                                     │
│  Git File Generator:                                                │
│  • Any key from the JSON/YAML file                                 │
│                                                                     │
│  Pull Request Generator:                                            │
│  • {{number}}    - PR number                                       │
│  • {{branch}}    - Source branch                                   │
│  • {{head_sha}}  - Latest commit SHA                               │
│  • {{head_short_sha}} - Short SHA                                  │
│  • {{labels}}    - PR labels                                       │
│                                                                     │
│  SCM Provider Generator:                                            │
│  • {{repository}} - Repository name                                │
│  • {{url}}        - Clone URL                                      │
│  • {{branch}}     - Default branch                                 │
│  • {{organization}} - Org/owner name                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Sync Policy

### ApplicationSet Sync Policy

```yaml
spec:
  syncPolicy:
    # Don't delete apps when ApplicationSet is deleted
    preserveResourcesOnDeletion: true

    # How to handle Applications
    applicationsSync: create-only  # create-only, create-update, create-delete
```

### Application Template Sync Policy

```yaml
spec:
  template:
    spec:
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## Progressive Rollout

Roll out changes progressively across clusters.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: progressive-rollout
spec:
  generators:
    - list:
        elements:
          - cluster: canary
            url: https://canary.example.com
            wave: "1"
          - cluster: region-1
            url: https://region1.example.com
            wave: "2"
          - cluster: region-2
            url: https://region2.example.com
            wave: "2"
          - cluster: global
            url: https://global.example.com
            wave: "3"

  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: wave
              operator: In
              values:
                - "1"
        - matchExpressions:
            - key: wave
              operator: In
              values:
                - "2"
        - matchExpressions:
            - key: wave
              operator: In
              values:
                - "3"

  template:
    metadata:
      name: 'app-{{cluster}}'
      labels:
        wave: '{{wave}}'
```

---

## Real-World Examples

### Multi-Tenant Platform

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-apps
spec:
  generators:
    - git:
        repoURL: https://github.com/platform/tenants.git
        revision: HEAD
        files:
          - path: "tenants/*/tenant.yaml"

  template:
    metadata:
      name: '{{tenant.name}}'
    spec:
      project: '{{tenant.project}}'
      source:
        repoURL: '{{tenant.repo}}'
        path: '{{tenant.path}}'
        targetRevision: '{{tenant.branch}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'tenant-{{tenant.name}}'
      syncPolicy:
        automated:
          prune: true
        syncOptions:
          - CreateNamespace=true
```

### Microservices Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - env: dev
                  cluster: https://dev.example.com
                - env: prod
                  cluster: https://prod.example.com
          - git:
              repoURL: https://github.com/org/services.git
              directories:
                - path: services/*

  template:
    metadata:
      name: '{{path.basename}}-{{env}}'
    spec:
      source:
        repoURL: https://github.com/org/services.git
        path: '{{path}}/overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: '{{path.basename}}'
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APPLICATIONSET BEST PRACTICES                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Naming:                                                            │
│  • Use consistent naming patterns                                  │
│  • Include environment/cluster in app names                        │
│  • Avoid name collisions                                           │
│                                                                     │
│  Generators:                                                        │
│  • Start with simple generators, add complexity as needed          │
│  • Use Matrix for combining dimensions                             │
│  • Use Merge for overrides                                         │
│                                                                     │
│  Safety:                                                            │
│  • Set preserveResourcesOnDeletion for production                  │
│  • Test ApplicationSets in dev first                               │
│  • Use progressive rollout for production                          │
│                                                                     │
│  Organization:                                                      │
│  • One ApplicationSet per logical group                            │
│  • Use labels for filtering                                        │
│  • Document generator patterns                                     │
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
│  Generators:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ List        │ Manual list of parameters                     │   │
│  │ Cluster     │ Registered ArgoCD clusters                    │   │
│  │ Git Dir     │ Directories in Git repo                       │   │
│  │ Git File    │ JSON/YAML files in Git                        │   │
│  │ Matrix      │ Cartesian product of generators               │   │
│  │ Merge       │ Combine with overrides                        │   │
│  │ SCM Provider│ Auto-discover repos from GitHub/GitLab        │   │
│  │ Pull Request│ Preview environments for PRs                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Benefits:                                                      │
│  • Reduce duplication                                              │
│  • Automated app management                                        │
│  • Consistent deployments                                          │
│  • Dynamic environments (PR previews)                              │
│                                                                     │
│  Next: Configure notifications for deployment events               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
