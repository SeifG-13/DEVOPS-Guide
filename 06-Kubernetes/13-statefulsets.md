# Kubernetes StatefulSets

## What is a StatefulSet?

StatefulSet manages stateful applications with unique, persistent identities.

```
┌─────────────────────────────────────────────────────────────────┐
│              StatefulSet vs Deployment                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Deployment                       StatefulSet                  │
│   ┌─────────────────────────┐     ┌─────────────────────────┐  │
│   │ nginx-abc123-xyz1       │     │ mysql-0                  │  │
│   │ nginx-abc123-xyz2       │     │ mysql-1                  │  │
│   │ nginx-abc123-xyz3       │     │ mysql-2                  │  │
│   └─────────────────────────┘     └─────────────────────────┘  │
│   • Random pod names              • Ordered, stable names       │
│   • Interchangeable               • Unique identity            │
│   • Shared storage (optional)     • Own persistent storage     │
│   • Parallel scaling              • Sequential scaling         │
│                                                                  │
│   Use Deployment for:             Use StatefulSet for:         │
│   • Stateless apps                • Databases                  │
│   • Web servers                   • Message queues             │
│   • API servers                   • Distributed systems        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## StatefulSet Guarantees

| Feature | Description |
|---------|-------------|
| Stable Network Identity | Predictable pod names (app-0, app-1, app-2) |
| Stable Storage | Each pod gets its own PVC |
| Ordered Deployment | Pods created in order (0, 1, 2) |
| Ordered Scaling | Scale up/down in order |
| Ordered Updates | Rolling update in reverse order |

## Creating a StatefulSet

### Basic StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web-headless"  # Required headless service
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
---
# Required headless service
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: web
  ports:
  - port: 80
```

### Complete StatefulSet Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast
      resources:
        requests:
          storage: 10Gi
```

## Pod Identity

```
┌─────────────────────────────────────────────────────────────────┐
│                   StatefulSet Pod Identity                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   StatefulSet: mysql (replicas: 3)                              │
│                                                                  │
│   Pod Name       Hostname    Stable Network ID                  │
│   ─────────────────────────────────────────────────────         │
│   mysql-0        mysql-0     mysql-0.mysql-headless.default     │
│   mysql-1        mysql-1     mysql-1.mysql-headless.default     │
│   mysql-2        mysql-2     mysql-2.mysql-headless.default     │
│                                                                  │
│   DNS Format:                                                   │
│   <pod-name>.<headless-service>.<namespace>.svc.cluster.local   │
│                                                                  │
│   PVC Names:                                                    │
│   data-mysql-0                                                  │
│   data-mysql-1                                                  │
│   data-mysql-2                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Accessing Pods

```bash
# Access specific pod
mysql -h mysql-0.mysql-headless.default.svc.cluster.local

# From same namespace
mysql -h mysql-0.mysql-headless

# Access any pod via regular service
mysql -h mysql-read
```

## Scaling

### Ordered Scaling

```
Scale Up (0 → 3):
mysql-0 created → Ready
mysql-1 created → Ready  (after mysql-0 is Ready)
mysql-2 created → Ready  (after mysql-1 is Ready)

Scale Down (3 → 1):
mysql-2 terminated
mysql-1 terminated (after mysql-2 is terminated)
```

```bash
# Scale StatefulSet
kubectl scale statefulset mysql --replicas=5

# Watch scaling
kubectl get pods -w -l app=mysql
```

### Pod Management Policy

```yaml
spec:
  podManagementPolicy: Parallel  # or OrderedReady (default)
```

| Policy | Behavior |
|--------|----------|
| OrderedReady | Sequential creation/deletion (default) |
| Parallel | All pods created/deleted simultaneously |

## Update Strategies

### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods
```

Updates in reverse order: mysql-2 → mysql-1 → mysql-0

### Partition-Based Updates

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # Only update pods >= 2
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  Partition-Based Updates                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   partition: 2                                                  │
│                                                                  │
│   mysql-0  [v1] ─ Not updated                                   │
│   mysql-1  [v1] ─ Not updated                                   │
│   mysql-2  [v2] ─ Updated (ordinal >= partition)                │
│   mysql-3  [v2] ─ Updated                                       │
│   mysql-4  [v2] ─ Updated                                       │
│                                                                  │
│   Use for canary deployments:                                   │
│   1. Set partition=4 (update only mysql-4)                      │
│   2. Test mysql-4                                               │
│   3. Lower partition to update more pods                        │
│   4. Set partition=0 to update all                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### OnDelete Strategy

```yaml
spec:
  updateStrategy:
    type: OnDelete  # Update only when pod is manually deleted
```

## Persistent Storage

### Volume Claim Templates

```yaml
volumeClaimTemplates:
- metadata:
    name: data
    labels:
      app: mysql
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 100Gi
```

### PVC Behavior

```bash
# PVCs are NOT deleted when StatefulSet is deleted
kubectl delete statefulset mysql

# PVCs remain:
kubectl get pvc
# data-mysql-0   Bound   pv-xxx   100Gi
# data-mysql-1   Bound   pv-xxx   100Gi
# data-mysql-2   Bound   pv-xxx   100Gi

# Manual PVC deletion required
kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

## StatefulSet for Databases

### PostgreSQL Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### Redis Cluster Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 6
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        command: ["redis-server"]
        args:
        - "--cluster-enabled"
        - "yes"
        - "--cluster-config-file"
        - "/data/nodes.conf"
        ports:
        - containerPort: 6379
        - containerPort: 16379
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

## Managing StatefulSets

```bash
# Create StatefulSet
kubectl apply -f statefulset.yaml

# List StatefulSets
kubectl get statefulsets
kubectl get sts

# Describe StatefulSet
kubectl describe sts mysql

# Get pods with ordinal
kubectl get pods -l app=mysql

# Scale
kubectl scale sts mysql --replicas=5

# Delete StatefulSet (pods deleted, PVCs retained)
kubectl delete sts mysql

# Delete StatefulSet without deleting pods
kubectl delete sts mysql --cascade=orphan

# Force delete stuck pod
kubectl delete pod mysql-0 --force --grace-period=0
```

## Quick Reference

### StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random | Ordered (app-0, app-1) |
| Scaling | Parallel | Sequential |
| Storage | Shared/None | Unique per pod |
| Network | Random IP | Stable hostname |
| Use case | Stateless | Stateful |

### Key Fields

| Field | Description |
|-------|-------------|
| `serviceName` | Required headless service |
| `replicas` | Number of pods |
| `volumeClaimTemplates` | PVC template for each pod |
| `podManagementPolicy` | OrderedReady or Parallel |
| `updateStrategy` | RollingUpdate or OnDelete |

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl get sts` | List StatefulSets |
| `kubectl describe sts` | Show details |
| `kubectl scale sts` | Scale replicas |
| `kubectl rollout status sts` | Check rollout |
| `kubectl rollout undo sts` | Rollback |

---

**Previous:** [12-volumes.md](12-volumes.md) | **Next:** [14-daemonsets.md](14-daemonsets.md)
