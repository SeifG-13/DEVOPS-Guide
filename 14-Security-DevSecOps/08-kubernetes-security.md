# Kubernetes Security

## Kubernetes Security Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES SECURITY LAYERS                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Infrastructure │ Nodes, network, etcd encryption            │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ Cluster        │ RBAC, admission control, API security      │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ Namespace      │ Resource quotas, network policies          │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ Workload       │ Pod security, service accounts             │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ Container      │ Image security, runtime protection         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  The 4 C's of Cloud Native Security:                               │
│  Cloud → Cluster → Container → Code                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## RBAC (Role-Based Access Control)

### RBAC Components

```yaml
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

```yaml
# ClusterRole (cluster-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
    # Restrict to specific secrets
    resourceNames: ["my-secret"]
```

```yaml
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: my-service-account
    namespace: production
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RBAC BEST PRACTICES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Principle of Least Privilege:                                      │
│  • Grant minimum required permissions                              │
│  • Use namespace-scoped Roles over ClusterRoles                    │
│  • Avoid wildcards (*) in rules                                    │
│  • Use resourceNames to restrict to specific resources             │
│                                                                     │
│  Service Accounts:                                                  │
│  • Create dedicated SA per application                             │
│  • Don't use default service account                               │
│  • Disable auto-mounting of SA tokens                              │
│  • Rotate tokens regularly                                         │
│                                                                     │
│  Audit:                                                             │
│  • Review RBAC regularly                                           │
│  • Use kubectl auth can-i to test                                  │
│  • Enable audit logging                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Pod Security Standards

### Security Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│                    POD SECURITY STANDARDS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Privileged (most permissive)                                       │
│  • No restrictions                                                 │
│  • For system/infrastructure workloads                             │
│                                                                     │
│  Baseline (default recommendation)                                  │
│  • Prevents known privilege escalations                            │
│  • Blocks hostNetwork, hostPID, hostIPC                           │
│  • Restricts capabilities                                          │
│                                                                     │
│  Restricted (most secure)                                           │
│  • Heavily restricted                                              │
│  • Must run as non-root                                            │
│  • No privilege escalation                                         │
│  • Seccomp required                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Applying Pod Security

```yaml
# Namespace with restricted policy
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### Compliant Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: myapp:v1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: tmp
          mountPath: /tmp

  volumes:
    - name: tmp
      emptyDir: {}

  automountServiceAccountToken: false
```

---

## Network Policies

### Default Deny All

```yaml
# Deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress

---
# Deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
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
        - port: 8080

    # Allow from ingress controller namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8080

  egress:
    # Allow to database
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - port: 5432

    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
```

---

## OPA Gatekeeper

### Installation

```bash
# Install Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper.yaml
```

### Constraint Template

```yaml
# Template: Require labels
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
```

### Constraint

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    labels:
      - "team"
      - "app"
```

### Common Policies

```yaml
# Block privileged containers
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivileged

        violation[{"msg": msg}] {
          c := input.review.object.spec.containers[_]
          c.securityContext.privileged
          msg := sprintf("Privileged container not allowed: %v", [c.name])
        }
```

---

## Secrets Management

### Kubernetes Secrets Best Practices

```yaml
# Encrypted at rest (cluster config)
# kube-apiserver configuration
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

```yaml
# Limit secret access with RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["my-secret"]  # Specific secret only
    verbs: ["get"]
```

---

## Audit Logging

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests to secrets
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]

  # Log authentication failures
  - level: Metadata
    users: ["system:anonymous"]
    verbs: ["*"]

  # Log changes to RBAC
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["*"]

  # Default: log metadata only
  - level: Metadata
```

---

## Security Scanning Tools

### Kubescape

```bash
# Install
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash

# Scan cluster
kubescape scan framework nsa

# Scan specific namespace
kubescape scan framework nsa --include-namespaces production

# Scan YAML files
kubescape scan framework nsa *.yaml

# Output formats
kubescape scan framework nsa --format json -o results.json
```

### Kube-bench

```bash
# Run CIS benchmark
docker run --pid=host -v /etc:/etc:ro \
  -v /var:/var:ro \
  aquasec/kube-bench:latest run --targets=node

# Or as job
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -l app=kube-bench
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Key Security Controls:                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ RBAC             │ Control who can do what                  │   │
│  │ Pod Security     │ Restrict container capabilities          │   │
│  │ Network Policies │ Control pod-to-pod communication         │   │
│  │ OPA Gatekeeper   │ Policy enforcement at admission          │   │
│  │ Secrets          │ Encrypt at rest, limit access            │   │
│  │ Audit Logging    │ Track all API activity                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Security Tools:                                                    │
│  • Kubescape - Framework compliance scanning                       │
│  • Kube-bench - CIS benchmark testing                              │
│  • Trivy - Misconfiguration scanning                               │
│  • Falco - Runtime threat detection                                │
│                                                                     │
│  Next: Learn about secrets management in depth                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
