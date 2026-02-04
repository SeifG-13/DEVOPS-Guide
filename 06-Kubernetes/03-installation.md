# Kubernetes Installation

## Installation Options

```
┌─────────────────────────────────────────────────────────────────┐
│                  Kubernetes Installation Options                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Local Development                                             │
│   ├── Minikube      (most popular for learning)                │
│   ├── Kind          (Kubernetes in Docker)                      │
│   ├── Docker Desktop (built-in option)                         │
│   ├── k3d           (k3s in Docker)                            │
│   └── MicroK8s      (Ubuntu/Canonical)                         │
│                                                                  │
│   Production Self-Managed                                       │
│   ├── kubeadm       (official tool)                            │
│   ├── k3s           (lightweight)                              │
│   └── Rancher       (multi-cluster)                            │
│                                                                  │
│   Managed Cloud Services                                        │
│   ├── EKS           (AWS)                                      │
│   ├── GKE           (Google Cloud)                             │
│   ├── AKS           (Azure)                                    │
│   └── DOKS          (DigitalOcean)                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Minikube Installation

### Install on Linux

```bash
# Download minikube binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify installation
minikube version
```

### Install on macOS

```bash
# Using Homebrew
brew install minikube

# Or download binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

### Install on Windows

```powershell
# Using Chocolatey
choco install minikube

# Using winget
winget install minikube

# Or download installer from:
# https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe
```

### Start Minikube Cluster

```bash
# Start with default driver
minikube start

# Start with specific driver
minikube start --driver=docker
minikube start --driver=virtualbox
minikube start --driver=hyperv

# Start with specific resources
minikube start --cpus=4 --memory=8192

# Start with specific Kubernetes version
minikube start --kubernetes-version=v1.28.0

# Start multi-node cluster
minikube start --nodes 3
```

### Common Minikube Commands

```bash
# Cluster status
minikube status

# Stop cluster
minikube stop

# Delete cluster
minikube delete

# SSH into node
minikube ssh

# Get cluster IP
minikube ip

# Open Kubernetes dashboard
minikube dashboard

# Enable addons
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server

# Access services
minikube service <service-name>
minikube service <service-name> --url
```

## Kind (Kubernetes in Docker)

### Install Kind

```bash
# Linux/macOS
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# macOS with Homebrew
brew install kind

# Windows with Chocolatey
choco install kind
```

### Create Kind Cluster

```bash
# Create simple cluster
kind create cluster

# Create named cluster
kind create cluster --name my-cluster

# Create with config file
kind create cluster --config kind-config.yaml
```

### Kind Configuration

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
```

### Kind Commands

```bash
# List clusters
kind get clusters

# Delete cluster
kind delete cluster --name my-cluster

# Load local image into cluster
kind load docker-image my-app:latest --name my-cluster

# Get kubeconfig
kind get kubeconfig --name my-cluster
```

## Docker Desktop Kubernetes

### Enable Kubernetes

```
1. Open Docker Desktop
2. Go to Settings/Preferences
3. Select "Kubernetes" tab
4. Check "Enable Kubernetes"
5. Click "Apply & Restart"
```

```bash
# Verify installation
kubectl cluster-info
kubectl get nodes
```

## kubectl Installation

kubectl is the command-line tool for Kubernetes.

### Install on Linux

```bash
# Download latest version
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

### Install on macOS

```bash
# Using Homebrew
brew install kubectl

# Or download binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

### Install on Windows

```powershell
# Using Chocolatey
choco install kubernetes-cli

# Using winget
winget install -e --id Kubernetes.kubectl

# Download binary
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"
# Add to PATH
```

### Configure kubectl

```bash
# View config
kubectl config view

# Get current context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context minikube

# Set namespace for context
kubectl config set-context --current --namespace=my-namespace
```

## kubeadm Installation (Production)

### Prerequisites

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### Install Container Runtime (containerd)

```bash
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Install kubeadm, kubelet, kubectl

```bash
# Add Kubernetes repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install packages
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize Control Plane

```bash
# Initialize cluster (on control plane node)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl for current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install Pod Network (Calico)

```bash
# Install Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Or Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Join Worker Nodes

```bash
# On control plane, get join command
kubeadm token create --print-join-command

# On worker nodes, run the join command
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

## k3s Installation (Lightweight)

### Install k3s Server

```bash
# Install server (control plane)
curl -sfL https://get.k3s.io | sh -

# Check status
sudo systemctl status k3s

# Get kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml

# Get node token (for workers)
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Join Worker Nodes

```bash
# On worker nodes
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -
```

### k3s with External Database

```bash
# MySQL
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="mysql://user:pass@tcp(host:3306)/k3s"

# PostgreSQL
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="postgres://user:pass@host:5432/k3s"
```

## Verify Installation

```bash
# Check cluster info
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check component status
kubectl get componentstatuses

# Run test pod
kubectl run test --image=nginx --restart=Never
kubectl get pod test
kubectl delete pod test
```

## Troubleshooting Installation

### Common Issues

```bash
# kubelet not starting
journalctl -xeu kubelet

# Check container runtime
crictl info

# Network issues
kubectl get pods -n kube-system
kubectl describe pod <coredns-pod> -n kube-system

# Certificate issues
kubeadm certs check-expiration

# Reset and start over
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
```

### Useful Debugging Commands

```bash
# Check kubelet logs
journalctl -u kubelet -f

# Check container runtime
systemctl status containerd
crictl ps -a

# Check kube-system components
kubectl get events -n kube-system
kubectl logs -n kube-system <pod-name>
```

## Quick Reference

| Tool | Best For |
|------|----------|
| Minikube | Learning, single-node development |
| Kind | CI/CD, testing, multi-node local |
| Docker Desktop | Simple local development |
| kubeadm | Production self-managed |
| k3s | Edge, IoT, lightweight production |
| EKS/GKE/AKS | Production managed services |

| Command | Description |
|---------|-------------|
| `minikube start` | Start Minikube cluster |
| `kind create cluster` | Create Kind cluster |
| `kubeadm init` | Initialize cluster |
| `kubectl cluster-info` | Verify cluster |
| `kubectl get nodes` | List nodes |

---

**Previous:** [02-architecture.md](02-architecture.md) | **Next:** [04-kubectl-basics.md](04-kubectl-basics.md)
