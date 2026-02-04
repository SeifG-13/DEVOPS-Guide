# Kubernetes Ingress

## What is Ingress?

Ingress manages external access to services, typically HTTP/HTTPS.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Ingress Concept                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without Ingress              With Ingress                     │
│                                                                  │
│   Internet                     Internet                         │
│      │                            │                             │
│      ├─► LoadBalancer:30001      Ingress Controller            │
│      ├─► LoadBalancer:30002        │                           │
│      └─► LoadBalancer:30003        ├─► /api    → api-svc       │
│                                    ├─► /web    → web-svc       │
│   Each service needs              └─► /admin  → admin-svc     │
│   its own LoadBalancer                                         │
│                                    Single entry point          │
│                                    Path-based routing          │
│                                    SSL termination             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Ingress Features

| Feature | Description |
|---------|-------------|
| Path-based routing | Route by URL path |
| Host-based routing | Route by hostname |
| TLS termination | Handle HTTPS |
| Load balancing | Distribute traffic |
| Name-based virtual hosting | Multiple domains |

## Ingress Controllers

Ingress requires an Ingress Controller to work.

| Controller | Description |
|-----------|-------------|
| NGINX Ingress | Most popular, NGINX-based |
| Traefik | Cloud-native, auto-discovery |
| HAProxy | High-performance |
| AWS ALB | AWS Application Load Balancer |
| GCE | Google Cloud Load Balancer |
| Contour | Envoy-based |

### Install NGINX Ingress Controller

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx

# Or using manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## Creating Ingress Resources

### Basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /admin
        pathType: Exact
        backend:
          service:
            name: admin-service
            port:
              number: 8000
```

### Host-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

## Path Types

| Type | Matching |
|------|----------|
| `Prefix` | Matches URL path prefix |
| `Exact` | Matches exact URL path |
| `ImplementationSpecific` | Controller-dependent |

```yaml
# Prefix: /api matches /api, /api/, /api/users
- path: /api
  pathType: Prefix

# Exact: /api matches only /api
- path: /api
  pathType: Exact
```

## TLS/HTTPS

### Create TLS Secret

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout tls.key -out tls.crt \
    -subj "/CN=example.com"

# Create secret
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

### Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Multiple TLS Certificates

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tls-ingress
spec:
  tls:
  - hosts:
    - app1.example.com
    secretName: app1-tls
  - hosts:
    - app2.example.com
    secretName: app2-tls
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

## Ingress Annotations

Annotations configure controller-specific behavior.

### NGINX Ingress Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Rewrite target
    nginx.ingress.kubernetes.io/rewrite-target: /

    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Proxy settings
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"

    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### URL Rewriting

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

Request `/api/users` → Backend receives `/users`

## Default Backend

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## IngressClass

```yaml
# IngressClass for NGINX
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
---
# Ingress using specific class
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
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
            name: web-service
            port:
              number: 80
```

## Complete Example

```yaml
# Backend Services
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - port: 8080
---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

## Managing Ingress

```bash
# Create Ingress
kubectl apply -f ingress.yaml

# List Ingresses
kubectl get ingress
kubectl get ing

# Describe Ingress
kubectl describe ingress my-ingress

# Get Ingress address
kubectl get ingress my-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Delete Ingress
kubectl delete ingress my-ingress

# List IngressClasses
kubectl get ingressclass
```

## Troubleshooting

```bash
# Check Ingress Controller pods
kubectl get pods -n ingress-nginx

# Check Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Verify Ingress is configured
kubectl describe ingress my-ingress

# Check backend services
kubectl get svc
kubectl get endpoints

# Test from inside cluster
kubectl run test --rm -it --image=busybox -- wget -qO- http://web-service
```

## Quick Reference

### Common Annotations (NGINX)

| Annotation | Description |
|-----------|-------------|
| `rewrite-target` | URL rewriting |
| `ssl-redirect` | Force HTTPS |
| `proxy-body-size` | Max request body |
| `limit-rps` | Rate limiting |
| `auth-type` | Authentication type |

### Key Commands

| Command | Description |
|---------|-------------|
| `kubectl get ing` | List Ingresses |
| `kubectl describe ing` | Show details |
| `kubectl get ingressclass` | List classes |

### Path Types

| Type | Example |
|------|---------|
| Prefix | `/api` matches `/api/*` |
| Exact | `/api` matches only `/api` |

---

**Previous:** [15-jobs-cronjobs.md](15-jobs-cronjobs.md) | **Next:** [17-networking.md](17-networking.md)
