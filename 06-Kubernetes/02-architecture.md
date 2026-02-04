# Kubernetes Architecture

## Cluster Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                      Control Plane (Master)                     │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │     │
│  │  │  API Server  │  │   Scheduler  │  │ Controller Manager   │ │     │
│  │  │  (kube-api)  │  │              │  │                      │ │     │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘ │     │
│  │  ┌──────────────┐  ┌──────────────────────────────────────────┐│     │
│  │  │     etcd     │  │        Cloud Controller Manager          ││     │
│  │  │  (database)  │  │            (optional)                    ││     │
│  │  └──────────────┘  └──────────────────────────────────────────┘│     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                    │                                     │
│                                    │ API calls                           │
│                                    ▼                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌────────────────┐   │
│  │    Worker Node 1    │  │    Worker Node 2    │  │  Worker Node N │   │
│  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌──────────┐  │   │
│  │  │    kubelet    │  │  │  │    kubelet    │  │  │  │  kubelet │  │   │
│  │  └───────────────┘  │  │  └───────────────┘  │  │  └──────────┘  │   │
│  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌──────────┐  │   │
│  │  │  kube-proxy   │  │  │  │  kube-proxy   │  │  │  │kube-proxy│  │   │
│  │  └───────────────┘  │  │  └───────────────┘  │  │  └──────────┘  │   │
│  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌──────────┐  │   │
│  │  │  Container    │  │  │  │  Container    │  │  │  │Container │  │   │
│  │  │  Runtime      │  │  │  │  Runtime      │  │  │  │Runtime   │  │   │
│  │  └───────────────┘  │  │  └───────────────┘  │  │  └──────────┘  │   │
│  │  ┌─────┐ ┌─────┐   │  │  ┌─────┐ ┌─────┐   │  │  ┌─────┐       │   │
│  │  │ Pod │ │ Pod │   │  │  │ Pod │ │ Pod │   │  │  │ Pod │       │   │
│  │  └─────┘ └─────┘   │  │  └─────┘ └─────┘   │  │  └─────┘       │   │
│  └─────────────────────┘  └─────────────────────┘  └────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Control Plane Components

The Control Plane makes global decisions about the cluster and detects/responds to cluster events.

### API Server (kube-apiserver)

The front-end for the Kubernetes control plane.

```
┌─────────────────────────────────────────────────────────────────┐
│                      API Server                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Clients                     API Server                        │
│   ┌─────────┐                ┌─────────────────────┐            │
│   │ kubectl │───────────────►│                     │            │
│   └─────────┘                │  • Authentication   │            │
│   ┌─────────┐                │  • Authorization    │            │
│   │   UI    │───────────────►│  • Admission        │──────► etcd│
│   └─────────┘                │  • Validation       │            │
│   ┌─────────┐                │  • REST API         │            │
│   │  Apps   │───────────────►│                     │            │
│   └─────────┘                └─────────────────────┘            │
│                                                                  │
│   Functions:                                                    │
│   • Exposes Kubernetes API                                      │
│   • Authenticates and authorizes requests                       │
│   • Validates and processes API requests                        │
│   • Only component that communicates with etcd                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# API Server endpoints
kubectl api-resources        # List all API resources
kubectl api-versions         # List API versions
kubectl get --raw /api/v1    # Direct API call
```

### etcd

Distributed key-value store for all cluster data.

```
┌─────────────────────────────────────────────────────────────────┐
│                          etcd                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Stores:                                                       │
│   ├── /registry/pods/default/nginx-pod                         │
│   ├── /registry/services/default/nginx-svc                     │
│   ├── /registry/deployments/default/nginx-deploy               │
│   ├── /registry/secrets/default/my-secret                      │
│   └── /registry/configmaps/default/my-config                   │
│                                                                  │
│   Characteristics:                                              │
│   • Consistent and highly-available                             │
│   • Uses Raft consensus algorithm                               │
│   • All cluster state is stored here                            │
│   • Should be backed up regularly                               │
│                                                                  │
│   High Availability:                                            │
│   ┌────────┐    ┌────────┐    ┌────────┐                       │
│   │ etcd-1 │◄──►│ etcd-2 │◄──►│ etcd-3 │                       │
│   │(leader)│    │        │    │        │                       │
│   └────────┘    └────────┘    └────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Backup etcd (example)
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Scheduler (kube-scheduler)

Assigns Pods to Nodes based on resource requirements and constraints.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Scheduler                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   New Pod ──► Scheduler ──► Best Node                           │
│                   │                                              │
│                   ▼                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Scheduling Process                          │   │
│   │                                                          │   │
│   │  1. Filtering (find feasible nodes)                     │   │
│   │     ├── Enough CPU?                                     │   │
│   │     ├── Enough Memory?                                  │   │
│   │     ├── Node selectors match?                           │   │
│   │     ├── Taints/tolerations?                             │   │
│   │     └── Affinity rules?                                 │   │
│   │                                                          │   │
│   │  2. Scoring (rank feasible nodes)                       │   │
│   │     ├── Resource balance                                │   │
│   │     ├── Affinity preferences                            │   │
│   │     └── Custom priorities                               │   │
│   │                                                          │   │
│   │  3. Binding (assign Pod to highest-scored Node)         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Controller Manager (kube-controller-manager)

Runs controller processes that regulate cluster state.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Controller Manager                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Controllers (control loops):                                  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Node Controller                                         │   │
│   │  • Monitors node health                                  │   │
│   │  • Marks nodes as unavailable                           │   │
│   │  • Evicts pods from unhealthy nodes                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Replication Controller                                  │   │
│   │  • Maintains correct number of pods                     │   │
│   │  • Creates/deletes pods as needed                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Endpoints Controller                                    │   │
│   │  • Populates endpoint objects                           │   │
│   │  • Joins Services & Pods                                │   │
│   └─────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Service Account Controller                              │   │
│   │  • Creates default accounts for namespaces              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Cloud Controller Manager

Embeds cloud-specific control logic.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Cloud Controller Manager                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Controllers:                                                  │
│   • Node Controller - checks cloud for node deletion            │
│   • Route Controller - sets up routes in cloud                  │
│   • Service Controller - creates cloud load balancers           │
│                                                                  │
│   Cloud Providers:                                              │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │   AWS   │  │   GCP   │  │  Azure  │  │  Other  │           │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Worker Node Components

Worker nodes run the containerized applications.

### Kubelet

Primary node agent that runs on each worker node.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubelet                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐                                              │
│   │  API Server  │                                              │
│   └──────┬───────┘                                              │
│          │ PodSpec                                              │
│          ▼                                                       │
│   ┌──────────────┐     ┌────────────────────────────────────┐   │
│   │   Kubelet    │────►│  Container Runtime Interface (CRI) │   │
│   └──────────────┘     └────────────────────────────────────┘   │
│          │                              │                        │
│          │                              ▼                        │
│          │             ┌─────────────────────────────────────┐  │
│          │             │      Container Runtime              │  │
│          │             │  (containerd, CRI-O, Docker)        │  │
│          │             └─────────────────────────────────────┘  │
│          │                              │                        │
│          ▼                              ▼                        │
│   ┌─────────────┐              ┌─────────────┐                  │
│   │ Health Check│              │ Containers  │                  │
│   │ & Reporting │              │   Running   │                  │
│   └─────────────┘              └─────────────┘                  │
│                                                                  │
│   Responsibilities:                                             │
│   • Register node with API server                               │
│   • Watch for PodSpecs assigned to its node                     │
│   • Start/stop containers via container runtime                 │
│   • Monitor container and node health                           │
│   • Report status back to API server                            │
│   • Execute liveness, readiness, startup probes                 │
│   • Mount volumes, manage secrets/configmaps                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Check kubelet status
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet

# Kubelet configuration
cat /var/lib/kubelet/config.yaml
```

### Kube-proxy

Network proxy that runs on each node, maintaining network rules.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kube-proxy                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Service: nginx-service (ClusterIP: 10.96.0.1)                 │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    kube-proxy                            │   │
│   │                         │                                │   │
│   │                         ▼                                │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │              Network Rules                       │   │   │
│   │   │                                                  │   │   │
│   │   │  10.96.0.1:80 ──► 10.244.1.5:80 (Pod 1)        │   │   │
│   │   │              └──► 10.244.2.3:80 (Pod 2)        │   │   │
│   │   │              └──► 10.244.1.8:80 (Pod 3)        │   │   │
│   │   │                                                  │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Modes:                                                        │
│   • iptables (default) - uses iptables rules                    │
│   • ipvs - uses IPVS (IP Virtual Server)                        │
│   • userspace (legacy) - proxies in userspace                   │
│                                                                  │
│   Functions:                                                    │
│   • Implements Service abstraction                              │
│   • Maintains network rules on nodes                            │
│   • Performs connection forwarding                              │
│   • Handles load balancing for Services                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Container Runtime

Software responsible for running containers.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Container Runtime                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      Kubelet                                     │
│                         │                                        │
│                         │ CRI (Container Runtime Interface)      │
│                         ▼                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              High-Level Runtime                          │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │   │
│   │   │  containerd  │  │    CRI-O     │  │   Docker*    │ │   │
│   │   │  (default)   │  │  (Red Hat)   │  │ (deprecated) │ │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘ │   │
│   └─────────────────────────────────────────────────────────┘   │
│                         │                                        │
│                         │ OCI (Open Container Initiative)        │
│                         ▼                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │               Low-Level Runtime                          │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │   │
│   │   │     runc     │  │    crun      │  │   kata       │ │   │
│   │   │  (default)   │  │  (faster)    │  │ (VMs)        │ │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘ │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   * Docker support removed in K8s 1.24+, use containerd         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### containerd

Most common container runtime in Kubernetes.

```bash
# Check containerd status
systemctl status containerd

# List containers with ctr
ctr containers list

# List images
ctr images list

# Using crictl (CRI CLI)
crictl ps           # List running containers
crictl pods         # List pods
crictl images       # List images
crictl logs <id>    # Container logs
```

#### CRI-O

Lightweight container runtime designed specifically for Kubernetes.

```bash
# Check CRI-O status
systemctl status crio

# CRI-O uses crictl for management
crictl info
```

### Container Runtime Comparison

| Feature | containerd | CRI-O | Docker |
|---------|------------|-------|--------|
| CRI Support | Native | Native | Via shim (removed) |
| Image Format | OCI | OCI | OCI/Docker |
| Used By | Most distros | OpenShift | Legacy |
| Footprint | Light | Lighter | Heavy |
| K8s Support | Full | Full | Deprecated |

## Component Communication

```
┌─────────────────────────────────────────────────────────────────┐
│                    Communication Flow                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User                                                          │
│     │                                                            │
│     │ kubectl apply                                             │
│     ▼                                                            │
│   ┌──────────────┐                                              │
│   │  API Server  │◄────────────────────────────────┐            │
│   └──────────────┘                                  │            │
│     │         │                                     │            │
│     │         │                                     │            │
│     ▼         ▼                                     │            │
│   ┌────┐  ┌───────────┐  ┌────────────────────┐   │            │
│   │etcd│  │ Scheduler │  │ Controller Manager │   │            │
│   └────┘  └───────────┘  └────────────────────┘   │            │
│               │                     │              │            │
│               │ Pod binding         │ Watch        │            │
│               ▼                     ▼              │            │
│           API Server ◄─────────────────────────────┤            │
│               │                                    │            │
│               │ Pod assignment                     │            │
│               ▼                                    │            │
│   ┌───────────────────────────────────────────┐   │            │
│   │              Kubelet (on node)             │   │            │
│   │                    │                       │───┘            │
│   │                    ▼ Status reporting                       │
│   │         Container Runtime                   │               │
│   │                    │                       │               │
│   │                    ▼                       │               │
│   │            Container Running               │               │
│   └───────────────────────────────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Control Plane High Availability

```
┌─────────────────────────────────────────────────────────────────┐
│                    HA Control Plane                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    Load Balancer                                 │
│                         │                                        │
│          ┌──────────────┼──────────────┐                        │
│          │              │              │                        │
│          ▼              ▼              ▼                        │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐                 │
│   │  Master 1  │ │  Master 2  │ │  Master 3  │                 │
│   │ API Server │ │ API Server │ │ API Server │                 │
│   │ Scheduler  │ │ Scheduler  │ │ Scheduler  │                 │
│   │ Controller │ │ Controller │ │ Controller │                 │
│   │    etcd    │ │    etcd    │ │    etcd    │                 │
│   └────────────┘ └────────────┘ └────────────┘                 │
│                                                                  │
│   Only one scheduler and controller manager active              │
│   (leader election)                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Summary

### Control Plane Components

| Component | Function |
|-----------|----------|
| API Server | Front-end, handles all API requests |
| etcd | Stores all cluster data |
| Scheduler | Assigns Pods to Nodes |
| Controller Manager | Runs controllers (loops) |
| Cloud Controller | Cloud-specific logic |

### Worker Node Components

| Component | Function |
|-----------|----------|
| Kubelet | Node agent, manages Pods |
| Kube-proxy | Network proxy, implements Services |
| Container Runtime | Runs containers (containerd, CRI-O) |

---

**Previous:** [01-introduction.md](01-introduction.md) | **Next:** [03-installation.md](03-installation.md)
