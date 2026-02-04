# Kubernetes Secrets

## What is a Secret?

Secrets store sensitive data like passwords, tokens, and keys.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Secret Concept                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Secret (db-credentials)                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  type: Opaque                                            │   │
│   │  data:                                                   │   │
│   │    username: YWRtaW4=        (base64: admin)            │   │
│   │    password: cGFzc3dvcmQ=    (base64: password)         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│              ┌─────────────┼─────────────┐                      │
│              ▼             ▼             ▼                      │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│   │  Env Vars    │ │ Volume Mount │ │  imagePull   │           │
│   │  in Pod      │ │  as Files    │ │  Secret      │           │
│   └──────────────┘ └──────────────┘ └──────────────┘           │
│                                                                  │
│   Important:                                                    │
│   • Base64 encoded, NOT encrypted by default                   │
│   • Enable encryption at rest for security                      │
│   • Use RBAC to limit access                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Secret Types

| Type | Usage |
|------|-------|
| `Opaque` | Generic secret (default) |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH authentication |
| `kubernetes.io/tls` | TLS certificate |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/service-account-token` | Service account token |

## Creating Secrets

### Generic/Opaque Secrets

```bash
# From literals
kubectl create secret generic db-credentials \
    --from-literal=username=admin \
    --from-literal=password=secretpass123

# From files
kubectl create secret generic ssh-keys \
    --from-file=ssh-privatekey=~/.ssh/id_rsa \
    --from-file=ssh-publickey=~/.ssh/id_rsa.pub

# From directory
kubectl create secret generic app-secrets --from-file=./secrets/
```

### Using YAML

```yaml
# Values must be base64 encoded
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # echo -n 'admin' | base64
  password: cGFzc3dvcmQxMjM=  # echo -n 'password123' | base64
```

### Using stringData (Plain Text)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # Plain text, automatically encoded
  username: admin
  password: password123
  config.yaml: |
    database:
      host: mysql.default
      port: 3306
```

### Docker Registry Secret

```bash
# Create docker registry secret
kubectl create secret docker-registry regcred \
    --docker-server=https://index.docker.io/v1/ \
    --docker-username=myuser \
    --docker-password=mypassword \
    --docker-email=myemail@example.com
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### TLS Secret

```bash
# Create TLS secret from certificate files
kubectl create secret tls tls-secret \
    --cert=path/to/tls.crt \
    --key=path/to/tls.key
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

### Basic Auth Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: secretpassword
```

## Using Secrets in Pods

### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Single key from Secret
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### All Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - secretRef:
        name: db-credentials
    # With prefix
    - secretRef:
        name: db-credentials
        prefix: DB_
```

### As Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

Files created:
- `/etc/secrets/username`
- `/etc/secrets/password`

### Mount Specific Keys

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      items:
      - key: username
        path: db-user
      - key: password
        path: db-pass
        mode: 0400  # File permissions
```

### Image Pull Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
  - name: app
    image: private-registry.io/myapp:v1
  imagePullSecrets:
  - name: regcred
```

### TLS Secret with Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

## Secrets with Deployments

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  database-url: "postgresql://user:pass@db:5432/mydb"
  api-key: "sk-xxxxxxxxxxxx"
  jwt-secret: "my-super-secret-jwt-key"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: webapp:v1
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
        volumeMounts:
        - name: jwt-secret
          mountPath: /etc/secrets/jwt
          readOnly: true
      volumes:
      - name: jwt-secret
        secret:
          secretName: app-secrets
          items:
          - key: jwt-secret
            path: secret.key
```

## Managing Secrets

### View Secrets

```bash
# List secrets
kubectl get secrets

# Describe secret (doesn't show values)
kubectl describe secret db-credentials

# Get secret YAML (shows base64 encoded values)
kubectl get secret db-credentials -o yaml

# Decode secret value
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode
```

### Update Secrets

```bash
# Edit secret
kubectl edit secret db-credentials

# Replace secret
kubectl create secret generic db-credentials \
    --from-literal=username=newuser \
    --from-literal=password=newpass \
    --dry-run=client -o yaml | kubectl apply -f -

# Patch secret
kubectl patch secret db-credentials -p '{"stringData":{"password":"newpassword"}}'
```

### Delete Secrets

```bash
kubectl delete secret db-credentials
kubectl delete -f secret.yaml
```

## Secret Security

### Encryption at Rest

```yaml
# EncryptionConfiguration for API server
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
    - identity: {}
```

### RBAC for Secrets

```yaml
# Role that can only read specific secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials"]  # Specific secrets only
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: default
subjects:
- kind: ServiceAccount
  name: myapp
  namespace: default
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Immutable Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
immutable: true  # Cannot be modified
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

## External Secret Management

### Using External Secrets Operator

```yaml
# ExternalSecret syncs from external store
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials
  data:
  - secretKey: username
    remoteRef:
      key: database/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: database/credentials
      property: password
```

### Using Sealed Secrets

```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Seal a secret (safe to commit to git)
kubeseal --format=yaml < secret.yaml > sealed-secret.yaml
```

```yaml
# Sealed secret (encrypted, safe for git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO...
    password: AgBy3i4OJSWK+PiTySYZZA9rO...
```

## Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                  Secret Best Practices                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ✓ Enable encryption at rest                                   │
│   ✓ Use RBAC to restrict secret access                          │
│   ✓ Avoid secrets in environment variables (prefer volumes)     │
│   ✓ Rotate secrets regularly                                    │
│   ✓ Use external secret managers for production                 │
│   ✓ Never commit secrets to git                                 │
│   ✓ Use sealed-secrets or external-secrets for GitOps          │
│   ✓ Mark secrets as immutable when possible                     │
│   ✓ Audit secret access                                         │
│                                                                  │
│   ✗ Don't store secrets in ConfigMaps                           │
│   ✗ Don't include secrets in container images                   │
│   ✗ Don't log secret values                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Reference

### Secret Types

| Type | Usage |
|------|-------|
| `Opaque` | Generic key-value pairs |
| `kubernetes.io/tls` | TLS certificates |
| `kubernetes.io/dockerconfigjson` | Docker registry auth |
| `kubernetes.io/basic-auth` | Basic authentication |

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl create secret generic` | Create generic secret |
| `kubectl create secret tls` | Create TLS secret |
| `kubectl create secret docker-registry` | Create registry secret |
| `kubectl get secrets` | List secrets |
| `kubectl describe secret` | Show secret details |
| `kubectl delete secret` | Delete secret |

### Key Points

- Secrets are base64 encoded, NOT encrypted by default
- Enable encryption at rest in production
- Use RBAC to control access
- Volume mounts are more secure than env vars
- Consider external secret management

---

**Previous:** [10-configmaps.md](10-configmaps.md) | **Next:** [12-volumes.md](12-volumes.md)
