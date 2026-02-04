# Introduction to Kubernetes

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Container Orchestration                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without Orchestration          With Kubernetes                │
│   ┌─────────────────┐           ┌─────────────────┐            │
│   │ Manual Deploy   │           │ Automated Deploy │            │
│   │ Manual Scaling  │    ──►    │ Auto Scaling     │            │
│   │ Manual Recovery │           │ Self Healing     │            │
│   │ Manual Updates  │           │ Rolling Updates  │            │
│   └─────────────────┘           └─────────────────┘            │
│                                                                  │
│   Problems Kubernetes Solves:                                   │
│   • How to deploy containers across multiple hosts?             │
│   • How to scale containers based on load?                      │
│   • What happens when a container crashes?                      │
│   • How to update without downtime?                             │
│   • How to manage configuration and secrets?                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Why Kubernetes?

### Key Features

| Feature | Description |
|---------|-------------|
| **Container Orchestration** | Manage containers across multiple hosts |
| **Self-Healing** | Automatically restarts failed containers |
| **Auto-Scaling** | Scale based on CPU/memory or custom metrics |
| **Load Balancing** | Distribute traffic across containers |
| **Rolling Updates** | Update applications without downtime |
| **Service Discovery** | Built-in DNS for services |
| **Storage Orchestration** | Automatically mount storage systems |
| **Secret Management** | Manage sensitive information securely |

### Kubernetes vs Docker

```
┌─────────────────────────────────────────────────────────────────┐
│                  Docker vs Kubernetes                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Docker                          Kubernetes                    │
│   ├── Container runtime           ├── Container orchestrator    │
│   ├── Build images                ├── Manage containers         │
│   ├── Run containers              ├── Scale applications        │
│   ├── Single host                 ├── Multi-host cluster        │
│   └── docker-compose              └── Production-grade          │
│                                                                  │
│   Docker = Building & Running Containers                        │
│   Kubernetes = Managing Containers at Scale                     │
│                                                                  │
│   They work together:                                           │
│   Docker builds images → Kubernetes runs them in production    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Kubernetes History

```
Timeline:
─────────────────────────────────────────────────────────────────►

2003-2004     2013         2014          2015         Present
    │           │            │             │             │
    ▼           ▼            ▼             ▼             ▼
  Borg      Docker       Kubernetes    CNCF          Dominant
 (Google)   Released     Open Source   Donation      Platform

• Borg: Google's internal container orchestration (15+ years)
• Kubernetes: Inspired by Borg, open-sourced by Google
• CNCF: Cloud Native Computing Foundation maintains K8s
• Today: De facto standard for container orchestration
```

## Core Concepts Overview

### Cluster

A Kubernetes cluster consists of:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Control Plane                         │   │
│   │  (Master - manages the cluster)                         │   │
│   │  • API Server    • Scheduler                            │   │
│   │  • etcd          • Controller Manager                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│   │  Worker Node │  │  Worker Node │  │  Worker Node │         │
│   │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │         │
│   │  │  Pod   │  │  │  │  Pod   │  │  │  │  Pod   │  │         │
│   │  │  Pod   │  │  │  │  Pod   │  │  │  │  Pod   │  │         │
│   │  └────────┘  │  │  └────────┘  │  │  └────────┘  │         │
│   └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Resources

| Resource | Description |
|----------|-------------|
| **Pod** | Smallest deployable unit, one or more containers |
| **Deployment** | Manages ReplicaSets and provides updates |
| **Service** | Exposes Pods to network traffic |
| **ConfigMap** | Configuration data (non-sensitive) |
| **Secret** | Sensitive data (passwords, tokens) |
| **Namespace** | Virtual clusters for resource isolation |
| **Volume** | Storage for Pods |

### Basic Workflow

```
1. Define desired state in YAML
   ─────────────────────────────►

2. Apply to cluster
   kubectl apply -f deployment.yaml
   ─────────────────────────────►

3. Kubernetes maintains desired state
   ┌─────────────────────────────────────┐
   │   Desired: 3 replicas               │
   │   Current: 2 replicas               │
   │   Action: Create 1 more Pod         │
   └─────────────────────────────────────┘

4. Self-healing
   Pod crashes → Kubernetes restarts it
```

## Kubernetes Distributions

### Local Development

| Distribution | Description |
|-------------|-------------|
| **Minikube** | Single-node cluster, most popular for learning |
| **Kind** | Kubernetes in Docker, lightweight |
| **Docker Desktop** | Built-in Kubernetes option |
| **k3d** | k3s in Docker |
| **MicroK8s** | Canonical's lightweight Kubernetes |

### Production

| Distribution | Description |
|-------------|-------------|
| **kubeadm** | Official tool for bootstrapping clusters |
| **k3s** | Lightweight Kubernetes for edge/IoT |
| **Rancher** | Multi-cluster management platform |
| **OpenShift** | Red Hat's enterprise Kubernetes |

### Managed Services

| Service | Provider |
|---------|----------|
| **EKS** | Amazon Web Services |
| **GKE** | Google Cloud Platform |
| **AKS** | Microsoft Azure |
| **DOKS** | DigitalOcean |

## Declarative vs Imperative

```
┌─────────────────────────────────────────────────────────────────┐
│              Declarative vs Imperative                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Imperative (How to do it):                                    │
│   ──────────────────────────                                    │
│   kubectl run nginx --image=nginx                               │
│   kubectl scale deployment nginx --replicas=3                   │
│   kubectl expose deployment nginx --port=80                     │
│                                                                  │
│   Declarative (What you want):                                  │
│   ────────────────────────────                                  │
│   apiVersion: apps/v1                                           │
│   kind: Deployment                                              │
│   metadata:                                                     │
│     name: nginx                                                 │
│   spec:                                                         │
│     replicas: 3                                                 │
│     ...                                                         │
│                                                                  │
│   kubectl apply -f deployment.yaml                              │
│                                                                  │
│   ✓ Declarative is preferred for production                    │
│   ✓ Version control friendly                                   │
│   ✓ Reproducible                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## YAML Structure

All Kubernetes resources follow this structure:

```yaml
apiVersion: v1          # API version
kind: Pod               # Resource type
metadata:               # Metadata
  name: my-pod
  labels:
    app: myapp
spec:                   # Specification (desired state)
  containers:
  - name: nginx
    image: nginx:latest
```

### Common API Versions

| API Version | Resources |
|-------------|-----------|
| `v1` | Pod, Service, ConfigMap, Secret, Namespace |
| `apps/v1` | Deployment, ReplicaSet, StatefulSet, DaemonSet |
| `batch/v1` | Job, CronJob |
| `networking.k8s.io/v1` | Ingress, NetworkPolicy |
| `rbac.authorization.k8s.io/v1` | Role, ClusterRole, RoleBinding |

## Quick Start Example

```yaml
# nginx-deployment.yaml
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
# Apply configuration
kubectl apply -f nginx-deployment.yaml

# Check deployment
kubectl get deployments
kubectl get pods
kubectl get services

# Access the application
kubectl port-forward service/nginx-service 8080:80
# Open http://localhost:8080
```

## Summary

| Concept | Description |
|---------|-------------|
| Kubernetes | Container orchestration platform |
| Cluster | Set of nodes running containerized applications |
| Control Plane | Manages the cluster |
| Worker Node | Runs application workloads |
| Pod | Smallest deployable unit |
| Declarative | Define desired state in YAML |

---

**Next:** [02-architecture.md](02-architecture.md)
