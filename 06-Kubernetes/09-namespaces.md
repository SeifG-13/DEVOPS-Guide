# Kubernetes Namespaces

## What is a Namespace?

Namespaces provide a way to divide cluster resources between multiple users, teams, or projects.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Namespace Concept                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Kubernetes Cluster                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │  ┌──────────────────┐  ┌──────────────────┐            │   │
│   │  │   development    │  │    production    │            │   │
│   │  │  ┌────┐ ┌────┐  │  │  ┌────┐ ┌────┐  │            │   │
│   │  │  │Pod │ │Pod │  │  │  │Pod │ │Pod │  │            │   │
│   │  │  └────┘ └────┘  │  │  └────┘ └────┘  │            │   │
│   │  │  ┌────┐         │  │  ┌────┐ ┌────┐  │            │   │
│   │  │  │Svc │         │  │  │Svc │ │Svc │  │            │   │
│   │  │  └────┘         │  │  └────┘ └────┘  │            │   │
│   │  └──────────────────┘  └──────────────────┘            │   │
│   │                                                          │   │
│   │  ┌──────────────────┐  ┌──────────────────┐            │   │
│   │  │     staging      │  │    kube-system   │            │   │
│   │  │  ┌────┐ ┌────┐  │  │  (system pods)   │            │   │
│   │  │  │Pod │ │Pod │  │  │  CoreDNS, etc.   │            │   │
│   │  │  └────┘ └────┘  │  │                   │            │   │
│   │  └──────────────────┘  └──────────────────┘            │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Default Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Output:
# NAME              STATUS   AGE
# default           Active   1d
# kube-node-lease   Active   1d
# kube-public       Active   1d
# kube-system       Active   1d
```

| Namespace | Purpose |
|-----------|---------|
| `default` | Default namespace for resources |
| `kube-system` | Kubernetes system components |
| `kube-public` | Publicly accessible resources |
| `kube-node-lease` | Node heartbeat leases |

## Creating Namespaces

### Using kubectl

```bash
# Create namespace
kubectl create namespace development
kubectl create ns staging

# Create from YAML
kubectl apply -f namespace.yaml
```

### Using YAML

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    env: development
    team: backend
```

### With Resource Quota

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
```

## Working with Namespaces

### Specify Namespace

```bash
# Get resources in specific namespace
kubectl get pods -n development
kubectl get pods --namespace=development

# Get resources in all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Create resource in namespace
kubectl create deployment nginx --image=nginx -n development

# Apply with namespace in YAML
kubectl apply -f deployment.yaml -n development
```

### Set Default Namespace

```bash
# Set default namespace for current context
kubectl config set-context --current --namespace=development

# Verify
kubectl config view --minify | grep namespace

# Now commands use development namespace by default
kubectl get pods  # Shows pods in development
```

### Namespace in YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: development  # Specify namespace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

## Cross-Namespace Communication

```
┌─────────────────────────────────────────────────────────────────┐
│                Cross-Namespace Communication                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Namespace: frontend                Namespace: backend         │
│   ┌──────────────────────┐          ┌──────────────────────┐   │
│   │  ┌────────────────┐  │          │  ┌────────────────┐  │   │
│   │  │  Frontend Pod  │  │──────────│  │   API Service  │  │   │
│   │  │                │  │          │  │   (api-svc)    │  │   │
│   │  └────────────────┘  │          │  └────────────────┘  │   │
│   └──────────────────────┘          └──────────────────────┘   │
│                                                                  │
│   DNS: api-svc.backend.svc.cluster.local                        │
│         ──────  ───────                                         │
│         service namespace                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### DNS Names

```bash
# Full DNS format
<service-name>.<namespace>.svc.cluster.local

# Examples
api-svc.backend.svc.cluster.local     # Full name
api-svc.backend                       # Short form (same cluster)
api-svc                              # Same namespace only

# From frontend pod accessing backend service:
curl http://api-svc.backend:8080
```

## Resource Quotas

Limit resources per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    # Compute resources
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi

    # Object counts
    pods: "50"
    services: "20"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "10"

    # Storage
    requests.storage: 100Gi
```

```bash
# View quota
kubectl get resourcequota -n development
kubectl describe resourcequota compute-quota -n development
```

## Limit Ranges

Set default limits for containers.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: development
spec:
  limits:
  - type: Container
    default:          # Default limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # Default requests
      cpu: "100m"
      memory: "128Mi"
    max:              # Maximum allowed
      cpu: "2"
      memory: "2Gi"
    min:              # Minimum allowed
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

## Network Policies

Isolate network traffic by namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # Apply to all pods
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}  # Same namespace only
```

## RBAC per Namespace

```yaml
# Role in specific namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
---
# Bind role to user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

## Namespace Best Practices

### Naming Conventions

```bash
# By environment
development
staging
production

# By team
team-frontend
team-backend
team-data

# By project
project-alpha
project-beta

# Combined
project-alpha-prod
project-alpha-staging
```

### Multi-Environment Setup

```yaml
# Production namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: platform
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    pods: "200"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "1"
      memory: "1Gi"
    defaultRequest:
      cpu: "500m"
      memory: "512Mi"
```

## Managing Namespaces

### List and Describe

```bash
# List namespaces
kubectl get ns
kubectl get namespaces

# Describe namespace
kubectl describe namespace development

# Get namespace YAML
kubectl get ns development -o yaml
```

### Delete Namespace

```bash
# Delete namespace (deletes ALL resources in it!)
kubectl delete namespace development

# Force delete stuck namespace
kubectl get namespace development -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/development/finalize" -f -
```

### Label Namespaces

```bash
# Add label
kubectl label namespace development env=dev

# Remove label
kubectl label namespace development env-

# Select by label
kubectl get ns -l env=dev
```

## Namespace-Scoped vs Cluster-Scoped

| Namespace-Scoped | Cluster-Scoped |
|-----------------|----------------|
| Pods | Nodes |
| Services | Namespaces |
| Deployments | PersistentVolumes |
| ConfigMaps | ClusterRoles |
| Secrets | ClusterRoleBindings |
| Roles | StorageClasses |
| RoleBindings | CustomResourceDefinitions |

```bash
# List namespace-scoped resources
kubectl api-resources --namespaced=true

# List cluster-scoped resources
kubectl api-resources --namespaced=false
```

## Quick Reference

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl get ns` | List namespaces |
| `kubectl create ns <name>` | Create namespace |
| `kubectl delete ns <name>` | Delete namespace |
| `kubectl get pods -n <ns>` | Get pods in namespace |
| `kubectl get pods -A` | Get pods in all namespaces |
| `kubectl config set-context --current --namespace=<ns>` | Set default namespace |

### Key Points

- Namespaces provide logical isolation
- Resources within namespace have unique names
- Cross-namespace access via DNS: `service.namespace`
- Use ResourceQuotas to limit resources per namespace
- Use LimitRanges to set default container limits
- Delete namespace = delete all resources in it

---

**Previous:** [08-services.md](08-services.md) | **Next:** [10-configmaps.md](10-configmaps.md)
