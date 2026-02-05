# ArgoCD Secrets Management

## The GitOps Secrets Challenge

Storing secrets in Git violates security best practices, but GitOps requires everything in Git.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THE SECRETS PROBLEM                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  GitOps Principle: Everything in Git                               │
│  Security Principle: Never store secrets in Git                    │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │     Problem     │                                               │
│  │                 │                                               │
│  │  apiVersion: v1 │                                               │
│  │  kind: Secret   │    ← Can't commit this to Git!               │
│  │  data:          │                                               │
│  │    password: xxx│                                               │
│  └─────────────────┘                                               │
│                                                                     │
│  Solutions:                                                         │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐      │
│  │ Sealed Secrets  │ │      SOPS       │ │External Secrets │      │
│  │                 │ │                 │ │                 │      │
│  │ Encrypt in Git  │ │ Encrypt in Git  │ │ Fetch at runtime│      │
│  │ Decrypt in K8s  │ │ Decrypt in CD   │ │ From vault      │      │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Solution 1: Sealed Secrets

Bitnami Sealed Secrets encrypts secrets that can only be decrypted by the cluster.

### How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SEALED SECRETS FLOW                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Install controller in cluster (holds private key)              │
│  2. Fetch public key                                               │
│  3. Encrypt secrets with kubeseal CLI                              │
│  4. Commit SealedSecret to Git                                     │
│  5. ArgoCD syncs SealedSecret                                      │
│  6. Controller decrypts to regular Secret                          │
│                                                                     │
│  ┌─────────────┐    kubeseal    ┌─────────────┐    Git            │
│  │   Secret    │───────────────▶│SealedSecret │──────────▶ Git Repo│
│  │  (plain)    │   (public key) │ (encrypted) │                    │
│  └─────────────┘                └─────────────┘                    │
│                                        │                            │
│                                        │ ArgoCD Sync                │
│                                        ▼                            │
│                                 ┌─────────────┐                    │
│                                 │  Sealed     │                    │
│                                 │  Secrets    │                    │
│                                 │ Controller  │                    │
│                                 └──────┬──────┘                    │
│                                        │ decrypt                    │
│                                        ▼                            │
│                                 ┌─────────────┐                    │
│                                 │   Secret    │                    │
│                                 │  (plain)    │                    │
│                                 └─────────────┘                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Install Sealed Secrets

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Install kubeseal CLI (macOS)
brew install kubeseal

# Install kubeseal CLI (Linux)
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.24.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

### Create Sealed Secret

```bash
# Create regular secret (don't commit this!)
kubectl create secret generic my-secret \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml > secret.yaml

# Seal the secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# Now safe to commit sealed-secret.yaml to Git
```

### SealedSecret Example

```yaml
# sealed-secret.yaml (safe to commit)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: default
spec:
  encryptedData:
    password: AgBy8BZ4p...encrypted...data
  template:
    metadata:
      name: my-secret
      namespace: default
    type: Opaque
```

### Scopes

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SEALED SECRETS SCOPES                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  strict (default):                                                  │
│  • Secret can only be unsealed in exact namespace with exact name  │
│  kubeseal --scope strict                                           │
│                                                                     │
│  namespace-wide:                                                    │
│  • Secret can be unsealed with any name in same namespace          │
│  kubeseal --scope namespace-wide                                   │
│                                                                     │
│  cluster-wide:                                                      │
│  • Secret can be unsealed anywhere in cluster                      │
│  kubeseal --scope cluster-wide                                     │
│                                                                     │
│  Recommendation: Use strict scope for security                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Solution 2: SOPS (Secrets OPerationS)

Mozilla SOPS encrypts files using cloud KMS, PGP, or age.

### How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SOPS FLOW                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    sops encrypt    ┌─────────────┐               │
│  │   secret    │───────────────────▶│  encrypted  │───▶ Git       │
│  │   .yaml     │    (AWS KMS/GCP/   │   .yaml     │               │
│  └─────────────┘     age/PGP)       └─────────────┘               │
│                                            │                        │
│                                            │ ArgoCD                 │
│                                            ▼                        │
│                                    ┌───────────────┐               │
│                                    │ KSOPS Plugin  │               │
│                                    │ (decrypt)     │               │
│                                    └───────┬───────┘               │
│                                            │                        │
│                                            ▼                        │
│                                    ┌─────────────┐                 │
│                                    │   Secret    │                 │
│                                    │  (plain)    │                 │
│                                    └─────────────┘                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Install SOPS

```bash
# macOS
brew install sops

# Linux
wget https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64
sudo mv sops-v3.8.1.linux.amd64 /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops
```

### Configure SOPS with age

```bash
# Generate age key pair
age-keygen -o keys.txt
# Public key: age1...
# Private key is in keys.txt

# Create .sops.yaml configuration
cat > .sops.yaml << EOF
creation_rules:
  - path_regex: .*\.enc\.yaml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
EOF

# Encrypt a file
sops --encrypt secret.yaml > secret.enc.yaml

# Decrypt a file
sops --decrypt secret.enc.yaml
```

### SOPS with AWS KMS

```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/.*\.yaml$
    kms: 'arn:aws:kms:us-east-1:123456789:key/abc-123-def'
```

### KSOPS for ArgoCD

```bash
# Install KSOPS as ArgoCD plugin
# Add to argocd-repo-server deployment

# kustomization.yaml using KSOPS
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

generators:
  - secret-generator.yaml
```

```yaml
# secret-generator.yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: secret-generator
files:
  - ./secrets/database.enc.yaml
  - ./secrets/api-keys.enc.yaml
```

### ArgoCD SOPS Plugin Configuration

```yaml
# Add to argocd-cm ConfigMap
data:
  configManagementPlugins: |
    - name: kustomize-sops
      generate:
        command: ["sh", "-c"]
        args: ["kustomize build --enable-alpha-plugins"]
```

---

## Solution 3: External Secrets Operator

Fetch secrets from external secret managers at runtime.

### Supported Providers

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SECRETS PROVIDERS                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Cloud Providers:                                                   │
│  • AWS Secrets Manager                                             │
│  • AWS Parameter Store                                             │
│  • Google Secret Manager                                           │
│  • Azure Key Vault                                                 │
│                                                                     │
│  Secret Managers:                                                   │
│  • HashiCorp Vault                                                 │
│  • CyberArk Conjur                                                 │
│  • Doppler                                                         │
│  • 1Password                                                       │
│                                                                     │
│  Kubernetes:                                                        │
│  • Kubernetes Secrets (cross-cluster)                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Install External Secrets Operator

```bash
# Add Helm repo
helm repo add external-secrets https://charts.external-secrets.io

# Install
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

### Configure SecretStore (AWS)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

### Configure SecretStore (Vault)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: default
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: "vault-auth"
```

### Create ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: default
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore

  target:
    name: database-secret  # K8s secret to create
    creationPolicy: Owner

  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username

    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

### Using in ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  source:
    repoURL: https://github.com/myorg/repo.git
    path: kubernetes/
    # ExternalSecret manifest is in this path
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

---

## Solution 4: HashiCorp Vault

Direct integration with Vault for secrets injection.

### Vault Agent Injector

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "my-app"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/my-app/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/my-app/config" -}}
          export DB_PASSWORD="{{ .Data.data.password }}"
          {{- end }}
    spec:
      serviceAccountName: my-app
      containers:
        - name: my-app
          image: my-app:latest
          command: ["/bin/sh", "-c", "source /vault/secrets/config && ./start.sh"]
```

### ArgoCD Vault Plugin (AVP)

```yaml
# Configure AVP in ArgoCD
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-plugin
  namespace: argocd
data:
  avp.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: argocd-vault-plugin
    spec:
      allowConcurrency: true
      generate:
        command:
          - argocd-vault-plugin
          - generate
          - ./
```

```yaml
# Application using AVP
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  source:
    repoURL: https://github.com/myorg/repo.git
    path: kubernetes/
    plugin:
      name: argocd-vault-plugin
```

```yaml
# Secret template with placeholders
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:
  password: <path:secret/data/my-app#password>
```

---

## Comparison of Solutions

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRETS SOLUTIONS COMPARISON                      │
├─────────────────┬──────────────┬──────────────┬─────────────────────┤
│ Feature         │Sealed Secrets│    SOPS      │ External Secrets    │
├─────────────────┼──────────────┼──────────────┼─────────────────────┤
│ Storage         │ Encrypted    │ Encrypted    │ External vault      │
│ Location        │ in Git       │ in Git       │ (not in Git)        │
├─────────────────┼──────────────┼──────────────┼─────────────────────┤
│ Key Management  │ Cluster      │ Cloud KMS/   │ External provider   │
│                 │ controller   │ PGP/age      │                     │
├─────────────────┼──────────────┼──────────────┼─────────────────────┤
│ Rotation        │ Manual       │ Manual       │ Automatic           │
│                 │ re-seal      │ re-encrypt   │ (configurable)      │
├─────────────────┼──────────────┼──────────────┼─────────────────────┤
│ Audit           │ Limited      │ Limited      │ Full (vault logs)   │
├─────────────────┼──────────────┼──────────────┼─────────────────────┤
│ Complexity      │ Low          │ Medium       │ High                │
├─────────────────┼──────────────┼──────────────┼─────────────────────┤
│ Multi-cluster   │ Per-cluster  │ Shared keys  │ Centralized         │
│                 │ keys         │ possible     │                     │
├─────────────────┼──────────────┼──────────────┼─────────────────────┤
│ Best For        │ Simple       │ Dev teams    │ Enterprise          │
│                 │ setups       │ familiar     │ with existing       │
│                 │              │ with Git     │ vault               │
└─────────────────┴──────────────┴──────────────┴─────────────────────┘
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRETS BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  General:                                                           │
│  • Never commit plain secrets to Git                               │
│  • Use separate secrets for each environment                       │
│  • Rotate secrets regularly                                        │
│  • Limit secret access (least privilege)                           │
│  • Audit secret access                                             │
│                                                                     │
│  ArgoCD Specific:                                                   │
│  • Don't store ArgoCD admin password in Git                        │
│  • Use SSO instead of local accounts                               │
│  • Encrypt repository credentials                                  │
│  • Use service accounts with limited permissions                   │
│                                                                     │
│  Choosing a Solution:                                               │
│  • Starting out? → Sealed Secrets                                  │
│  • Need cloud KMS? → SOPS                                          │
│  • Have existing vault? → External Secrets Operator                │
│  • Enterprise requirements? → HashiCorp Vault + AVP                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Secret Management Options:                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Sealed Secrets      │ Encrypt secrets, store in Git         │   │
│  │ SOPS                │ Encrypt files with cloud KMS/age      │   │
│  │ External Secrets    │ Fetch from external vaults            │   │
│  │ Vault + AVP         │ Direct Vault integration              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Recommendation by Use Case:                                        │
│  • Simple/Learning: Sealed Secrets                                 │
│  • Cloud-native: SOPS with cloud KMS                               │
│  • Enterprise: External Secrets + HashiCorp Vault                  │
│                                                                     │
│  Key Principle: Secrets should be encrypted at rest and            │
│  decrypted only at deployment time in the cluster.                 │
│                                                                     │
│  Next: Learn about multi-cluster management                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
