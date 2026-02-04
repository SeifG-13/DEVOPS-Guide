# Kubernetes DaemonSets

## What is a DaemonSet?

A DaemonSet ensures that all (or selected) nodes run a copy of a pod.

```
┌─────────────────────────────────────────────────────────────────┐
│                     DaemonSet Concept                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   DaemonSet: fluentd-logging                                    │
│                                                                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│   │   Node 1    │  │   Node 2    │  │   Node 3    │            │
│   │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │            │
│   │  │fluentd│  │  │  │fluentd│  │  │  │fluentd│  │            │
│   │  └───────┘  │  │  └───────┘  │  │  └───────┘  │            │
│   │  Other Pods │  │  Other Pods │  │  Other Pods │            │
│   └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                  │
│   When new node added:                                          │
│   ┌─────────────┐                                               │
│   │   Node 4    │  ← DaemonSet automatically deploys fluentd   │
│   │  ┌───────┐  │                                               │
│   │  │fluentd│  │                                               │
│   │  └───────┘  │                                               │
│   └─────────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Common Use Cases

| Use Case | Description |
|----------|-------------|
| Logging | Collect logs from all nodes (fluentd, logstash) |
| Monitoring | Node metrics collection (node-exporter, datadog) |
| Networking | Network plugins (calico, weave, cilium) |
| Storage | Distributed storage (ceph, glusterfs) |
| Security | Security agents (falco, sysdig) |

## Creating a DaemonSet

### Basic DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.14
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
```

### Node Exporter DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.0
        ports:
        - containerPort: 9100
          hostPort: 9100
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

## Node Selection

### Run on Specific Nodes

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-monitor
spec:
  selector:
    matchLabels:
      name: gpu-monitor
  template:
    metadata:
      labels:
        name: gpu-monitor
    spec:
      nodeSelector:
        gpu: "true"  # Only nodes with gpu=true label
      containers:
      - name: gpu-monitor
        image: nvidia/gpu-monitor:latest
```

### Using Node Affinity

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      name: ssd-monitor
  template:
    metadata:
      labels:
        name: ssd-monitor
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disk-type
                operator: In
                values:
                - ssd
                - nvme
      containers:
      - name: ssd-monitor
        image: ssd-monitor:latest
```

### Tolerating Taints

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: network-plugin
spec:
  selector:
    matchLabels:
      name: network-plugin
  template:
    metadata:
      labels:
        name: network-plugin
    spec:
      tolerations:
      # Run on control plane nodes
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      # Run on any tainted node
      - operator: Exists
      containers:
      - name: network-plugin
        image: calico/node:latest
```

## Update Strategies

### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # or percentage "25%"
```

### OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete  # Pods updated only when manually deleted
```

```bash
# With OnDelete, manually delete pods to update
kubectl delete pod fluentd-xxxxx

# New pod will use updated spec
```

## DaemonSet with Init Containers

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      name: log-collector
  template:
    metadata:
      labels:
        name: log-collector
    spec:
      initContainers:
      - name: setup
        image: busybox
        command: ['sh', '-c', 'mkdir -p /var/log/app && chmod 755 /var/log/app']
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      containers:
      - name: collector
        image: log-collector:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

## Priority and Preemption

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: critical-agent
spec:
  selector:
    matchLabels:
      name: critical-agent
  template:
    metadata:
      labels:
        name: critical-agent
    spec:
      priorityClassName: system-node-critical
      containers:
      - name: agent
        image: critical-agent:latest
```

## Managing DaemonSets

```bash
# Create DaemonSet
kubectl apply -f daemonset.yaml

# List DaemonSets
kubectl get daemonsets
kubectl get ds

# Get details
kubectl describe ds fluentd

# Check status
kubectl get ds fluentd -o wide

# View pods created by DaemonSet
kubectl get pods -l name=fluentd -o wide

# Update DaemonSet
kubectl set image ds/fluentd fluentd=fluentd:v1.15

# Check rollout status
kubectl rollout status ds/fluentd

# Rollback
kubectl rollout undo ds/fluentd

# Delete DaemonSet
kubectl delete ds fluentd
```

## DaemonSet Status

```bash
kubectl get ds fluentd

# Output:
# NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
# fluentd   3         3         3       3            3           <none>          5m
```

| Field | Description |
|-------|-------------|
| DESIRED | Number of nodes that should run the pod |
| CURRENT | Number of nodes running at least one pod |
| READY | Number of pods ready |
| UP-TO-DATE | Pods with latest template |
| AVAILABLE | Pods available (ready for minReadySeconds) |

## Practical Examples

### Log Collection

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: elastic/filebeat:8.10.0
        args:
        - -c
        - /etc/filebeat/filebeat.yml
        - -e
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: filebeat-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
```

### Network Plugin (Calico)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
      containers:
      - name: calico-node
        image: calico/node:v3.26.0
        env:
        - name: DATASTORE_TYPE
          value: "kubernetes"
        - name: CALICO_NETWORKING_BACKEND
          value: "bird"
        securityContext:
          privileged: true
        volumeMounts:
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: var-run-calico
          mountPath: /var/run/calico
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: var-run-calico
        hostPath:
          path: /var/run/calico
```

## Quick Reference

### DaemonSet vs Other Controllers

| Controller | Use Case |
|-----------|----------|
| DaemonSet | One pod per node |
| Deployment | Stateless applications |
| StatefulSet | Stateful applications |
| ReplicaSet | Specific number of replicas |

### Key Fields

| Field | Description |
|-------|-------------|
| `nodeSelector` | Select specific nodes |
| `tolerations` | Tolerate node taints |
| `updateStrategy` | RollingUpdate or OnDelete |
| `hostNetwork` | Use host networking |
| `hostPID` | Access host processes |

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl get ds` | List DaemonSets |
| `kubectl describe ds` | Show details |
| `kubectl rollout status ds` | Check update status |
| `kubectl rollout undo ds` | Rollback |
| `kubectl set image ds` | Update image |

---

**Previous:** [13-statefulsets.md](13-statefulsets.md) | **Next:** [15-jobs-cronjobs.md](15-jobs-cronjobs.md)
