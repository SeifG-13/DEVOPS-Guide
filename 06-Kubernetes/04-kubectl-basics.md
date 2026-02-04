# kubectl Basics

## What is kubectl?

kubectl is the command-line tool for interacting with Kubernetes clusters.

```
┌─────────────────────────────────────────────────────────────────┐
│                      kubectl Overview                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User ──► kubectl ──► API Server ──► Cluster                   │
│                                                                  │
│   Command Structure:                                            │
│   kubectl [command] [TYPE] [NAME] [flags]                       │
│                                                                  │
│   Examples:                                                     │
│   kubectl get pods                                              │
│   kubectl describe pod nginx                                    │
│   kubectl delete deployment myapp                               │
│   kubectl apply -f config.yaml                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Configuration

### kubeconfig File

```bash
# Default location
~/.kube/config

# View config
kubectl config view

# Use different config
kubectl --kubeconfig=/path/to/config get pods
export KUBECONFIG=/path/to/config
```

### Context Management

```bash
# List contexts
kubectl config get-contexts

# Current context
kubectl config current-context

# Switch context
kubectl config use-context my-cluster

# Set default namespace
kubectl config set-context --current --namespace=development

# Create new context
kubectl config set-context dev-context \
    --cluster=my-cluster \
    --user=dev-user \
    --namespace=development
```

## Basic Commands

### Get Resources

```bash
# Get all resources of a type
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get nodes

# Get from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Get with more details
kubectl get pods -o wide

# Get in different formats
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name

# Get specific resource
kubectl get pod nginx
kubectl get pod/nginx

# Get multiple types
kubectl get pods,services,deployments

# Watch for changes
kubectl get pods -w
kubectl get pods --watch
```

### Describe Resources

```bash
# Detailed information
kubectl describe pod nginx
kubectl describe node worker-1
kubectl describe service my-service

# Useful for debugging - shows events
kubectl describe pod nginx | tail -20
```

### Create Resources

```bash
# Create from file
kubectl create -f pod.yaml
kubectl create -f https://example.com/pod.yaml

# Create deployment imperatively
kubectl create deployment nginx --image=nginx

# Create service
kubectl create service clusterip nginx --tcp=80:80

# Create namespace
kubectl create namespace development

# Create secret
kubectl create secret generic my-secret --from-literal=key=value

# Create configmap
kubectl create configmap my-config --from-literal=key=value
```

### Apply Configuration

```bash
# Apply configuration (create or update)
kubectl apply -f deployment.yaml

# Apply multiple files
kubectl apply -f file1.yaml -f file2.yaml

# Apply all files in directory
kubectl apply -f ./configs/

# Apply from URL
kubectl apply -f https://example.com/deployment.yaml

# Dry run (client-side)
kubectl apply -f deployment.yaml --dry-run=client

# Dry run (server-side)
kubectl apply -f deployment.yaml --dry-run=server
```

### Delete Resources

```bash
# Delete by name
kubectl delete pod nginx
kubectl delete deployment myapp

# Delete from file
kubectl delete -f pod.yaml

# Delete all pods in namespace
kubectl delete pods --all

# Delete with label selector
kubectl delete pods -l app=myapp

# Force delete (stuck pods)
kubectl delete pod nginx --force --grace-period=0

# Delete namespace (and all resources in it)
kubectl delete namespace development
```

## Working with Pods

### Run Pods

```bash
# Run a pod
kubectl run nginx --image=nginx

# Run with port
kubectl run nginx --image=nginx --port=80

# Run and expose
kubectl run nginx --image=nginx --port=80 --expose

# Run interactive pod
kubectl run -it busybox --image=busybox -- sh

# Run one-time pod (job-like)
kubectl run test --image=busybox --restart=Never -- echo "Hello"

# Generate YAML without creating
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

### Execute Commands in Pods

```bash
# Execute command
kubectl exec nginx -- ls /
kubectl exec nginx -- cat /etc/nginx/nginx.conf

# Interactive shell
kubectl exec -it nginx -- /bin/bash
kubectl exec -it nginx -- sh

# Execute in specific container (multi-container pod)
kubectl exec -it nginx -c sidecar -- sh

# Copy files to/from pod
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
kubectl cp ./config.txt nginx:/app/config.txt
```

### View Logs

```bash
# View logs
kubectl logs nginx

# Follow logs
kubectl logs -f nginx

# Tail last N lines
kubectl logs nginx --tail=100

# Logs from specific container
kubectl logs nginx -c sidecar

# Logs from previous instance
kubectl logs nginx --previous

# Logs with timestamps
kubectl logs nginx --timestamps

# Logs from all pods with label
kubectl logs -l app=nginx

# Logs since time
kubectl logs nginx --since=1h
kubectl logs nginx --since-time=2024-01-01T00:00:00Z
```

### Port Forwarding

```bash
# Forward local port to pod
kubectl port-forward nginx 8080:80

# Forward to service
kubectl port-forward service/nginx 8080:80

# Forward to deployment
kubectl port-forward deployment/nginx 8080:80

# Listen on all interfaces
kubectl port-forward --address 0.0.0.0 nginx 8080:80
```

## Labels and Selectors

### Working with Labels

```bash
# Show labels
kubectl get pods --show-labels

# Filter by label
kubectl get pods -l app=nginx
kubectl get pods -l 'app=nginx,env=prod'
kubectl get pods -l 'app in (nginx,apache)'
kubectl get pods -l 'env!=prod'

# Add label
kubectl label pod nginx env=prod

# Update label (overwrite)
kubectl label pod nginx env=staging --overwrite

# Remove label
kubectl label pod nginx env-
```

### Annotations

```bash
# Add annotation
kubectl annotate pod nginx description="My nginx pod"

# Remove annotation
kubectl annotate pod nginx description-

# View annotations
kubectl get pod nginx -o jsonpath='{.metadata.annotations}'
```

## Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Get resources in namespace
kubectl get pods -n kube-system
kubectl get pods --namespace=development

# Get resources in all namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# Create namespace
kubectl create namespace development

# Set default namespace
kubectl config set-context --current --namespace=development

# Delete namespace
kubectl delete namespace development
```

## Resource Management

### Scale Resources

```bash
# Scale deployment
kubectl scale deployment nginx --replicas=5

# Scale multiple
kubectl scale deployment nginx apache --replicas=3

# Scale with condition
kubectl scale deployment nginx --replicas=3 --current-replicas=2
```

### Edit Resources

```bash
# Edit resource (opens in editor)
kubectl edit deployment nginx
kubectl edit pod nginx

# Set environment variable
KUBE_EDITOR="nano" kubectl edit deployment nginx
```

### Patch Resources

```bash
# Strategic merge patch
kubectl patch deployment nginx -p '{"spec":{"replicas":3}}'

# JSON patch
kubectl patch deployment nginx --type='json' \
    -p='[{"op":"replace","path":"/spec/replicas","value":3}]'

# Merge patch
kubectl patch deployment nginx --type='merge' \
    -p '{"spec":{"replicas":3}}'
```

## Output Formatting

### Common Formats

```bash
# Wide output
kubectl get pods -o wide

# YAML output
kubectl get pod nginx -o yaml

# JSON output
kubectl get pod nginx -o json

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pod nginx -o jsonpath='{.spec.containers[0].image}'

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
```

### Useful JSONPath Examples

```bash
# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Get images used by pods
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Get pod name and status
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

## Help and Documentation

```bash
# General help
kubectl --help
kubectl <command> --help

# Explain resource
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy

# API resources
kubectl api-resources
kubectl api-versions

# Verbose output
kubectl get pods -v=8  # Shows HTTP requests
```

## Shell Completion

```bash
# Bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# Zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
source ~/.zshrc

# Set alias with completion
alias k=kubectl
complete -F __start_kubectl k
```

## Quick Reference

### Most Common Commands

| Command | Description |
|---------|-------------|
| `kubectl get` | List resources |
| `kubectl describe` | Show detailed info |
| `kubectl create` | Create resource |
| `kubectl apply` | Apply configuration |
| `kubectl delete` | Delete resource |
| `kubectl logs` | View logs |
| `kubectl exec` | Execute command |
| `kubectl port-forward` | Forward port |
| `kubectl scale` | Scale resource |

### Useful Shortcuts

```bash
# Short forms
kubectl get po          # pods
kubectl get svc         # services
kubectl get deploy      # deployments
kubectl get ns          # namespaces
kubectl get no          # nodes
kubectl get cm          # configmaps
kubectl get pv          # persistentvolumes
kubectl get pvc         # persistentvolumeclaims

# Quick pod creation
kubectl run test --image=busybox --rm -it -- sh
```

### Common Flags

| Flag | Description |
|------|-------------|
| `-n, --namespace` | Specify namespace |
| `-A, --all-namespaces` | All namespaces |
| `-o, --output` | Output format |
| `-l, --selector` | Label selector |
| `-f, --filename` | File/directory path |
| `-w, --watch` | Watch for changes |
| `--dry-run` | Don't create resource |
| `-v` | Verbosity level |

---

**Previous:** [03-installation.md](03-installation.md) | **Next:** [05-pods.md](05-pods.md)
