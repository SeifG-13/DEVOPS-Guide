# Kubernetes Deployments

## What is a Deployment?

A Deployment provides declarative updates for Pods and ReplicaSets.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Deployment Hierarchy                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Deployment (nginx-deployment)                                 │
│   │                                                              │
│   └── ReplicaSet (nginx-deployment-abc123)                      │
│       │                                                          │
│       ├── Pod (nginx-deployment-abc123-xyz1)                    │
│       ├── Pod (nginx-deployment-abc123-xyz2)                    │
│       └── Pod (nginx-deployment-abc123-xyz3)                    │
│                                                                  │
│   Features:                                                     │
│   • Rolling updates                                             │
│   • Rollback to previous versions                               │
│   • Scaling                                                     │
│   • Pause and resume updates                                    │
│   • Revision history                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Creating a Deployment

### Basic Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        ports:
        - containerPort: 80
```

```bash
# Create deployment
kubectl apply -f deployment.yaml

# Create imperatively
kubectl create deployment nginx --image=nginx --replicas=3

# Generate YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

### Complete Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
  annotations:
    kubernetes.io/change-cause: "Initial deployment"
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: myapp:v1
        ports:
        - containerPort: 8080
        env:
        - name: ENV
          value: "production"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```

## Update Strategies

### RollingUpdate (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Max pods above desired
      maxUnavailable: 25%  # Max pods unavailable
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Rolling Update Process                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   replicas: 3, maxSurge: 1, maxUnavailable: 0                   │
│                                                                  │
│   Step 1: Create new pod (4 total)                              │
│   [v1] [v1] [v1] [v2-new]                                       │
│                                                                  │
│   Step 2: New pod ready, terminate old (3 total)                │
│   [v1] [v1] [v2] [terminating]                                  │
│                                                                  │
│   Step 3: Create new pod (4 total)                              │
│   [v1] [v1] [v2] [v2-new]                                       │
│                                                                  │
│   Step 4: Continue until all updated                            │
│   [v2] [v2] [v2]                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Recreate Strategy

```yaml
spec:
  strategy:
    type: Recreate  # All old pods terminated before new ones created
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Recreate Strategy                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Step 1: Initial state                                         │
│   [v1] [v1] [v1]                                                │
│                                                                  │
│   Step 2: All pods terminated                                   │
│   [ ] [ ] [ ]  ← Downtime here!                                 │
│                                                                  │
│   Step 3: New pods created                                      │
│   [v2] [v2] [v2]                                                │
│                                                                  │
│   Use when:                                                     │
│   • Application can't run multiple versions simultaneously      │
│   • Short downtime is acceptable                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Updating Deployments

### Update Image

```bash
# Update image
kubectl set image deployment/nginx nginx=nginx:1.26

# Update using edit
kubectl edit deployment nginx

# Update using patch
kubectl patch deployment nginx -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.26"}]}}}}'

# Update via apply (modify YAML first)
kubectl apply -f deployment.yaml
```

### Record Changes

```bash
# Record the change cause
kubectl set image deployment/nginx nginx=nginx:1.26 --record

# Or use annotation
kubectl annotate deployment nginx kubernetes.io/change-cause="Update to nginx 1.26"
```

### Monitor Update

```bash
# Watch rollout status
kubectl rollout status deployment/nginx

# Check rollout history
kubectl rollout history deployment/nginx

# View specific revision
kubectl rollout history deployment/nginx --revision=2
```

## Rollback

```bash
# Rollback to previous version
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# Check rollout history
kubectl rollout history deployment/nginx
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      Rollback Process                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Revision History:                                             │
│   REVISION  CHANGE-CAUSE                                        │
│   1         Initial deployment                                  │
│   2         Update to v1.1                                      │
│   3         Update to v1.2 (current) ← Problem!                 │
│                                                                  │
│   kubectl rollout undo deployment/nginx --to-revision=2         │
│                                                                  │
│   REVISION  CHANGE-CAUSE                                        │
│   1         Initial deployment                                  │
│   3         Update to v1.2                                      │
│   4         Rollback to v1.1 (current) ← Fixed!                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Scaling

```bash
# Scale deployment
kubectl scale deployment nginx --replicas=5

# Auto-scale based on CPU
kubectl autoscale deployment nginx --min=3 --max=10 --cpu-percent=80

# Check HPA
kubectl get hpa
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

## Pause and Resume

```bash
# Pause deployment (for multiple updates)
kubectl rollout pause deployment/nginx

# Make changes
kubectl set image deployment/nginx nginx=nginx:1.26
kubectl set resources deployment/nginx -c=nginx --limits=cpu=200m,memory=512Mi

# Resume deployment (applies all changes at once)
kubectl rollout resume deployment/nginx
```

## Deployment Management

### View Deployments

```bash
# List deployments
kubectl get deployments
kubectl get deploy

# Detailed view
kubectl get deploy -o wide

# Describe deployment
kubectl describe deployment nginx

# Get YAML
kubectl get deployment nginx -o yaml
```

### Delete Deployments

```bash
# Delete deployment (also deletes ReplicaSets and Pods)
kubectl delete deployment nginx

# Delete from file
kubectl delete -f deployment.yaml

# Delete but keep pods
kubectl delete deployment nginx --cascade=orphan
```

## Advanced Configurations

### Min Ready Seconds

```yaml
spec:
  minReadySeconds: 10  # Pod must be ready for 10s before considered available
```

### Progress Deadline

```yaml
spec:
  progressDeadlineSeconds: 600  # Deployment fails if no progress in 10 minutes
```

### Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  # Or: maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
```

### Deployment with InitContainers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      initContainers:
      - name: init-db
        image: busybox
        command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
      containers:
      - name: webapp
        image: webapp:v1
```

## Blue-Green Deployment Pattern

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1
---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# Service (switch between blue/green)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Change to 'green' to switch
  ports:
  - port: 80
```

## Canary Deployment Pattern

```yaml
# Stable deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: app
        image: myapp:v1
---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# Service routes to both
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # Matches both stable and canary
  ports:
  - port: 80
```

## Quick Reference

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl create deployment` | Create deployment |
| `kubectl get deploy` | List deployments |
| `kubectl describe deploy` | Show details |
| `kubectl set image` | Update image |
| `kubectl scale deploy` | Scale replicas |
| `kubectl rollout status` | Check update status |
| `kubectl rollout history` | View history |
| `kubectl rollout undo` | Rollback |
| `kubectl rollout pause/resume` | Pause/resume updates |

### Strategy Comparison

| Strategy | Downtime | Use Case |
|----------|----------|----------|
| RollingUpdate | No | Default, gradual update |
| Recreate | Yes | When versions can't coexist |

---

**Previous:** [06-replicasets.md](06-replicasets.md) | **Next:** [08-services.md](08-services.md)
