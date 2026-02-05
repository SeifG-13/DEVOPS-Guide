# ArgoCD Repositories

## Repository Types

ArgoCD supports multiple types of repositories for sourcing application manifests.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    REPOSITORY TYPES                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   Git Repos     │  │   Helm Repos    │  │   OCI Repos     │    │
│  │                 │  │                 │  │                 │    │
│  │ • GitHub        │  │ • Chart Museum  │  │ • Docker Hub    │    │
│  │ • GitLab        │  │ • Artifact Hub  │  │ • GHCR          │    │
│  │ • Bitbucket     │  │ • Custom        │  │ • ECR           │    │
│  │ • Azure DevOps  │  │                 │  │ • ACR           │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
│                                                                     │
│  Authentication Methods:                                            │
│  • HTTPS with username/password (token)                            │
│  • SSH with private key                                            │
│  • GitHub App                                                      │
│  • Google Cloud Source Repositories                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Adding Git Repositories

### Via CLI - HTTPS

```bash
# Public repository (no auth needed)
argocd repo add https://github.com/argoproj/argocd-example-apps.git

# Private repository with token
argocd repo add https://github.com/myorg/private-repo.git \
  --username git \
  --password $GITHUB_TOKEN

# With specific name
argocd repo add https://github.com/myorg/private-repo.git \
  --username git \
  --password $GITHUB_TOKEN \
  --name my-private-repo
```

### Via CLI - SSH

```bash
# SSH repository with key file
argocd repo add git@github.com:myorg/private-repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa

# SSH with insecure ignore host key (not recommended)
argocd repo add git@github.com:myorg/private-repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa \
  --insecure-ignore-host-key
```

### Via Kubernetes Secret - HTTPS

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/myorg/private-repo.git
  username: git
  password: ghp_xxxxxxxxxxxxxxxxxxxx
```

### Via Kubernetes Secret - SSH

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:myorg/private-repo.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdz
    ...
    -----END OPENSSH PRIVATE KEY-----
```

---

## Credential Templates

Credential templates allow you to define credentials once for multiple repositories.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: https://github.com/myorg  # URL prefix to match
  username: git
  password: ghp_xxxxxxxxxxxxxxxxxxxx
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CREDENTIAL TEMPLATE MATCHING                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Template URL: https://github.com/myorg                            │
│                                                                     │
│  Matches:                                                           │
│  ✓ https://github.com/myorg/repo1.git                              │
│  ✓ https://github.com/myorg/repo2.git                              │
│  ✓ https://github.com/myorg/team/repo3.git                         │
│                                                                     │
│  Does NOT match:                                                    │
│  ✗ https://github.com/otherorg/repo.git                            │
│  ✗ https://gitlab.com/myorg/repo.git                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### SSH Credential Template

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-ssh-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: git@github.com:myorg
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

---

## Adding Helm Repositories

### Public Helm Repository

```bash
# Add public Helm repo
argocd repo add https://charts.helm.sh/stable --type helm --name stable

# Add Bitnami
argocd repo add https://charts.bitnami.com/bitnami --type helm --name bitnami

# Add Prometheus community
argocd repo add https://prometheus-community.github.io/helm-charts \
  --type helm --name prometheus-community
```

### Private Helm Repository

```bash
# With username/password
argocd repo add https://charts.example.com \
  --type helm \
  --name private-charts \
  --username admin \
  --password secretpassword

# With TLS client certificate
argocd repo add https://charts.example.com \
  --type helm \
  --name private-charts \
  --tls-client-cert-path /path/to/cert.pem \
  --tls-client-cert-key-path /path/to/key.pem
```

### Via Kubernetes Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: helm
  name: private-charts
  url: https://charts.example.com
  username: admin
  password: secretpassword
```

---

## OCI Repositories

ArgoCD supports Helm charts stored as OCI artifacts.

### Adding OCI Repository

```bash
# Add OCI registry
argocd repo add oci://ghcr.io/myorg \
  --type helm \
  --name ghcr-charts \
  --username myuser \
  --password $GITHUB_TOKEN

# AWS ECR
argocd repo add oci://123456789.dkr.ecr.us-east-1.amazonaws.com \
  --type helm \
  --name ecr-charts \
  --username AWS \
  --password $(aws ecr get-login-password)
```

### Via Kubernetes Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: oci-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: helm
  name: ghcr-charts
  url: oci://ghcr.io/myorg
  username: myuser
  password: ghp_xxxxxxxxxxxxxxxxxxxx
  enableOCI: "true"
```

### Using OCI in Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-oci-app
spec:
  source:
    repoURL: oci://ghcr.io/myorg/charts
    chart: my-chart
    targetRevision: 1.0.0
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

---

## GitHub App Authentication

GitHub App provides better security and higher rate limits.

### Create GitHub App

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GITHUB APP SETUP                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Go to GitHub → Settings → Developer settings → GitHub Apps     │
│  2. Create New GitHub App                                          │
│  3. Configure:                                                      │
│     • Name: ArgoCD-MyOrg                                           │
│     • Homepage URL: https://argocd.example.com                     │
│     • Webhook: Disable                                             │
│  4. Permissions:                                                    │
│     • Repository contents: Read-only                               │
│     • Metadata: Read-only                                          │
│  5. Install app on organization/repositories                       │
│  6. Note: App ID, Installation ID, Private Key                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Configure in ArgoCD

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-app-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: https://github.com/myorg
  githubAppID: "123456"
  githubAppInstallationID: "12345678"
  githubAppPrivateKey: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
```

---

## TLS Configuration

### Custom CA Certificate

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-tls
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://git.internal.example.com/myorg/repo.git
  username: git
  password: token123
  tlsClientCertData: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  tlsClientCertKey: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
```

### Skip TLS Verification (Not Recommended)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: insecure-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://git.internal.example.com/repo.git
  username: git
  password: token123
  insecure: "true"  # Skip TLS verification
```

### Add CA to ArgoCD

```yaml
# Add to argocd-tls-certs-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-tls-certs-cm
  namespace: argocd
data:
  git.internal.example.com: |
    -----BEGIN CERTIFICATE-----
    MIIFazCCA1OgAwIBAgIUe...
    -----END CERTIFICATE-----
```

---

## SSH Known Hosts

### Configure Known Hosts

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-ssh-known-hosts-cm
  namespace: argocd
data:
  ssh_known_hosts: |
    # GitHub
    github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl
    github.com ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEmKSENjQEezOmxkZMy7opKgwFB9nkt5YRrYMjNuG5N87uRgg6CLrbo5wAdT/y6v0mKV0U2w0WZ2YB/++Tpockg=
    github.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCj7ndNxQowgcQnjshcLrqPEiiphnt+VTTvDP6mHBL9j1aNUkY4Ue1gvwnGLVlOhGeYrnZaMgRK6+PKCUXaDbC7qtbW8gIkhL7aGCsOr/C56SJMy/BCZfxd1nWzAOxSDPgVsmerOBYfNqltV9/hWCqBywINIR+5dIg6JTJ72pcEpEjcYgXkE2YEFXV1JHnsKgbLWNlhScqb2UmyRkQyytRLtL+38TGxkxCflmO+5Z8CSSNY7GidjMIZ7Q4zMjA2n1nGrlTDkzwDCsw+wqFPGQA179cnfGWOWRVruj16z6XyvxvjJwbz0wQZ75XK5tKSb7FNyeIEs4TT4jk+S4dhPeAUC5y+bDYirYgM4GC7uEnztnZyaVWQ7B381AK4Qdrwt51ZqExKbQpTUNn+EjqoTwvqNj4kqx5QUCI0ThS/YkOxJCXmPUWZbhjpCg56i+2aB6CmK2JGhn57K5mj0MNdBXA4/WnwH6XoPWJzK5Nyu2zB3nAZp+S5hpQs+p1vN1/wsjk=

    # GitLab
    gitlab.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAfuCHKVTjquxvt6CM6tdG4SLp1Btn/nOeHHE5UOzRdf
    gitlab.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCsj2bNKTBSpIYDEGk9KxsGh3mySTRgMtXL583qmBpzeQ+jqCMRgBqB98u3z++J1sKlXHWfM9dyhSevkMwSbhoR8XIq/U0tCNyokEi/ueaBMCvbcTHhO7FcwzY92WK4Yt0aGROY5qX2UKSeOvuP4D6TPqKF1onrSzH9bx9XUf2lEdWT/ia1NEKjunUqu1xOB/StKDHMoX4/OKyIzuS0q/T1zOATthvasJFoPrAjkohTyaDUz2LN5JoH839hViyEG82yB+MjcFV5MU3N1l1QL3cVUCh93xSaua1N85qivl+siMkPGbO5xR/En4iEY6K2XPASUEMaieWVNTRCtJ4S8H+9

    # Internal Git Server
    git.internal.example.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB...
```

### Get SSH Keys

```bash
# Get GitHub's SSH keys
ssh-keyscan github.com

# Get GitLab's SSH keys
ssh-keyscan gitlab.com

# Get custom server keys
ssh-keyscan git.internal.example.com
```

---

## Managing Repositories

### List Repositories

```bash
# List all repositories
argocd repo list

# Output format
argocd repo list -o yaml
argocd repo list -o json
```

### Get Repository Details

```bash
argocd repo get https://github.com/myorg/repo.git
```

### Remove Repository

```bash
argocd repo rm https://github.com/myorg/repo.git
```

### Validate Repository

```bash
# Test repository connection
argocd repo get https://github.com/myorg/repo.git --refresh
```

---

## Repository Server Configuration

### Increase Clone Timeout

```yaml
# In argocd-cmd-params-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  reposerver.git.request.timeout: "300"  # 5 minutes
```

### Configure Parallelism

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  reposerver.parallelism.limit: "10"
```

---

## Troubleshooting

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TROUBLESHOOTING                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Issue: Repository connection failed                                │
│  ─────────────────────────────────────                              │
│  • Check credentials are correct                                   │
│  • Verify URL format (HTTPS vs SSH)                                │
│  • Check network connectivity                                      │
│  kubectl logs -n argocd deployment/argocd-repo-server              │
│                                                                     │
│  Issue: SSH host key verification failed                           │
│  ─────────────────────────────────────────                          │
│  • Add host key to argocd-ssh-known-hosts-cm                       │
│  • Or use --insecure-ignore-host-key (not recommended)             │
│                                                                     │
│  Issue: TLS certificate error                                       │
│  ───────────────────────────────                                    │
│  • Add CA cert to argocd-tls-certs-cm                              │
│  • Or set insecure: "true" in secret (not recommended)             │
│                                                                     │
│  Issue: Rate limiting (GitHub)                                      │
│  ────────────────────────────                                       │
│  • Use GitHub App authentication                                   │
│  • Use personal access token with higher limits                    │
│                                                                     │
│  Check repo server logs:                                           │
│  kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server│
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
│  Repository Types:                                                  │
│  • Git (HTTPS, SSH)                                                │
│  • Helm repositories                                               │
│  • OCI registries                                                  │
│                                                                     │
│  Authentication Methods:                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ HTTPS         │ Username + password/token                   │   │
│  │ SSH           │ Private key                                 │   │
│  │ GitHub App    │ App ID + Installation ID + Private Key      │   │
│  │ TLS           │ Client certificate                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Best Practices:                                                    │
│  • Use credential templates for multiple repos                     │
│  • Use GitHub Apps for better rate limits                          │
│  • Store credentials in Kubernetes Secrets                         │
│  • Configure SSH known hosts properly                              │
│  • Use dedicated service accounts/tokens                           │
│                                                                     │
│  Next: Learn Helm and Kustomize integration                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
