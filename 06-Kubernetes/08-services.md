# Kubernetes Services

## What is a Service?

A Service is an abstraction that defines a logical set of Pods and a policy to access them.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Service Concept                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without Service:                                              │
│   ┌─────┐ ┌─────┐ ┌─────┐                                      │
│   │Pod 1│ │Pod 2│ │Pod 3│   Pods have dynamic IPs              │
│   │10.1 │ │10.2 │ │10.3 │   IPs change when pods restart       │
│   └─────┘ └─────┘ └─────┘                                      │
│                                                                  │
│   With Service:                                                 │
│   ┌─────────────────────────────────────┐                      │
│   │          Service (nginx-svc)         │                      │
│   │          ClusterIP: 10.96.0.10       │                      │
│   │          DNS: nginx-svc.default      │                      │
│   └─────────────────────────────────────┘                      │
│               │           │          │                          │
│               ▼           ▼          ▼                          │
│           ┌─────┐     ┌─────┐    ┌─────┐                       │
│           │Pod 1│     │Pod 2│    │Pod 3│                       │
│           └─────┘     └─────┘    └─────┘                       │
│                                                                  │
│   Service provides:                                             │
│   • Stable IP address                                           │
│   • DNS name                                                    │
│   • Load balancing                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Service Types

```
┌─────────────────────────────────────────────────────────────────┐
│                      Service Types                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ClusterIP (default)                                           │
│   └── Internal cluster access only                              │
│                                                                  │
│   NodePort                                                      │
│   └── External access via node IP + port                        │
│                                                                  │
│   LoadBalancer                                                  │
│   └── External access via cloud load balancer                   │
│                                                                  │
│   ExternalName                                                  │
│   └── Maps to external DNS name                                 │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐   │
│   │                      External                           │   │
│   │  ┌──────────────────────────────────────────────────┐  │   │
│   │  │              LoadBalancer                         │  │   │
│   │  │  ┌────────────────────────────────────────────┐  │  │   │
│   │  │  │               NodePort                      │  │  │   │
│   │  │  │  ┌──────────────────────────────────────┐  │  │  │   │
│   │  │  │  │             ClusterIP                 │  │  │  │   │
│   │  │  │  │              (Pods)                   │  │  │  │   │
│   │  │  │  └──────────────────────────────────────┘  │  │  │   │
│   │  │  └────────────────────────────────────────────┘  │  │   │
│   │  └──────────────────────────────────────────────────┘  │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## ClusterIP Service

Internal cluster communication only.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: ClusterIP  # Default type
  selector:
    app: nginx
  ports:
  - port: 80         # Service port
    targetPort: 80   # Container port
    protocol: TCP
```

```bash
# Create service
kubectl apply -f service.yaml

# Create imperatively
kubectl expose deployment nginx --port=80 --target-port=80

# Access from within cluster
curl http://nginx-clusterip.default.svc.cluster.local:80
curl http://nginx-clusterip:80  # Same namespace
```

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None  # Headless - no ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

Headless services:
- No load balancing
- Returns pod IPs directly via DNS
- Used for stateful applications

## NodePort Service

Exposes service on each node's IP at a static port.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80           # Service port (internal)
    targetPort: 80     # Container port
    nodePort: 30080    # Node port (30000-32767)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                     NodePort Service                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   External Traffic                                              │
│        │                                                        │
│        │ http://node-ip:30080                                   │
│        ▼                                                        │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│   │   Node 1    │   │   Node 2    │   │   Node 3    │          │
│   │ :30080      │   │ :30080      │   │ :30080      │          │
│   └─────────────┘   └─────────────┘   └─────────────┘          │
│        │                  │                  │                  │
│        └──────────────────┼──────────────────┘                  │
│                           │                                      │
│                           ▼                                      │
│                    ┌─────────────┐                              │
│                    │   Service   │                              │
│                    │  ClusterIP  │                              │
│                    └─────────────┘                              │
│                           │                                      │
│              ┌────────────┼────────────┐                        │
│              ▼            ▼            ▼                        │
│          ┌─────┐      ┌─────┐      ┌─────┐                     │
│          │ Pod │      │ Pod │      │ Pod │                     │
│          └─────┘      └─────┘      └─────┘                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Create NodePort service
kubectl expose deployment nginx --type=NodePort --port=80

# Get NodePort
kubectl get svc nginx-nodeport

# Access
curl http://<node-ip>:30080
```

## LoadBalancer Service

Provisions external load balancer (cloud providers).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   LoadBalancer Service                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   External Traffic                                              │
│        │                                                        │
│        │ http://external-ip:80                                  │
│        ▼                                                        │
│   ┌─────────────────────────────────────────────┐              │
│   │           Cloud Load Balancer                │              │
│   │        (AWS ELB, GCP LB, Azure LB)          │              │
│   └─────────────────────────────────────────────┘              │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│   │   Node 1    │   │   Node 2    │   │   Node 3    │          │
│   │  NodePort   │   │  NodePort   │   │  NodePort   │          │
│   └─────────────┘   └─────────────┘   └─────────────┘          │
│        │                  │                  │                  │
│        └──────────────────┼──────────────────┘                  │
│                           ▼                                      │
│                        Pods                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Create LoadBalancer service
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Get external IP (may take a moment)
kubectl get svc nginx-loadbalancer

# Output:
# NAME                 TYPE           EXTERNAL-IP      PORT(S)
# nginx-loadbalancer   LoadBalancer   203.0.113.10     80:30080/TCP
```

## ExternalName Service

Maps service to external DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

```bash
# Access external service
# Inside pod: nslookup external-db
# Returns: database.example.com
```

## Multi-Port Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-svc
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

## Service Discovery

### DNS-Based Discovery

```bash
# Full DNS name format
<service-name>.<namespace>.svc.cluster.local

# Examples
nginx.default.svc.cluster.local      # Full name
nginx.default                        # Namespace included
nginx                               # Same namespace only

# SRV records for ports
_http._tcp.nginx.default.svc.cluster.local
```

### Environment Variables

```bash
# Kubernetes injects env vars for services
# Format: <SERVICE_NAME>_SERVICE_HOST
#         <SERVICE_NAME>_SERVICE_PORT

# Example for 'nginx' service:
NGINX_SERVICE_HOST=10.96.0.10
NGINX_SERVICE_PORT=80
```

## Session Affinity

Route all requests from a client to the same pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-sticky
spec:
  selector:
    app: nginx
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  ports:
  - port: 80
```

## External IPs

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-external
spec:
  selector:
    app: nginx
  ports:
  - port: 80
  externalIPs:
  - 192.168.1.100
  - 192.168.1.101
```

## Endpoints

Services automatically create Endpoints to track pod IPs.

```bash
# View endpoints
kubectl get endpoints nginx

# Output:
# NAME    ENDPOINTS
# nginx   10.244.1.5:80,10.244.2.6:80,10.244.3.7:80
```

### Manual Endpoints

```yaml
# Service without selector
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
---
# Manual endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
- addresses:
  - ip: 192.168.1.100
  - ip: 192.168.1.101
  ports:
  - port: 80
```

## Service Commands

```bash
# Create service
kubectl expose deployment nginx --port=80 --target-port=80
kubectl apply -f service.yaml

# List services
kubectl get services
kubectl get svc

# Describe service
kubectl describe svc nginx

# Get endpoints
kubectl get endpoints

# Delete service
kubectl delete svc nginx

# Get service YAML
kubectl get svc nginx -o yaml

# Port forward to service
kubectl port-forward svc/nginx 8080:80
```

## Practical Examples

### Full Application Stack

```yaml
# Database Service (internal only)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
---
# API Service (internal)
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 8080
---
# Frontend Service (external)
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
```

### Service with Different Port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80           # Clients connect to port 80
    targetPort: 8080   # Traffic goes to container port 8080
```

## Quick Reference

### Service Types

| Type | Scope | Use Case |
|------|-------|----------|
| ClusterIP | Internal | Default, inter-service communication |
| NodePort | External | Development, debugging |
| LoadBalancer | External | Production, cloud environments |
| ExternalName | External | Connect to external services |

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl expose` | Create service |
| `kubectl get svc` | List services |
| `kubectl describe svc` | Show details |
| `kubectl get endpoints` | Show endpoints |
| `kubectl port-forward svc/name` | Forward port |

### Port Terminology

| Port | Description |
|------|-------------|
| `port` | Service port (what clients use) |
| `targetPort` | Container port |
| `nodePort` | Node port (30000-32767) |

---

**Previous:** [07-deployments.md](07-deployments.md) | **Next:** [09-namespaces.md](09-namespaces.md)
