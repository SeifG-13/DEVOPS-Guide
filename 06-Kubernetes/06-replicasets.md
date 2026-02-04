# Kubernetes ReplicaSets

## What is a ReplicaSet?

A ReplicaSet ensures a specified number of pod replicas are running at all times.

```
┌─────────────────────────────────────────────────────────────────┐
│                     ReplicaSet Concept                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ReplicaSet: nginx-rs (replicas: 3)                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   ┌─────────┐   ┌─────────┐   ┌─────────┐              │   │
│   │   │  Pod 1  │   │  Pod 2  │   │  Pod 3  │              │   │
│   │   │  nginx  │   │  nginx  │   │  nginx  │              │   │
│   │   └─────────┘   └─────────┘   └─────────┘              │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   If a pod fails:                                               │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐                      │
│   │  Pod 1  │   │  Pod 2  │   │   New   │ ◄── Automatically    │
│   │  nginx  │   │  nginx  │   │  Pod 3  │     created          │
│   └─────────┘   └─────────┘   └─────────┘                      │
│       ✓             ✓           ✓ (replaced)                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## ReplicaSet vs ReplicationController

ReplicaSet is the next-generation ReplicationController with enhanced selector support.

```
┌─────────────────────────────────────────────────────────────────┐
│           ReplicaSet vs ReplicationController                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Feature              ReplicationController    ReplicaSet      │
│   ─────────────────────────────────────────────────────────     │
│   API Version          v1                       apps/v1         │
│   Selector Type        Equality-based only      Set-based       │
│   Selector Operators   = , !=                   In, NotIn,      │
│                                                 Exists, !Exists │
│   Rolling Updates      No (manual)              Via Deployment  │
│   Status               Legacy                   Current         │
│   Recommended          No                       Yes             │
│                                                                  │
│   ReplicationController Example:                                │
│   selector:                                                     │
│     app: nginx        # Only equality (=)                       │
│                                                                  │
│   ReplicaSet Example:                                           │
│   selector:                                                     │
│     matchLabels:                                                │
│       app: nginx                                                │
│     matchExpressions:                                           │
│       - key: env                                                │
│         operator: In                                            │
│         values: [prod, staging]                                 │
│                                                                  │
│   ✓ Always use ReplicaSet (via Deployment) instead of          │
│     ReplicationController                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ReplicationController Example (Legacy)

```yaml
# Deprecated - use ReplicaSet instead
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: nginx          # Equality-based only
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

### ReplicaSet Example (Current)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
    matchExpressions:        # Set-based selectors
    - key: environment
      operator: In
      values:
      - production
      - staging
  template:
    metadata:
      labels:
        app: nginx
        environment: production
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

## Creating a ReplicaSet

### Basic ReplicaSet

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
# Create ReplicaSet
kubectl apply -f replicaset.yaml

# Check status
kubectl get replicaset
kubectl get rs

# Describe
kubectl describe rs nginx-replicaset
```

### ReplicaSet with Set-Based Selectors

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: app-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
      - staging
    - key: tier
      operator: NotIn
      values:
      - backend
  template:
    metadata:
      labels:
        app: myapp
        environment: production
        tier: frontend
    spec:
      containers:
      - name: app
        image: myapp:v1
```

### Selector Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `In` | Label value is in set | `key: env, values: [prod, staging]` |
| `NotIn` | Label value not in set | `key: env, values: [test]` |
| `Exists` | Label key exists | `key: env` |
| `DoesNotExist` | Label key doesn't exist | `key: temp` |

## Managing ReplicaSets

### Scaling

```bash
# Scale imperatively
kubectl scale rs nginx-replicaset --replicas=5

# Scale via edit
kubectl edit rs nginx-replicaset

# Scale via patch
kubectl patch rs nginx-replicaset -p '{"spec":{"replicas":5}}'
```

### Viewing ReplicaSets

```bash
# List ReplicaSets
kubectl get rs
kubectl get replicasets

# With more details
kubectl get rs -o wide

# Show labels
kubectl get rs --show-labels

# Describe
kubectl describe rs nginx-replicaset

# Get YAML
kubectl get rs nginx-replicaset -o yaml
```

### Deleting ReplicaSets

```bash
# Delete ReplicaSet (also deletes pods)
kubectl delete rs nginx-replicaset

# Delete ReplicaSet but keep pods
kubectl delete rs nginx-replicaset --cascade=orphan

# Delete from file
kubectl delete -f replicaset.yaml
```

## ReplicaSet Behavior

### Pod Ownership

```bash
# Check pod's owner reference
kubectl get pod <pod-name> -o yaml | grep -A 5 ownerReferences

# Output shows:
# ownerReferences:
# - apiVersion: apps/v1
#   kind: ReplicaSet
#   name: nginx-replicaset
#   uid: ...
```

### What Happens When...

```
┌─────────────────────────────────────────────────────────────────┐
│                  ReplicaSet Behavior                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Scenario                         ReplicaSet Action            │
│   ─────────────────────────────────────────────────────         │
│   Pod crashes                      Creates new pod              │
│   Pod deleted                      Creates new pod              │
│   Node fails                       Recreates pods on other nodes│
│   Scale up (replicas: 3→5)         Creates 2 new pods          │
│   Scale down (replicas: 5→3)       Deletes 2 pods              │
│   Matching pod exists              Adopts the pod               │
│   Pod label changed                May orphan/adopt pods        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Label Matching

```bash
# If you manually create a pod with matching labels
kubectl run extra-nginx --image=nginx --labels="app=nginx"

# ReplicaSet will see 4 pods (3 its own + 1 manual)
# It will delete one to maintain replicas: 3
```

```yaml
# The selector must match template labels
spec:
  selector:
    matchLabels:
      app: nginx    # Must match
  template:
    metadata:
      labels:
        app: nginx  # Must match selector
```

## ReplicaSet vs Deployment

```
┌─────────────────────────────────────────────────────────────────┐
│                 ReplicaSet vs Deployment                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Deployment (recommended)                                      │
│   └── ReplicaSet (created by Deployment)                        │
│       └── Pods (created by ReplicaSet)                          │
│                                                                  │
│   ReplicaSet alone:                                             │
│   • No rolling updates                                          │
│   • No rollback capability                                      │
│   • No declarative updates                                      │
│   • Manual scaling only                                         │
│                                                                  │
│   Deployment adds:                                              │
│   • Rolling updates                                             │
│   • Rollback                                                    │
│   • Update strategies                                           │
│   • Revision history                                            │
│   • Pause/resume updates                                        │
│                                                                  │
│   ✓ Always use Deployments in production                       │
│   ✓ Deployments manage ReplicaSets for you                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use ReplicaSet Directly

- Rarely needed directly
- Custom update orchestration
- Understanding Kubernetes internals
- Special use cases where Deployment doesn't fit

## Practical Examples

### Web Application ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-rs
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      tier: frontend
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

### ReplicaSet with Probes

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v1
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Troubleshooting

```bash
# Check ReplicaSet status
kubectl get rs
kubectl describe rs <name>

# Check events
kubectl get events --sort-by='.lastTimestamp'

# Check pods created by ReplicaSet
kubectl get pods -l app=nginx

# Debug pod issues
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Common issues:
# - Image pull errors
# - Resource quota exceeded
# - Node selector not matching
# - Selector doesn't match template labels
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `kubectl get rs` | List ReplicaSets |
| `kubectl describe rs <name>` | Show details |
| `kubectl scale rs <name> --replicas=N` | Scale |
| `kubectl delete rs <name>` | Delete |
| `kubectl delete rs <name> --cascade=orphan` | Delete but keep pods |

### Key Points

- ReplicaSet maintains desired number of pod replicas
- Replaces deprecated ReplicationController
- Supports set-based selectors (In, NotIn, Exists, DoesNotExist)
- Usually managed by Deployments, not directly
- Selector must match template labels

---

**Previous:** [05-pods.md](05-pods.md) | **Next:** [07-deployments.md](07-deployments.md)
