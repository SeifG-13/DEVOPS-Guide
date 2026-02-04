# Kubernetes Pods

## What is a Pod?

A Pod is the smallest deployable unit in Kubernetes - a group of one or more containers.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Pod Concept                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                         Pod                              │   │
│   │  ┌───────────────┐  ┌───────────────┐                   │   │
│   │  │  Container 1  │  │  Container 2  │                   │   │
│   │  │   (nginx)     │  │   (sidecar)   │                   │   │
│   │  └───────────────┘  └───────────────┘                   │   │
│   │                                                          │   │
│   │  Shared:                                                 │   │
│   │  • Network namespace (same IP)                          │   │
│   │  • Storage volumes                                       │   │
│   │  • IPC namespace                                        │   │
│   │                                                          │   │
│   │  Pod IP: 10.244.1.5                                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Key Points:                                                   │
│   • Containers in a pod share localhost                         │
│   • Pods are ephemeral (can be deleted/recreated)              │
│   • Each pod gets a unique IP address                          │
│   • Usually run one main container per pod                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Pod Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                      Pod Lifecycle                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Pending ──► Running ──► Succeeded/Failed                      │
│      │           │                                               │
│      │           └──► CrashLoopBackOff (restart loop)           │
│      │                                                           │
│      └──► ImagePullBackOff (can't pull image)                   │
│                                                                  │
│   Pod Phases:                                                   │
│   • Pending    - Accepted but containers not running            │
│   • Running    - At least one container running                 │
│   • Succeeded  - All containers terminated successfully         │
│   • Failed     - All containers terminated, one with error      │
│   • Unknown    - State cannot be determined                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Creating Pods

### Basic Pod YAML

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: development
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

```bash
# Create pod
kubectl apply -f pod.yaml

# Create pod imperatively
kubectl run nginx --image=nginx

# Generate YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

### Pod with Multiple Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo $(date) > /data/index.html; sleep 10; done']
    volumeMounts:
    - name: shared-data
      mountPath: /data

  volumes:
  - name: shared-data
    emptyDir: {}
```

### Pod with Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DATABASE_HOST
      value: "mysql.default.svc.cluster.local"
    - name: DATABASE_PORT
      value: "3306"
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
    - name: CONFIG_VALUE
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: some-key
```

### Pod with Resource Limits

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
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Pod with Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: data-volume
      mountPath: /data

  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-pvc
```

## Pod Commands and Args

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-pod
spec:
  containers:
  - name: app
    image: busybox
    # Override ENTRYPOINT
    command: ["echo"]
    # Override CMD
    args: ["Hello, Kubernetes!"]
```

```yaml
# Multiple commands
apiVersion: v1
kind: Pod
metadata:
  name: multi-command-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "echo Hello && sleep 3600"]
```

## Init Containers

Init containers run before the main containers start.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done']

  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done']

  containers:
  - name: app
    image: myapp:latest
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Init Containers Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Pod Created                                                   │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│   │ Init-1 Run  │───►│ Init-2 Run  │───►│ Main Start  │        │
│   │ (complete)  │    │ (complete)  │    │  (running)  │        │
│   └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                  │
│   • Init containers run sequentially                            │
│   • Each must complete successfully                             │
│   • Main containers start after all init containers succeed     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Restart Policies

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-pod
spec:
  restartPolicy: Always  # Default for Pods in Deployment
  # restartPolicy: OnFailure  # For Jobs
  # restartPolicy: Never  # For one-time pods
  containers:
  - name: app
    image: myapp:latest
```

| Policy | Description |
|--------|-------------|
| `Always` | Always restart (default for Deployments) |
| `OnFailure` | Restart only on failure (for Jobs) |
| `Never` | Never restart |

## Node Scheduling

### NodeSelector

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-pod
spec:
  nodeSelector:
    disktype: ssd
    kubernetes.io/os: linux
  containers:
  - name: app
    image: myapp:latest
```

### NodeName (Direct Assignment)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodename-pod
spec:
  nodeName: worker-node-1
  containers:
  - name: app
    image: myapp:latest
```

### Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: myapp:latest
```

### Tolerations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
  - key: "special"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: app
    image: myapp:latest
```

## Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

## Pod Management Commands

```bash
# Create pod
kubectl apply -f pod.yaml
kubectl run nginx --image=nginx

# List pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels

# Describe pod
kubectl describe pod nginx

# Get pod YAML
kubectl get pod nginx -o yaml

# Delete pod
kubectl delete pod nginx
kubectl delete -f pod.yaml

# Execute command
kubectl exec nginx -- ls /
kubectl exec -it nginx -- /bin/bash

# View logs
kubectl logs nginx
kubectl logs nginx -f
kubectl logs nginx -c container-name

# Port forward
kubectl port-forward nginx 8080:80

# Copy files
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
kubectl cp ./file.txt nginx:/tmp/file.txt
```

## Pod Debugging

```bash
# Check pod status
kubectl get pod nginx -o wide

# Describe for events
kubectl describe pod nginx

# Check logs
kubectl logs nginx
kubectl logs nginx --previous

# Run debug container
kubectl debug nginx -it --image=busybox

# Create ephemeral debug container (K8s 1.23+)
kubectl debug -it nginx --image=busybox --target=nginx
```

## Static Pods

Static pods are managed directly by kubelet, not the API server.

```bash
# Static pod manifest location
/etc/kubernetes/manifests/

# Create static pod
sudo cp pod.yaml /etc/kubernetes/manifests/static-pod.yaml

# kubelet watches this directory and manages static pods
```

## Quick Reference

### Pod Spec Fields

| Field | Description |
|-------|-------------|
| `containers` | List of containers |
| `initContainers` | Init containers |
| `volumes` | Volumes for containers |
| `nodeSelector` | Node selection |
| `affinity` | Advanced scheduling |
| `tolerations` | Tolerate taints |
| `restartPolicy` | Restart behavior |
| `securityContext` | Security settings |
| `serviceAccountName` | Service account |

### Container Spec Fields

| Field | Description |
|-------|-------------|
| `name` | Container name |
| `image` | Image to use |
| `command` | Entrypoint override |
| `args` | Arguments |
| `env` | Environment variables |
| `ports` | Container ports |
| `resources` | Resource limits |
| `volumeMounts` | Mount volumes |
| `livenessProbe` | Health check |
| `readinessProbe` | Ready check |

---

**Previous:** [04-kubectl-basics.md](04-kubectl-basics.md) | **Next:** [06-replicasets.md](06-replicasets.md)
