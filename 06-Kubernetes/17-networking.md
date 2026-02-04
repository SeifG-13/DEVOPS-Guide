# Kubernetes Networking

## Kubernetes Network Model

```
┌─────────────────────────────────────────────────────────────────┐
│                 Kubernetes Network Model                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Fundamental Requirements:                                     │
│                                                                  │
│   1. Every pod gets its own IP address                          │
│   2. Pods can communicate with all other pods without NAT       │
│   3. Nodes can communicate with all pods without NAT            │
│   4. The IP that a pod sees itself as is the same IP others see │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     Cluster Network                      │   │
│   │                                                          │   │
│   │   Node 1                           Node 2                │   │
│   │   ┌────────────────────┐          ┌────────────────────┐│   │
│   │   │ Pod A    Pod B     │          │ Pod C    Pod D     ││   │
│   │   │ 10.1.1.5 10.1.1.6  │◄────────►│ 10.1.2.5 10.1.2.6  ││   │
│   │   └────────────────────┘          └────────────────────┘│   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Pod Networking

### Pod-to-Pod Communication

```yaml
# Pods communicate directly via IP
apiVersion: v1
kind: Pod
metadata:
  name: client
spec:
  containers:
  - name: client
    image: busybox
    command: ['sh', '-c', 'wget -qO- http://10.1.2.5:80']  # Direct pod IP
```

### Container-to-Container in Same Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: web
    image: nginx
    ports:
    - containerPort: 80
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'wget -qO- http://localhost:80']  # localhost works
```

## DNS in Kubernetes

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes DNS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Service DNS:                                                  │
│   <service>.<namespace>.svc.cluster.local                       │
│                                                                  │
│   Examples:                                                     │
│   • api-service.default.svc.cluster.local                       │
│   • db.production.svc.cluster.local                             │
│   • api-service.default (short form)                            │
│   • api-service (same namespace)                                │
│                                                                  │
│   Pod DNS (headless service):                                   │
│   <pod-name>.<service>.<namespace>.svc.cluster.local            │
│                                                                  │
│   Example:                                                      │
│   • mysql-0.mysql-headless.default.svc.cluster.local            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### CoreDNS Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### Custom DNS Policy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - mycompany.local
    options:
      - name: ndots
        value: "2"
  containers:
  - name: app
    image: myapp
```

| DNS Policy | Description |
|-----------|-------------|
| ClusterFirst | Use cluster DNS (default) |
| ClusterFirstWithHostNet | Use cluster DNS with host network |
| Default | Use node's DNS |
| None | Custom DNS via dnsConfig |

## Network Policies

Control traffic flow between pods.

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow Specific Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow from Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
```

### Allow Egress to External

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
```

### Combined Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from frontend pods
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  # Allow from ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Allow to database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

## CNI Plugins

Container Network Interface plugins implement pod networking.

```
┌─────────────────────────────────────────────────────────────────┐
│                     CNI Plugins                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Plugin      Features                                          │
│   ─────────────────────────────────────────────────────         │
│   Calico      Network policies, BGP, high performance           │
│   Cilium      eBPF-based, L7 policies, observability            │
│   Flannel     Simple overlay, no network policies               │
│   Weave       Encryption, simple setup                          │
│   AWS VPC CNI Native AWS VPC networking                         │
│   Azure CNI   Native Azure networking                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Install Calico

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

### Install Cilium

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system
```

## Service Networking

```
┌─────────────────────────────────────────────────────────────────┐
│                   Service Types                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ClusterIP (10.96.0.1)                                         │
│   └── Internal cluster access                                   │
│                                                                  │
│   NodePort (30000-32767)                                        │
│   └── External access via node IP + port                        │
│                                                                  │
│   LoadBalancer                                                  │
│   └── Cloud load balancer + NodePort + ClusterIP               │
│                                                                  │
│   ExternalName                                                  │
│   └── DNS CNAME to external service                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Service CIDR Configuration

```bash
# kube-apiserver service CIDR
--service-cluster-ip-range=10.96.0.0/12

# Check current service CIDR
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range
```

## Pod CIDR

```bash
# Check pod CIDR
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# kube-controller-manager configuration
--cluster-cidr=10.244.0.0/16
--allocate-node-cidrs=true
```

## kube-proxy Modes

```
┌─────────────────────────────────────────────────────────────────┐
│                   kube-proxy Modes                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   iptables (default)                                            │
│   ├── Uses iptables rules                                       │
│   ├── No additional software needed                             │
│   └── Good for most clusters                                    │
│                                                                  │
│   ipvs                                                          │
│   ├── Uses IPVS (IP Virtual Server)                            │
│   ├── Better performance for large clusters                     │
│   ├── More load balancing options                               │
│   └── Requires ipvs kernel modules                              │
│                                                                  │
│   userspace (legacy)                                            │
│   └── Proxies in userspace, slow                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Configure IPVS Mode

```yaml
# kube-proxy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"
    ipvs:
      scheduler: "rr"  # round-robin
```

## Debugging Network Issues

```bash
# Check pod networking
kubectl exec -it debug-pod -- ping 10.1.2.5
kubectl exec -it debug-pod -- nslookup kubernetes

# Check DNS
kubectl exec -it debug-pod -- cat /etc/resolv.conf
kubectl exec -it debug-pod -- nslookup api-service.default

# Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy allow-frontend

# Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Check CNI
kubectl get pods -n kube-system -l k8s-app=calico-node

# Debug with netshoot
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
```

## Quick Reference

### Network Commands

| Command | Description |
|---------|-------------|
| `kubectl get svc` | List services |
| `kubectl get endpoints` | List endpoints |
| `kubectl get networkpolicies` | List network policies |
| `kubectl describe np` | Show policy details |

### DNS Names

| Resource | DNS Format |
|----------|-----------|
| Service | `<svc>.<ns>.svc.cluster.local` |
| Pod (headless) | `<pod>.<svc>.<ns>.svc.cluster.local` |

### Network Policy Selectors

| Selector | Description |
|----------|-------------|
| `podSelector` | Select pods by labels |
| `namespaceSelector` | Select namespaces |
| `ipBlock` | Select IP ranges |

---

**Previous:** [16-ingress.md](16-ingress.md) | **Next:** [18-rbac.md](18-rbac.md)
