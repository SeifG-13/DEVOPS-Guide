# Kubernetes Cheat Sheet for DevOps Engineers

## Quick Reference Commands

### Cluster Info
```bash
kubectl cluster-info                    # Cluster information
kubectl get nodes                       # List nodes
kubectl describe node <node>            # Node details
kubectl top nodes                       # Node resource usage
kubectl get componentstatuses           # Cluster health
kubectl config view                     # View kubeconfig
kubectl config current-context          # Current context
kubectl config use-context <context>    # Switch context
```

### Pods
```bash
kubectl get pods                        # List pods
kubectl get pods -o wide                # Detailed view
kubectl get pods -A                     # All namespaces
kubectl get pods -w                     # Watch mode
kubectl describe pod <pod>              # Pod details
kubectl logs <pod>                      # Pod logs
kubectl logs <pod> -c <container>       # Container logs
kubectl logs -f <pod>                   # Follow logs
kubectl logs --previous <pod>           # Previous container logs
kubectl exec -it <pod> -- bash          # Execute in pod
kubectl port-forward <pod> 8080:80      # Port forward
kubectl cp <pod>:/path ./local          # Copy from pod
kubectl delete pod <pod>                # Delete pod
```

### Deployments
```bash
kubectl get deployments                 # List deployments
kubectl describe deployment <name>     # Deployment details
kubectl create deployment nginx --image=nginx
kubectl scale deployment <name> --replicas=3
kubectl rollout status deployment <name>
kubectl rollout history deployment <name>
kubectl rollout undo deployment <name>
kubectl rollout restart deployment <name>
kubectl set image deployment/<name> container=image:tag
```

### Services
```bash
kubectl get services                    # List services
kubectl get svc                         # Shorthand
kubectl expose deployment <name> --port=80 --type=LoadBalancer
kubectl describe service <name>
kubectl delete service <name>
```

### ConfigMaps & Secrets
```bash
# ConfigMaps
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=config.properties
kubectl get configmaps
kubectl describe configmap <name>

# Secrets
kubectl create secret generic <name> --from-literal=password=secret
kubectl create secret generic <name> --from-file=cert.pem
kubectl get secrets
kubectl describe secret <name>
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d
```

### Namespaces
```bash
kubectl get namespaces
kubectl create namespace <name>
kubectl delete namespace <name>
kubectl config set-context --current --namespace=<name>
kubectl get all -n <namespace>
```

### Apply & Delete
```bash
kubectl apply -f file.yaml              # Apply configuration
kubectl apply -f ./dir/                 # Apply directory
kubectl delete -f file.yaml             # Delete from file
kubectl delete pod,svc --all            # Delete all pods/services
kubectl delete all --all -n <namespace> # Delete everything
```

---

## Core Resource Manifests

### Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP  # NodePort, LoadBalancer
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
  tls:
    - hosts:
        - example.com
      secretName: tls-secret
```

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "db.example.com"
  config.json: |
    {
      "key": "value"
    }
```

### Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  password: cGFzc3dvcmQ=  # base64 encoded
stringData:
  api-key: plain-text-value
```

---

## Interview Q&A

### Q1: What is the difference between a Pod and a Deployment?
**A:**
- **Pod**: Smallest deployable unit, single or group of containers, ephemeral
- **Deployment**: Manages ReplicaSets, provides declarative updates, rolling updates, rollbacks, scaling

### Q2: Explain Service types in Kubernetes
**A:**
- **ClusterIP**: Internal cluster IP (default)
- **NodePort**: Exposes on each node's IP at static port (30000-32767)
- **LoadBalancer**: External load balancer (cloud providers)
- **ExternalName**: Maps to external DNS name

### Q3: What is the difference between ConfigMap and Secret?
**A:**
- **ConfigMap**: Non-sensitive configuration data, stored in plain text
- **Secret**: Sensitive data, base64 encoded, can be encrypted at rest

### Q4: Explain Kubernetes probes
**A:**
- **Liveness**: Restarts container if fails (is it alive?)
- **Readiness**: Removes from service if fails (is it ready to serve traffic?)
- **Startup**: Disables liveness/readiness until success (for slow-starting apps)

### Q5: What is a StatefulSet vs Deployment?
**A:**
- **Deployment**: Stateless apps, interchangeable pods
- **StatefulSet**: Stateful apps, stable network identity, persistent storage, ordered deployment

### Q6: How does rolling update work?
**A:**
1. Creates new ReplicaSet with new version
2. Gradually scales up new, scales down old
3. Maintains desired replicas during update
4. Controlled by `maxSurge` and `maxUnavailable`

### Q7: What is a DaemonSet?
**A:** Ensures a copy of a pod runs on all (or selected) nodes. Use cases:
- Log collectors (Fluentd)
- Monitoring agents (Prometheus node exporter)
- Network plugins

### Q8: Explain Kubernetes networking model
**A:**
- Every pod gets unique IP
- Pods can communicate without NAT
- Agents on nodes can communicate with all pods
- CNI plugins implement networking (Calico, Flannel, Cilium)

### Q9: What is an Ingress Controller?
**A:** Implementation of Ingress rules. Common controllers:
- NGINX Ingress Controller
- Traefik
- HAProxy
- AWS ALB Ingress Controller

### Q10: How do you troubleshoot a pod that won't start?
**A:**
```bash
kubectl describe pod <pod>              # Check events
kubectl logs <pod>                      # Check logs
kubectl logs <pod> --previous           # Previous container
kubectl get events                      # Cluster events
kubectl exec -it <pod> -- sh            # Debug inside pod
```
Check: ImagePullBackOff, CrashLoopBackOff, Pending (resources), ContainerCreating

---

## Useful Patterns

### Resource Limits
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### Environment from ConfigMap/Secret
```yaml
env:
  - name: DATABASE_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DATABASE_HOST
  - name: DATABASE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: password
```

### Volume Mounts
```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: app-secret
containers:
  - volumeMounts:
      - name: config-volume
        mountPath: /etc/config
      - name: secret-volume
        mountPath: /etc/secrets
        readOnly: true
```

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Debugging Commands

```bash
# Get all resources
kubectl get all -A

# Describe for events
kubectl describe pod <pod>

# Check resource usage
kubectl top pods
kubectl top nodes

# Get pod YAML
kubectl get pod <pod> -o yaml

# Run debug container
kubectl run debug --image=busybox -it --rm -- sh

# Check DNS
kubectl run test --image=busybox -it --rm -- nslookup kubernetes

# Check connectivity
kubectl exec <pod> -- curl -v service-name:port
```

---

## Best Practices

1. **Use namespaces** - Organize and isolate resources
2. **Set resource limits** - Prevent resource starvation
3. **Use probes** - Health checks for reliability
4. **Use labels** - Organize and select resources
5. **Use ConfigMaps/Secrets** - Externalize configuration
6. **Enable RBAC** - Principle of least privilege
7. **Use network policies** - Restrict pod communication
8. **Regular backups** - etcd and persistent volumes
