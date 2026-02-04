# Kubernetes RBAC

## What is RBAC?

Role-Based Access Control (RBAC) regulates access to Kubernetes resources.

```
┌─────────────────────────────────────────────────────────────────┐
│                      RBAC Concept                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Who (Subject)        What (Role)          Where (Binding)     │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│   │    User      │    │    Role      │    │ RoleBinding  │     │
│   │    Group     │◄───│  ClusterRole │◄───│ClusterRole-  │     │
│   │ServiceAccount│    │              │    │  Binding     │     │
│   └──────────────┘    └──────────────┘    └──────────────┘     │
│                                                                  │
│   Role: Defines permissions (verbs on resources)               │
│   RoleBinding: Connects subjects to roles                       │
│                                                                  │
│   Namespace-scoped: Role + RoleBinding                          │
│   Cluster-scoped: ClusterRole + ClusterRoleBinding              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## RBAC Components

| Component | Scope | Description |
|-----------|-------|-------------|
| Role | Namespace | Permissions within namespace |
| ClusterRole | Cluster | Cluster-wide permissions |
| RoleBinding | Namespace | Binds Role/ClusterRole to subjects in namespace |
| ClusterRoleBinding | Cluster | Binds ClusterRole to subjects cluster-wide |

## Roles

### Create a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]  # "" = core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

### Role with Multiple Resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]
  verbs: ["get", "create"]
```

### Role with Specific Resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config", "db-config"]  # Specific resources
  verbs: ["get"]
```

## ClusterRoles

### Read-Only ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
```

### Admin ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### Non-Resource URLs

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: health-checker
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"]
  verbs: ["get"]
```

## RoleBindings

### Bind Role to User

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Bind to Group

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-binding
  namespace: development
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Bind to ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Bind ClusterRole to Namespace

```yaml
# Use ClusterRole permissions in specific namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-reader-binding
  namespace: development
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole  # ClusterRole used via RoleBinding
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

## ClusterRoleBindings

### Cluster-Wide Binding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
- kind: User
  name: admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Bind to Multiple Subjects

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: readers-binding
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: viewers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

## Service Accounts

### Create ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
```

### Use in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:latest
```

### ServiceAccount with Image Pull Secret

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
imagePullSecrets:
- name: docker-registry-secret
```

### Complete ServiceAccount Example

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployment-manager
  namespace: default
---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager-role
  namespace: default
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-manager-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: deployment-manager
  namespace: default
roleRef:
  kind: Role
  name: deployment-manager-role
  apiGroup: rbac.authorization.k8s.io
```

## Default ClusterRoles

| Role | Description |
|------|-------------|
| `cluster-admin` | Full cluster access |
| `admin` | Full namespace access |
| `edit` | Read/write namespace resources |
| `view` | Read-only namespace access |

```bash
# List default roles
kubectl get clusterroles | grep -E "^(cluster-admin|admin|edit|view)"

# Describe a role
kubectl describe clusterrole view
```

## Verbs Reference

| Verb | Description |
|------|-------------|
| `get` | Read a resource |
| `list` | List resources |
| `watch` | Watch for changes |
| `create` | Create resource |
| `update` | Update resource |
| `patch` | Partial update |
| `delete` | Delete resource |
| `deletecollection` | Delete multiple |

## API Groups

| Group | Resources |
|-------|-----------|
| `""` (core) | pods, services, configmaps, secrets, namespaces |
| `apps` | deployments, replicasets, statefulsets, daemonsets |
| `batch` | jobs, cronjobs |
| `networking.k8s.io` | ingresses, networkpolicies |
| `rbac.authorization.k8s.io` | roles, rolebindings |

```bash
# List API groups
kubectl api-resources --api-group=apps
```

## Managing RBAC

```bash
# Create role
kubectl create role pod-reader --verb=get,list,watch --resource=pods

# Create clusterrole
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes

# Create rolebinding
kubectl create rolebinding read-pods --role=pod-reader --user=jane

# Create clusterrolebinding
kubectl create clusterrolebinding read-nodes --clusterrole=node-reader --user=jane

# Check permissions
kubectl auth can-i create pods
kubectl auth can-i create pods --as=jane
kubectl auth can-i create pods --as=jane -n development
kubectl auth can-i --list --as=jane

# List roles
kubectl get roles
kubectl get clusterroles

# Describe
kubectl describe role pod-reader
kubectl describe rolebinding read-pods
```

## Testing Permissions

```bash
# Test as user
kubectl auth can-i get pods --as=jane
kubectl auth can-i delete pods --as=jane
kubectl auth can-i create deployments --as=jane -n development

# List all permissions
kubectl auth can-i --list --as=jane -n default

# Test as ServiceAccount
kubectl auth can-i get pods --as=system:serviceaccount:default:myapp-sa
```

## Quick Reference

### RBAC Resources

| Resource | Scope | Purpose |
|----------|-------|---------|
| Role | Namespace | Define permissions |
| ClusterRole | Cluster | Define cluster-wide permissions |
| RoleBinding | Namespace | Bind role to subjects |
| ClusterRoleBinding | Cluster | Bind clusterrole to subjects |

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl create role` | Create role |
| `kubectl create rolebinding` | Create binding |
| `kubectl auth can-i` | Test permissions |
| `kubectl get roles` | List roles |

### Subject Types

| Type | Example |
|------|---------|
| User | `jane` |
| Group | `developers` |
| ServiceAccount | `system:serviceaccount:default:myapp` |

---

**Previous:** [17-networking.md](17-networking.md) | **Next:** [19-resource-management.md](19-resource-management.md)
