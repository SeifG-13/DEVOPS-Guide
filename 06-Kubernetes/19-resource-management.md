# Kubernetes Resource Management

## Resource Requests and Limits

```
┌─────────────────────────────────────────────────────────────────┐
│               Requests vs Limits                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Requests: Minimum guaranteed resources                        │
│   Limits: Maximum allowed resources                             │
│                                                                  │
│   Memory Usage                                                  │
│   ─────────────────────────────────────────────────────         │
│   0        request (256Mi)            limit (512Mi)    max      │
│   │            │                          │             │       │
│   ▼            ▼                          ▼             ▼       │
│   ├────────────┼──────────────────────────┼─────────────┤       │
│   │ guaranteed │     burstable            │   OOMKilled │       │
│   │            │                          │             │       │
│   ├────────────┴──────────────────────────┴─────────────┤       │
│                                                                  │
│   • Requests: Used for scheduling (will pod fit on node?)       │
│   • Limits: Enforced at runtime                                 │
│   • CPU: Throttled when limit reached                           │
│   • Memory: OOMKilled when limit reached                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Setting Resources

### Container Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

### CPU Units

| Value | Description |
|-------|-------------|
| `1` | 1 CPU core |
| `500m` | 0.5 CPU (500 millicores) |
| `100m` | 0.1 CPU (100 millicores) |
| `0.1` | Same as 100m |

### Memory Units

| Value | Description |
|-------|-------------|
| `128Mi` | 128 mebibytes |
| `1Gi` | 1 gibibyte |
| `128M` | 128 megabytes (decimal) |
| `1G` | 1 gigabyte (decimal) |

## QoS Classes

```
┌─────────────────────────────────────────────────────────────────┐
│                    QoS Classes                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Guaranteed (highest priority)                                 │
│   • requests == limits (for all containers)                     │
│   • Memory and CPU both set                                     │
│                                                                  │
│   Burstable (medium priority)                                   │
│   • requests < limits (at least one container)                  │
│   • Some resource requests set                                  │
│                                                                  │
│   BestEffort (lowest priority)                                  │
│   • No requests or limits set                                   │
│   • First to be evicted under pressure                         │
│                                                                  │
│   Eviction priority (under memory pressure):                    │
│   BestEffort → Burstable → Guaranteed                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Guaranteed QoS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"  # Same as request
        cpu: "500m"      # Same as request
```

### Burstable QoS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"  # Higher than request
        cpu: "500m"
```

### Check QoS Class

```bash
kubectl get pod mypod -o jsonpath='{.status.qosClass}'
```

## LimitRange

Set default and limits for namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limit
  namespace: development
spec:
  limits:
  # Container limits
  - type: Container
    default:          # Default limits if not specified
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # Default requests if not specified
      cpu: "100m"
      memory: "128Mi"
    max:             # Maximum allowed
      cpu: "2"
      memory: "2Gi"
    min:             # Minimum required
      cpu: "50m"
      memory: "64Mi"
    maxLimitRequestRatio:  # Max limit/request ratio
      cpu: "4"

  # Pod limits
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"

  # PVC limits
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 100Gi
```

```bash
# View LimitRange
kubectl get limitrange
kubectl describe limitrange cpu-mem-limit
```

## ResourceQuota

Limit total resources in namespace.

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
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"

    # Object counts
    pods: "50"
    services: "20"
    configmaps: "50"
    secrets: "50"
    persistentvolumeclaims: "20"
    replicationcontrollers: "20"
    services.loadbalancers: "5"
    services.nodeports: "10"

    # Storage
    requests.storage: "500Gi"
    persistentvolumeclaims: "10"

    # Extended resources
    requests.nvidia.com/gpu: "4"
```

### Quota Scopes

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
spec:
  hard:
    pods: "10"
    cpu: "20"
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values: ["high"]
```

### View Quota Usage

```bash
kubectl get resourcequota
kubectl describe resourcequota compute-quota

# Output shows used/hard limits
# Name:           compute-quota
# Resource        Used    Hard
# --------        ----    ----
# limits.cpu      4       20
# limits.memory   8Gi     40Gi
# pods            5       50
```

## Priority Classes

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority for critical workloads"
preemptionPolicy: PreemptLowerPriority
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: false
description: "Low priority for batch jobs"
preemptionPolicy: Never
```

### Use PriorityClass

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: myapp:latest
```

## Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

```bash
# Create HPA imperatively
kubectl autoscale deployment webapp --min=2 --max=10 --cpu-percent=70

# View HPA
kubectl get hpa
kubectl describe hpa webapp-hpa
```

## Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Initial, Off
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
```

## Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
spec:
  minAvailable: 2    # Minimum pods that must be available
  # or: maxUnavailable: 1
  selector:
    matchLabels:
      app: webapp
```

```bash
kubectl get pdb
kubectl describe pdb webapp-pdb
```

## Node Resources

```bash
# View node capacity and allocatable
kubectl describe node <node-name>

# Output includes:
# Capacity:
#   cpu:                4
#   memory:             8Gi
#   pods:               110
# Allocatable:
#   cpu:                3800m
#   memory:             7Gi
#   pods:               110

# View resource usage
kubectl top nodes
kubectl top pods
```

## Complete Example

```yaml
# Namespace with quotas and limits
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: "100Gi"
    limits.cpu: "100"
    limits.memory: "200Gi"
    pods: "100"
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
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "256Mi"
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
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
      containers:
      - name: webapp
        image: webapp:v1
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## Quick Reference

### Resource Units

| Resource | Units |
|----------|-------|
| CPU | `m` (millicores), cores |
| Memory | `Ki`, `Mi`, `Gi`, `Ti` |

### QoS Classes

| Class | Condition |
|-------|-----------|
| Guaranteed | requests == limits |
| Burstable | requests < limits |
| BestEffort | No resources set |

### Commands

| Command | Description |
|---------|-------------|
| `kubectl top nodes` | Node resource usage |
| `kubectl top pods` | Pod resource usage |
| `kubectl describe node` | Node capacity |
| `kubectl get resourcequota` | View quotas |
| `kubectl get limitrange` | View limits |
| `kubectl get hpa` | View autoscalers |

---

**Previous:** [18-rbac.md](18-rbac.md) | **Next:** [20-health-checks.md](20-health-checks.md)
