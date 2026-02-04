# Kubernetes Best Practices

## Application Design

### Twelve-Factor App Principles

```
┌─────────────────────────────────────────────────────────────────┐
│              12-Factor App for Kubernetes                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Codebase        One codebase, many deploys                 │
│   2. Dependencies    Explicitly declare dependencies            │
│   3. Config          Store config in environment                │
│   4. Backing Services Treat as attached resources               │
│   5. Build/Release/Run Strict separation                        │
│   6. Processes       Stateless, share-nothing                   │
│   7. Port Binding    Export via port binding                    │
│   8. Concurrency     Scale via process model                    │
│   9. Disposability   Fast startup, graceful shutdown           │
│   10. Dev/Prod Parity Keep environments similar                 │
│   11. Logs           Treat as event streams                     │
│   12. Admin Processes Run as one-off processes                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Stateless Applications

```yaml
# Good: Stateless application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: api:v1
        env:
        - name: SESSION_STORE
          value: "redis://redis:6379"  # External session store
```

## Pod Best Practices

### Always Set Resource Limits

```yaml
containers:
- name: app
  image: myapp:v1
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

### Always Use Health Probes

```yaml
containers:
- name: app
  image: myapp:v1
  livenessProbe:
    httpGet:
      path: /healthz
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

### Use Non-Root User

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

### Graceful Shutdown

```yaml
containers:
- name: app
  image: myapp:v1
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 10"]
terminationGracePeriodSeconds: 30
```

## Deployment Best Practices

### Use Rolling Updates

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0  # Zero downtime
```

### Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

### Use Pod Anti-Affinity

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: api
          topologyKey: kubernetes.io/hostname
```

### Multiple Replicas

```yaml
spec:
  replicas: 3  # At least 2 for high availability
```

## Configuration Best Practices

### Use ConfigMaps and Secrets

```yaml
# ConfigMap for non-sensitive data
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  API_ENDPOINT: "https://api.example.com"
---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "secret123"
```

### Externalize All Configuration

```yaml
containers:
- name: app
  image: myapp:v1
  envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secrets
```

## Security Best Practices

### Security Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│                   Security Checklist                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Pod Security                                                  │
│   □ Run as non-root user                                        │
│   □ Read-only root filesystem                                   │
│   □ Drop all capabilities                                       │
│   □ No privilege escalation                                     │
│   □ Set resource limits                                         │
│                                                                  │
│   Image Security                                                │
│   □ Use minimal base images                                     │
│   □ Scan images for vulnerabilities                             │
│   □ Use specific image tags (not latest)                        │
│   □ Pull from trusted registries                                │
│                                                                  │
│   Network Security                                              │
│   □ Use NetworkPolicies                                         │
│   □ Encrypt traffic with TLS                                    │
│   □ Limit exposed services                                      │
│                                                                  │
│   Access Control                                                │
│   □ Use RBAC with least privilege                               │
│   □ Use ServiceAccounts per application                         │
│   □ Rotate secrets regularly                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Network Policies

```yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

### RBAC Least Privilege

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  resourceNames: ["app-config", "app-secrets"]
  verbs: ["get"]
```

## Namespace Best Practices

### Organize by Team/Environment

```yaml
# Development namespace
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    env: development
---
# Production namespace with resource quota
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
```

## Monitoring and Logging

### Use Liveness and Readiness Probes

```yaml
livenessProbe:
  httpGet:
    path: /healthz
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

### Log to stdout/stderr

```yaml
# Application should log to stdout
# Don't log to files inside container
containers:
- name: app
  image: myapp:v1
  # Logs automatically captured by container runtime
```

### Expose Metrics

```yaml
containers:
- name: app
  image: myapp:v1
  ports:
  - name: http
    containerPort: 8080
  - name: metrics
    containerPort: 9090  # Prometheus metrics
```

## Labels and Annotations

### Standard Labels

```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: myapp-suite
    app.kubernetes.io/managed-by: helm
```

### Use Labels for Selection

```yaml
# Deployment
metadata:
  labels:
    app: myapp
    tier: frontend
    env: production

# Service selector
spec:
  selector:
    app: myapp
    tier: frontend
```

## Image Best Practices

### Use Specific Tags

```yaml
# Bad
image: nginx:latest

# Good
image: nginx:1.25.3
image: nginx:1.25.3-alpine

# Best (with digest)
image: nginx@sha256:abc123...
```

### Use Minimal Base Images

```dockerfile
# Use alpine or distroless
FROM node:20-alpine
# or
FROM gcr.io/distroless/nodejs20
```

## CI/CD Best Practices

### GitOps Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                   GitOps Workflow                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Developer                                                     │
│      │                                                          │
│      │ 1. Push code                                             │
│      ▼                                                          │
│   Git Repository                                                │
│      │                                                          │
│      │ 2. CI builds image, updates manifests                    │
│      ▼                                                          │
│   Config Repository                                             │
│      │                                                          │
│      │ 3. ArgoCD/Flux detects changes                          │
│      ▼                                                          │
│   Kubernetes Cluster                                            │
│      │                                                          │
│      │ 4. Applies manifests                                     │
│      ▼                                                          │
│   Application Running                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Production Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│                Production Readiness Checklist                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Reliability                                                   │
│   □ Multiple replicas (≥2)                                      │
│   □ Pod disruption budget                                       │
│   □ Pod anti-affinity                                           │
│   □ Health probes configured                                    │
│   □ Graceful shutdown handling                                  │
│                                                                  │
│   Resources                                                     │
│   □ Resource requests and limits set                            │
│   □ Horizontal pod autoscaler configured                        │
│   □ Resource quotas for namespace                               │
│                                                                  │
│   Security                                                      │
│   □ Run as non-root                                            │
│   □ Read-only root filesystem                                   │
│   □ Network policies defined                                    │
│   □ RBAC configured                                            │
│   □ Secrets properly managed                                    │
│   □ Images scanned for vulnerabilities                         │
│                                                                  │
│   Observability                                                 │
│   □ Logging to stdout/stderr                                    │
│   □ Metrics endpoint exposed                                    │
│   □ Tracing configured                                         │
│   □ Alerts configured                                          │
│                                                                  │
│   Operations                                                    │
│   □ Rolling update strategy                                     │
│   □ Rollback tested                                            │
│   □ Backup strategy for stateful apps                          │
│   □ Documentation updated                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Reference

### Pod Spec Best Practices

| Practice | Why |
|----------|-----|
| Set resource limits | Prevent resource starvation |
| Use probes | Ensure health monitoring |
| Run non-root | Security |
| Use read-only fs | Security |
| Set anti-affinity | High availability |

### Deployment Best Practices

| Practice | Why |
|----------|-----|
| Multiple replicas | High availability |
| Rolling updates | Zero downtime |
| PDB | Protect availability during maintenance |
| Specific image tags | Reproducibility |

### Security Best Practices

| Practice | Why |
|----------|-----|
| Network policies | Limit blast radius |
| RBAC | Least privilege |
| Scan images | Find vulnerabilities |
| Rotate secrets | Limit exposure |

---

**Previous:** [21-troubleshooting.md](21-troubleshooting.md)

---

## Congratulations!

You've completed the Kubernetes section of the DevOps roadmap. You now understand:

- Kubernetes architecture and core concepts
- Pod, Deployment, Service management
- ConfigMaps, Secrets, and Volumes
- StatefulSets, DaemonSets, Jobs
- Ingress and Networking
- RBAC and Security
- Resource management and scaling
- Troubleshooting and best practices

**Next Topic:** [07-CI-CD](../07-CI-CD/) 
