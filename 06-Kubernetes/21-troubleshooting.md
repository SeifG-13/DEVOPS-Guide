# Kubernetes Troubleshooting

## Troubleshooting Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                 Troubleshooting Workflow                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Identify the problem                                       │
│      └── kubectl get pods/svc/deploy                           │
│                                                                  │
│   2. Gather information                                         │
│      └── kubectl describe <resource>                           │
│      └── kubectl logs <pod>                                    │
│      └── kubectl get events                                    │
│                                                                  │
│   3. Debug                                                      │
│      └── kubectl exec -it <pod> -- sh                         │
│      └── kubectl debug <pod>                                   │
│                                                                  │
│   4. Fix and verify                                            │
│      └── Apply fix                                             │
│      └── Monitor with kubectl get -w                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Pod Troubleshooting

### Common Pod States

| State | Meaning | Common Causes |
|-------|---------|---------------|
| `Pending` | Not scheduled | Resource issues, node selector |
| `ContainerCreating` | Setting up | Image pull, volume mount |
| `Running` | Container running | - |
| `CrashLoopBackOff` | Crashing repeatedly | App error, missing config |
| `ImagePullBackOff` | Can't pull image | Wrong image, no auth |
| `Error` | Container failed | App error |
| `Completed` | Finished (Jobs) | - |
| `Terminating` | Being deleted | Stuck finalizers |

### Debugging Pending Pods

```bash
# Check why pod is pending
kubectl describe pod <pod-name>

# Common issues:
# - Insufficient resources
# - Node selector not matching
# - Taint not tolerated
# - PVC not bound

# Check events
kubectl get events --sort-by='.lastTimestamp' | grep <pod-name>

# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes
```

### Debugging CrashLoopBackOff

```bash
# Check pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container instance

# Check container exit code
kubectl describe pod <pod-name> | grep -A 10 "Last State"

# Common exit codes:
# 0   - Success (unexpected for long-running)
# 1   - Application error
# 137 - OOMKilled (128 + 9)
# 143 - SIGTERM (128 + 15)

# Debug with shell
kubectl exec -it <pod-name> -- /bin/sh

# Check if OOMKilled
kubectl describe pod <pod-name> | grep -i oom
```

### Debugging ImagePullBackOff

```bash
# Check image name
kubectl describe pod <pod-name> | grep "Image:"

# Check pull errors
kubectl describe pod <pod-name> | grep -A 5 "Events"

# Common issues:
# - Wrong image name/tag
# - Private registry without imagePullSecret
# - Network issues

# Test image pull
kubectl run test --image=<image> --restart=Never

# Check imagePullSecrets
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'
```

## Service Troubleshooting

### Service Not Working

```bash
# Check service
kubectl get svc <service-name>
kubectl describe svc <service-name>

# Check endpoints (should have pod IPs)
kubectl get endpoints <service-name>

# If no endpoints:
# - Selector doesn't match pod labels
# - No pods running
# - Pods not ready

# Check pod labels
kubectl get pods --show-labels

# Test service from within cluster
kubectl run test --rm -it --image=busybox -- wget -qO- http://<service-name>:<port>

# Check service DNS
kubectl run test --rm -it --image=busybox -- nslookup <service-name>
```

### Debugging Service Connection

```bash
# 1. Check if service exists
kubectl get svc <service-name>

# 2. Check endpoints
kubectl get endpoints <service-name>

# 3. Check if pods are running and ready
kubectl get pods -l <selector>

# 4. Test pod connectivity
kubectl exec -it <client-pod> -- curl <service-ip>:<port>

# 5. Test DNS
kubectl exec -it <client-pod> -- nslookup <service-name>

# 6. Check network policies
kubectl get networkpolicies
```

## Deployment Troubleshooting

### Deployment Not Progressing

```bash
# Check deployment status
kubectl get deployment <name>
kubectl describe deployment <name>

# Check rollout status
kubectl rollout status deployment/<name>

# Check ReplicaSet
kubectl get rs -l app=<name>
kubectl describe rs <replicaset-name>

# Common issues:
# - Image pull failures
# - Insufficient resources
# - Probe failures
# - Resource quota exceeded

# Check events
kubectl get events --field-selector involvedObject.name=<deployment-name>
```

### Rolling Update Stuck

```bash
# Check deployment conditions
kubectl describe deployment <name> | grep Conditions -A 5

# Check new ReplicaSet pods
kubectl get pods -l app=<name>

# Rollback if needed
kubectl rollout undo deployment/<name>

# View rollout history
kubectl rollout history deployment/<name>
```

## Node Troubleshooting

### Node Not Ready

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check node conditions
kubectl get node <node-name> -o jsonpath='{.status.conditions}' | jq

# Common conditions:
# - Ready: Node is healthy
# - MemoryPressure: Running out of memory
# - DiskPressure: Running out of disk
# - PIDPressure: Too many processes
# - NetworkUnavailable: Network not configured

# Check kubelet logs (on node)
journalctl -u kubelet -f

# Check system resources (on node)
top
df -h
free -m
```

### Node Resource Issues

```bash
# Check resource usage
kubectl top nodes

# Check pods on node
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node>

# Check allocatable resources
kubectl describe node <node> | grep -A 10 "Allocated resources"

# Cordon node (prevent new pods)
kubectl cordon <node>

# Drain node (evict pods)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

## Networking Troubleshooting

### DNS Issues

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS resolution
kubectl run test --rm -it --image=busybox -- nslookup kubernetes.default

# Check DNS configuration in pod
kubectl exec <pod> -- cat /etc/resolv.conf
```

### Network Connectivity

```bash
# Debug pod with network tools
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Inside pod:
ping <pod-ip>
curl <service-name>:<port>
traceroute <target>
nslookup <service-name>
tcpdump -i any port 80

# Check network policies
kubectl get networkpolicies -A
kubectl describe networkpolicy <name>
```

## Storage Troubleshooting

### PVC Pending

```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc <pvc-name>

# Common issues:
# - No matching PV
# - StorageClass not found
# - Insufficient storage

# Check PVs
kubectl get pv

# Check StorageClass
kubectl get storageclass

# Check events
kubectl get events | grep <pvc-name>
```

### Volume Mount Issues

```bash
# Check pod events for mount errors
kubectl describe pod <pod> | grep -A 20 Events

# Common issues:
# - Permission denied
# - Volume not found
# - Read-only filesystem

# Check volume mounts in pod spec
kubectl get pod <pod> -o jsonpath='{.spec.volumes}'
```

## Useful Debugging Commands

### General Commands

```bash
# Get all resources in namespace
kubectl get all -n <namespace>

# Get events sorted by time
kubectl get events --sort-by='.lastTimestamp'

# Watch resources
kubectl get pods -w

# Get resource YAML
kubectl get pod <name> -o yaml

# Explain resource fields
kubectl explain pod.spec.containers
```

### Debug Containers

```bash
# Ephemeral debug container (K8s 1.23+)
kubectl debug -it <pod> --image=busybox --target=<container>

# Copy debugging tools to running container
kubectl debug <pod> -it --copy-to=debug-pod --image=busybox

# Debug node
kubectl debug node/<node> -it --image=ubuntu
```

### Logs and Events

```bash
# Pod logs
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> --previous
kubectl logs <pod> -f  # Follow
kubectl logs <pod> --tail=100
kubectl logs <pod> --since=1h

# All pod logs with label
kubectl logs -l app=myapp --all-containers

# Events
kubectl get events -n <namespace>
kubectl get events --field-selector type=Warning
```

## Common Issues and Solutions

```
┌─────────────────────────────────────────────────────────────────┐
│              Common Issues Quick Reference                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Issue                      Solution                           │
│   ─────────────────────────────────────────────────────         │
│   Pod Pending                Check resources, node selector     │
│   ImagePullBackOff           Verify image name, check secrets   │
│   CrashLoopBackOff           Check logs, verify config          │
│   Service no endpoints       Check selector labels              │
│   PVC Pending                Check StorageClass, PV             │
│   Node NotReady              Check kubelet, system resources    │
│   DNS not working            Check CoreDNS pods                 │
│   Cannot exec into pod       Container might not have shell     │
│   Deployment stuck           Check resource quota, probe        │
│   OOMKilled                  Increase memory limits             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Troubleshooting Tools

```bash
# Install useful tools in debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash

# Tools available:
# - curl, wget
# - dig, nslookup
# - ping, traceroute
# - tcpdump, netstat
# - iperf, mtr
# - jq, vim
```

## Quick Reference

### Debugging Commands

| Command | Purpose |
|---------|---------|
| `kubectl describe` | Detailed resource info |
| `kubectl logs` | Container logs |
| `kubectl exec` | Run command in container |
| `kubectl debug` | Create debug container |
| `kubectl get events` | Cluster events |
| `kubectl top` | Resource usage |

### Pod States

| State | Action |
|-------|--------|
| Pending | Check resources, selectors |
| CrashLoopBackOff | Check logs, config |
| ImagePullBackOff | Verify image, secrets |
| Error | Check logs |
| Terminating | Check finalizers |

---

**Previous:** [20-health-checks.md](20-health-checks.md) | **Next:** [22-best-practices.md](22-best-practices.md)
