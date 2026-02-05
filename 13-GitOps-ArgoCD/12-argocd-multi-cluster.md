# ArgoCD Multi-Cluster Management

## Overview

ArgoCD can manage applications across multiple Kubernetes clusters from a single control plane.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MULTI-CLUSTER ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                     ┌─────────────────────┐                        │
│                     │    ArgoCD Server    │                        │
│                     │   (Control Plane)   │                        │
│                     └──────────┬──────────┘                        │
│                                │                                    │
│              ┌─────────────────┼─────────────────┐                 │
│              │                 │                 │                 │
│              ▼                 ▼                 ▼                 │
│     ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│     │  Dev Cluster│   │ Stage Cluster│   │ Prod Cluster│          │
│     │             │   │              │   │             │          │
│     │ namespace:  │   │ namespace:   │   │ namespace:  │          │
│     │  - dev-app  │   │  - stage-app │   │  - prod-app │          │
│     └─────────────┘   └──────────────┘   └─────────────┘          │
│                                                                     │
│  Benefits:                                                          │
│  • Single dashboard for all clusters                               │
│  • Consistent deployment across environments                       │
│  • Centralized RBAC and policies                                   │
│  • Simplified operations                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Adding Clusters

### Using ArgoCD CLI

```bash
# List existing contexts
kubectl config get-contexts

# Add cluster from kubeconfig context
argocd cluster add my-production-context

# Add with custom name
argocd cluster add my-production-context --name production

# Add with specific namespace
argocd cluster add my-production-context --namespace argocd
```

### What Happens When Adding a Cluster

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CLUSTER REGISTRATION PROCESS                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. ArgoCD creates a ServiceAccount in target cluster              │
│  2. Creates ClusterRole with required permissions                  │
│  3. Creates ClusterRoleBinding                                     │
│  4. Stores cluster credentials as Secret in ArgoCD namespace       │
│                                                                     │
│  Target Cluster:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ ServiceAccount: argocd-manager                              │   │
│  │ ClusterRole: argocd-manager-role                            │   │
│  │ ClusterRoleBinding: argocd-manager-role-binding             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ArgoCD Cluster:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Secret: cluster-<server-url>                                │   │
│  │ Labels: argocd.argoproj.io/secret-type: cluster             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Declarative Cluster Registration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: production-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  name: production
  server: https://production.example.com:6443
  config: |
    {
      "bearerToken": "eyJhbGciOiJSUzI1NiIs...",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0..."
      }
    }
```

### Cluster with Service Account Token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: staging-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  name: staging
  server: https://staging.example.com:6443
  config: |
    {
      "bearerToken": "eyJhbGciOiJSUzI1NiIs...",
      "tlsClientConfig": {
        "caData": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t..."
      }
    }
```

### Cluster with Client Certificate

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dev-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  name: development
  server: https://dev.example.com:6443
  config: |
    {
      "tlsClientConfig": {
        "caData": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t...",
        "certData": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t...",
        "keyData": "LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVkt..."
      }
    }
```

---

## Managing Clusters

### List Clusters

```bash
# List all registered clusters
argocd cluster list

# Output:
# SERVER                          NAME        VERSION  STATUS
# https://kubernetes.default.svc  in-cluster  1.28     Successful
# https://prod.example.com        production  1.28     Successful
# https://stage.example.com       staging     1.27     Successful
```

### Get Cluster Details

```bash
argocd cluster get https://prod.example.com

# Or by name
argocd cluster get production
```

### Remove Cluster

```bash
argocd cluster rm https://prod.example.com
```

### Rotate Cluster Credentials

```bash
# Generate new token and update secret
argocd cluster rotate-auth production
```

---

## Deploying to Multiple Clusters

### Single App to Single Cluster

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/apps.git
    path: my-app/overlays/production
  destination:
    server: https://production.example.com  # Target cluster
    namespace: my-app
```

### Same App to Multiple Clusters (Manual)

```yaml
# development.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
spec:
  destination:
    server: https://dev.example.com
    namespace: my-app
  source:
    path: overlays/development
---
# staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
spec:
  destination:
    server: https://staging.example.com
    namespace: my-app
  source:
    path: overlays/staging
---
# production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
spec:
  destination:
    server: https://production.example.com
    namespace: my-app
  source:
    path: overlays/production
```

### Using ApplicationSets (Recommended)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: development
            url: https://dev.example.com
            namespace: my-app-dev
          - cluster: staging
            url: https://staging.example.com
            namespace: my-app-staging
          - cluster: production
            url: https://production.example.com
            namespace: my-app-prod

  template:
    metadata:
      name: 'my-app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/apps.git
        targetRevision: HEAD
        path: 'my-app/overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Cluster Patterns

### Hub and Spoke

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HUB AND SPOKE PATTERN                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                      ┌─────────────────┐                           │
│                      │   Hub Cluster   │                           │
│                      │    (ArgoCD)     │                           │
│                      └────────┬────────┘                           │
│                               │                                     │
│          ┌────────────────────┼────────────────────┐               │
│          │                    │                    │               │
│          ▼                    ▼                    ▼               │
│   ┌────────────┐      ┌────────────┐      ┌────────────┐         │
│   │   Spoke    │      │   Spoke    │      │   Spoke    │         │
│   │  Cluster 1 │      │  Cluster 2 │      │  Cluster 3 │         │
│   │   (Dev)    │      │  (Stage)   │      │   (Prod)   │         │
│   └────────────┘      └────────────┘      └────────────┘         │
│                                                                     │
│  Characteristics:                                                   │
│  • Single ArgoCD instance manages all clusters                     │
│  • Centralized control and visibility                              │
│  • Network connectivity required from hub to all spokes            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Standalone per Cluster

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STANDALONE PATTERN                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐      │
│   │   Dev Cluster  │  │  Stage Cluster │  │  Prod Cluster  │      │
│   │  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │      │
│   │  │  ArgoCD  │  │  │  │  ArgoCD  │  │  │  │  ArgoCD  │  │      │
│   │  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │      │
│   │                │  │                │  │                │      │
│   │  Apps: dev     │  │  Apps: stage   │  │  Apps: prod    │      │
│   └────────────────┘  └────────────────┘  └────────────────┘      │
│                                                                     │
│  Characteristics:                                                   │
│  • Each cluster has its own ArgoCD                                 │
│  • Complete isolation                                              │
│  • No cross-cluster network requirements                           │
│  • Multiple dashboards to manage                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ArgoCD Managing Itself

```yaml
# App-of-apps pattern for ArgoCD on each cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/cluster-configs.git
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
```

---

## Project Configuration for Multi-Cluster

### Restrict Destinations per Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
  namespace: argocd
spec:
  description: Frontend team - dev and staging only

  destinations:
    # Allow dev cluster
    - namespace: frontend-*
      server: https://dev.example.com

    # Allow staging cluster
    - namespace: frontend-*
      server: https://staging.example.com

    # Production NOT listed = not allowed

  sourceRepos:
    - https://github.com/myorg/frontend-*
```

### Production Project

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications only

  destinations:
    - namespace: '*'
      server: https://production.example.com

  sourceRepos:
    - https://github.com/myorg/production-configs.git

  # Require manual sync
  syncWindows:
    - kind: deny
      schedule: '* * * * *'
      duration: 24h
      manualSync: true

  roles:
    - name: production-deployer
      policies:
        - p, proj:production:production-deployer, applications, sync, production/*, allow
      groups:
        - production-team
```

---

## Network Considerations

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NETWORK REQUIREMENTS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ArgoCD needs to reach:                                            │
│  • Target cluster API server (port 6443 typically)                 │
│  • Git repositories                                                │
│  • Helm repositories                                               │
│                                                                     │
│  Options for connectivity:                                          │
│                                                                     │
│  1. Direct connectivity (public/VPN)                               │
│     ArgoCD ──────────────────────────▶ Target Cluster              │
│                                                                     │
│  2. Through bastion/jump host                                      │
│     ArgoCD ──▶ Bastion ──▶ Target Cluster                         │
│                                                                     │
│  3. Using cluster gateway                                          │
│     ArgoCD ──▶ Gateway Agent ──▶ Target Cluster                   │
│     (Cluster agent initiates connection outbound)                  │
│                                                                     │
│  4. Service mesh                                                   │
│     ArgoCD ──▶ Istio/Linkerd ──▶ Target Cluster                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Cluster Labels and Annotations

### Add Metadata to Clusters

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: production-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    # Custom labels for filtering
    environment: production
    region: us-east-1
    tier: critical
  annotations:
    team: platform
    cost-center: "12345"
stringData:
  name: production
  server: https://production.example.com
  config: |
    {...}
```

### Use Labels in ApplicationSets

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monitoring
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
        # Or match expressions
        # matchExpressions:
        #   - key: tier
        #     operator: In
        #     values: [critical, standard]

  template:
    metadata:
      name: 'monitoring-{{name}}'
    spec:
      source:
        repoURL: https://github.com/myorg/monitoring.git
        path: overlays/production
      destination:
        server: '{{server}}'
        namespace: monitoring
```

---

## Troubleshooting

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TROUBLESHOOTING                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Issue: Cluster shows "Unknown" status                             │
│  ─────────────────────────────────────                              │
│  • Check network connectivity to cluster                           │
│  • Verify credentials haven't expired                              │
│  • Check argocd-application-controller logs                        │
│                                                                     │
│  Issue: Permission denied when deploying                           │
│  ────────────────────────────────────────                           │
│  • Verify ServiceAccount has required permissions                  │
│  • Check ClusterRole bindings on target cluster                    │
│  • Review AppProject destination restrictions                      │
│                                                                     │
│  Issue: Connection timeout                                          │
│  ────────────────────────────                                       │
│  • Check firewall rules                                            │
│  • Verify cluster API endpoint is accessible                       │
│  • Try: curl -k https://cluster-api:6443/healthz                   │
│                                                                     │
│  Issue: Certificate errors                                          │
│  ────────────────────────                                           │
│  • Verify CA certificate in cluster secret                         │
│  • Check certificate hasn't expired                                │
│  • Option: Set insecure: true (not for production)                 │
│                                                                     │
│  Debug commands:                                                    │
│  kubectl logs -n argocd deployment/argocd-application-controller   │
│  argocd cluster get <cluster-url> --refresh                        │
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
│  Multi-Cluster Capabilities:                                        │
│  • Register multiple clusters via CLI or declaratively             │
│  • Deploy same app to multiple clusters                            │
│  • Use ApplicationSets for automation                              │
│  • Project-level destination restrictions                          │
│                                                                     │
│  Patterns:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Hub-and-Spoke │ Central ArgoCD manages all clusters         │   │
│  │ Standalone    │ ArgoCD per cluster, isolated                │   │
│  │ Hybrid        │ Central + local for specific needs          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Best Practices:                                                    │
│  • Use least-privilege ServiceAccount on target clusters           │
│  • Restrict cluster access via Projects                            │
│  • Use cluster labels for organization                             │
│  • Monitor cluster connectivity                                    │
│  • Rotate credentials regularly                                    │
│                                                                     │
│  Next: Learn ApplicationSets for advanced multi-app patterns       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
