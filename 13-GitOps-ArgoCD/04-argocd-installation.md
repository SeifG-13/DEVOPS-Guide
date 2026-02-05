# ArgoCD Installation

## Prerequisites

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PREREQUISITES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Required:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Kubernetes cluster (v1.23+)                               │   │
│  │ • kubectl configured with cluster access                    │   │
│  │ • Cluster admin permissions                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Optional:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Helm v3 (for Helm installation method)                    │   │
│  │ • Ingress controller (for external access)                  │   │
│  │ • TLS certificates (for HTTPS)                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Minimum Resources (per component):                                 │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ Component              │ CPU      │ Memory                 │    │
│  ├────────────────────────┼──────────┼────────────────────────┤    │
│  │ argocd-server          │ 50m      │ 64Mi                   │    │
│  │ argocd-repo-server     │ 50m      │ 64Mi                   │    │
│  │ argocd-app-controller  │ 100m     │ 128Mi                  │    │
│  │ argocd-dex-server      │ 10m      │ 32Mi                   │    │
│  │ argocd-redis           │ 50m      │ 64Mi                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Installation Methods

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INSTALLATION METHODS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Method              │ Best For                              │   │
│  ├─────────────────────┼───────────────────────────────────────┤   │
│  │ kubectl (manifest)  │ Quick start, learning                 │   │
│  │ Helm chart          │ Production, customization             │   │
│  │ Operator (OLM)      │ OpenShift, enterprise                 │   │
│  │ Autopilot           │ Automated GitOps bootstrap            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Method 1: kubectl Installation (Recommended for Learning)

### Standard Installation

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD (stable version)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### High Availability Installation

```bash
# For production environments with HA
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

### Installation Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INSTALLATION TYPES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Standard (install.yaml)                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Single replica of each component                          │   │
│  │ • Good for: development, testing, small teams               │   │
│  │ • Not recommended for: production                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  High Availability (ha/install.yaml)                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Multiple replicas with leader election                    │   │
│  │ • Redis HA with Sentinel                                    │   │
│  │ • Good for: production environments                         │   │
│  │ • Requires: More resources                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Core Only (core-install.yaml)                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • No UI, no API server                                      │   │
│  │ • Only application controller and repo server               │   │
│  │ • Good for: headless/automated deployments                  │   │
│  │ • Minimal resource footprint                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Verify Installation

```bash
# Check all pods are running
kubectl get pods -n argocd

# Expected output:
# NAME                                               READY   STATUS    RESTARTS
# argocd-application-controller-0                   1/1     Running   0
# argocd-dex-server-xxxxxxxxxx-xxxxx               1/1     Running   0
# argocd-redis-xxxxxxxxxx-xxxxx                    1/1     Running   0
# argocd-repo-server-xxxxxxxxxx-xxxxx              1/1     Running   0
# argocd-server-xxxxxxxxxx-xxxxx                   1/1     Running   0

# Check services
kubectl get svc -n argocd

# Check CRDs installed
kubectl get crd | grep argo
```

---

## Method 2: Helm Installation (Recommended for Production)

### Add Helm Repository

```bash
# Add the ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### Basic Helm Installation

```bash
# Create namespace
kubectl create namespace argocd

# Install with default values
helm install argocd argo/argo-cd -n argocd
```

### Customized Helm Installation

```bash
# Install with custom values
helm install argocd argo/argo-cd -n argocd \
  --set server.service.type=LoadBalancer \
  --set server.ingress.enabled=true \
  --set server.ingress.hosts[0]=argocd.example.com
```

### Custom Values File

```yaml
# values.yaml
global:
  image:
    tag: "v2.9.3"  # Pin version

server:
  replicas: 2

  service:
    type: LoadBalancer
    # type: ClusterIP  # Use with ingress

  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - argocd.example.com
    tls:
      - secretName: argocd-tls
        hosts:
          - argocd.example.com

  # Extra args
  extraArgs:
    - --insecure  # Disable TLS (if using ingress TLS termination)

controller:
  replicas: 1

  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

repoServer:
  replicas: 2

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

redis:
  enabled: true

redis-ha:
  enabled: false  # Set to true for HA

dex:
  enabled: true

configs:
  params:
    server.insecure: true  # If using ingress with TLS

  repositories:
    private-repo:
      url: https://github.com/org/private-repo
      password: ${GITHUB_TOKEN}
      username: not-used

  credentialTemplates:
    github-https:
      url: https://github.com/org
      password: ${GITHUB_TOKEN}
      username: not-used
```

### Install with Values File

```bash
helm install argocd argo/argo-cd -n argocd -f values.yaml
```

### Upgrade ArgoCD

```bash
# Update repo
helm repo update

# Upgrade
helm upgrade argocd argo/argo-cd -n argocd -f values.yaml
```

---

## Method 3: Operator Installation (OpenShift/OLM)

### Using OpenShift GitOps Operator

```yaml
# Install operator via OLM
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

### ArgoCD Instance via Operator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: argocd
spec:
  server:
    route:
      enabled: true
      tls:
        termination: reencrypt
  dex:
    openShiftOAuth: true
  rbac:
    defaultPolicy: ''
    policy: |
      g, system:cluster-admins, role:admin
    scopes: '[groups]'
```

---

## Accessing ArgoCD

### Option 1: Port Forward (Quick Access)

```bash
# Port forward the API server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access at https://localhost:8080
```

### Option 2: LoadBalancer Service

```bash
# Patch service to LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get external IP
kubectl get svc argocd-server -n argocd

# Access at https://<EXTERNAL-IP>
```

### Option 3: Ingress (Recommended for Production)

```yaml
# argocd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
  tls:
    - hosts:
        - argocd.example.com
      secretName: argocd-tls
```

```bash
kubectl apply -f argocd-ingress.yaml
```

### Option 4: Ingress with HTTP (TLS Termination at Ingress)

```yaml
# When using TLS termination at ingress level
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
  tls:
    - hosts:
        - argocd.example.com
      secretName: argocd-tls
```

```bash
# Also need to run server in insecure mode
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'

# Restart server
kubectl rollout restart deployment argocd-server -n argocd
```

---

## Initial Admin Password

### Get Initial Password

```bash
# ArgoCD v1.9+
# Password is stored in a secret
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Username: admin
```

### Change Admin Password

```bash
# Using CLI (after logging in)
argocd account update-password

# Or via UI: Settings → Accounts → admin → Update Password
```

### Delete Initial Secret (After Password Change)

```bash
# Recommended: Delete the initial secret after changing password
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## Post-Installation Configuration

### Configure TLS Certificate

```yaml
# argocd-tls-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-server-tls
  namespace: argocd
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

### Configure Repository Credentials

```bash
# Add private repository via CLI
argocd repo add https://github.com/org/private-repo \
  --username git \
  --password $GITHUB_TOKEN

# Or via secret
kubectl create secret generic repo-creds \
  -n argocd \
  --from-literal=url=https://github.com/org/private-repo \
  --from-literal=username=git \
  --from-literal=password=$GITHUB_TOKEN
kubectl label secret repo-creds -n argocd argocd.argoproj.io/secret-type=repository
```

### Configure RBAC

```yaml
# argocd-rbac-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # Grant admin to users in 'admin' group
    g, admin-group, role:admin

    # Read-only access for developers
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    g, dev-group, role:developer

  policy.default: role:readonly
```

---

## Verify Installation

### Complete Verification Checklist

```bash
# 1. Check all pods are running
kubectl get pods -n argocd
# All should be Running with 1/1 READY

# 2. Check all services
kubectl get svc -n argocd

# 3. Check CRDs
kubectl get crd | grep argoproj

# 4. Check API server is responding
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
curl -k https://localhost:8080/api/version

# 5. Login with CLI
argocd login localhost:8080 --insecure

# 6. List default project
argocd proj list

# 7. Check cluster connectivity
argocd cluster list
```

### Expected CRDs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ARGOCD CRDS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  applications.argoproj.io                                          │
│  applicationsets.argoproj.io                                       │
│  appprojects.argoproj.io                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting Installation

### Common Issues

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TROUBLESHOOTING                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Issue: Pods stuck in Pending                                       │
│  ─────────────────────────────                                      │
│  kubectl describe pod <pod-name> -n argocd                         │
│  • Check for resource constraints                                  │
│  • Check PVC binding (if using PVs)                                │
│                                                                     │
│  Issue: Cannot access UI                                            │
│  ────────────────────────                                           │
│  • Verify service type (ClusterIP vs LoadBalancer)                 │
│  • Check ingress configuration                                     │
│  • Verify TLS certificates                                         │
│  kubectl logs deployment/argocd-server -n argocd                   │
│                                                                     │
│  Issue: Login fails                                                 │
│  ──────────────────                                                 │
│  • Verify initial admin secret exists                              │
│  • Check dex-server logs for SSO issues                            │
│  kubectl logs deployment/argocd-dex-server -n argocd               │
│                                                                     │
│  Issue: Repository connection fails                                 │
│  ──────────────────────────────────                                 │
│  kubectl logs deployment/argocd-repo-server -n argocd              │
│  • Check credentials                                               │
│  • Verify network access to Git server                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### View Component Logs

```bash
# API Server logs
kubectl logs -n argocd deployment/argocd-server

# Application Controller logs
kubectl logs -n argocd statefulset/argocd-application-controller

# Repo Server logs
kubectl logs -n argocd deployment/argocd-repo-server

# Dex logs
kubectl logs -n argocd deployment/argocd-dex-server
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Installation Commands:                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ # Quick install                                             │   │
│  │ kubectl create namespace argocd                             │   │
│  │ kubectl apply -n argocd -f \                                │   │
│  │   https://raw.githubusercontent.com/argoproj/argo-cd/\      │   │
│  │   stable/manifests/install.yaml                             │   │
│  │                                                             │   │
│  │ # Get password                                              │   │
│  │ kubectl -n argocd get secret argocd-initial-admin-secret \  │   │
│  │   -o jsonpath="{.data.password}" | base64 -d                │   │
│  │                                                             │   │
│  │ # Access UI                                                 │   │
│  │ kubectl port-forward svc/argocd-server -n argocd 8080:443   │   │
│  │ # Open https://localhost:8080                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Post-Installation:                                                 │
│  1. Change admin password                                          │
│  2. Configure repository credentials                               │
│  3. Set up ingress for external access                             │
│  4. Configure SSO if needed                                        │
│  5. Set up RBAC policies                                           │
│                                                                     │
│  Next: Learn the ArgoCD CLI                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
