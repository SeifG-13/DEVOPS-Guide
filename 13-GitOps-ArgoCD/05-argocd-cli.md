# ArgoCD CLI

## Installing the CLI

### macOS

```bash
# Using Homebrew
brew install argocd

# Or download directly
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
curl -sSL -o argocd-darwin-amd64 https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-darwin-amd64
sudo install -m 555 argocd-darwin-amd64 /usr/local/bin/argocd
rm argocd-darwin-amd64
```

### Linux

```bash
# Download latest version
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### Windows

```powershell
# Using Chocolatey
choco install argocd-cli

# Or download manually from GitHub releases
# https://github.com/argoproj/argo-cd/releases
```

### Verify Installation

```bash
argocd version --client
# argocd: v2.9.3+...
```

---

## Logging In

### Login to ArgoCD Server

```bash
# Login with username/password
argocd login <ARGOCD_SERVER>

# Example with port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080

# Login with insecure flag (self-signed certs)
argocd login localhost:8080 --insecure

# Login with specific credentials
argocd login localhost:8080 --username admin --password <password>

# Login with SSO
argocd login localhost:8080 --sso

# Login using current kubeconfig context
argocd login --core
```

### Login Options

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LOGIN OPTIONS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  --insecure           Skip TLS certificate verification            │
│  --username, -u       Username for authentication                  │
│  --password, -p       Password for authentication                  │
│  --sso                Use SSO login                                │
│  --sso-port           Port for SSO callback (default 8085)         │
│  --grpc-web           Use gRPC-web protocol                        │
│  --core               Use in-cluster login via kubeconfig          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Check Login Context

```bash
# Show current context
argocd context

# List all contexts
argocd context list

# Switch context
argocd context <context-name>

# Delete context
argocd context delete <context-name>
```

---

## Application Commands

### Create Application

```bash
# Create from command line
argocd app create my-app \
  --repo https://github.com/org/repo.git \
  --path apps/my-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Create with Helm chart
argocd app create my-helm-app \
  --repo https://charts.helm.sh/stable \
  --helm-chart nginx \
  --revision 1.2.3 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Create with Helm values
argocd app create my-helm-app \
  --repo https://github.com/org/repo.git \
  --path helm/my-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --values values-prod.yaml \
  --helm-set image.tag=v1.0.0

# Create with Kustomize
argocd app create my-kustomize-app \
  --repo https://github.com/org/repo.git \
  --path kustomize/overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Create from YAML file
argocd app create -f application.yaml
```

### List Applications

```bash
# List all applications
argocd app list

# List with output format
argocd app list -o wide
argocd app list -o yaml
argocd app list -o json

# Filter by project
argocd app list --project production

# Filter by repo
argocd app list --repo https://github.com/org/repo.git
```

### Get Application Details

```bash
# Get application info
argocd app get my-app

# Get specific output
argocd app get my-app -o yaml
argocd app get my-app -o json

# Show resource tree
argocd app get my-app --show-operation
argocd app get my-app --refresh
```

### Application Output Example

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APP GET OUTPUT                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Name:               my-app                                        │
│  Project:            default                                       │
│  Server:             https://kubernetes.default.svc                │
│  Namespace:          production                                    │
│  URL:                https://argocd.example.com/applications/my-app│
│  Repo:               https://github.com/org/repo.git               │
│  Target:             HEAD                                          │
│  Path:               apps/my-app                                   │
│  SyncWindow:         Sync Allowed                                  │
│  Sync Policy:        Automated (Prune)                             │
│  Sync Status:        Synced to HEAD (abc1234)                      │
│  Health Status:      Healthy                                       │
│                                                                     │
│  GROUP  KIND        NAMESPACE   NAME       STATUS  HEALTH  HOOK    │
│  ───────────────────────────────────────────────────────────────   │
│         Service     production  my-app     Synced  Healthy         │
│  apps   Deployment  production  my-app     Synced  Healthy         │
│         ConfigMap   production  my-config  Synced                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Sync Operations

### Sync Application

```bash
# Sync application
argocd app sync my-app

# Sync with prune (delete resources not in Git)
argocd app sync my-app --prune

# Sync with force (replace resources)
argocd app sync my-app --force

# Sync specific resources only
argocd app sync my-app --resource apps:Deployment:my-app

# Sync with dry-run (preview changes)
argocd app sync my-app --dry-run

# Sync to specific revision
argocd app sync my-app --revision abc1234
argocd app sync my-app --revision v1.0.0  # tag
argocd app sync my-app --revision main    # branch

# Sync with specific strategy
argocd app sync my-app --strategy apply   # kubectl apply
argocd app sync my-app --strategy hook    # run hooks only
```

### Sync Options

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNC OPTIONS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  --prune              Delete resources not in Git                  │
│  --force              Force sync (recreate resources)              │
│  --dry-run            Preview changes without applying             │
│  --revision           Sync to specific Git revision                │
│  --resource           Sync specific resources only                 │
│  --async              Don't wait for sync to complete              │
│  --timeout            Sync timeout in seconds                      │
│  --retry-limit        Number of sync retries                       │
│  --replace            Replace resources (kubectl replace)          │
│  --server-side        Use server-side apply                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Wait for Sync

```bash
# Wait for application to be synced and healthy
argocd app wait my-app

# Wait with timeout
argocd app wait my-app --timeout 300

# Wait for specific health status
argocd app wait my-app --health

# Wait for sync only (not health)
argocd app wait my-app --sync
```

---

## Diff and History

### View Diff

```bash
# Show diff between Git and cluster
argocd app diff my-app

# Show diff for specific revision
argocd app diff my-app --revision HEAD~1

# Show diff with local manifests
argocd app diff my-app --local ./path/to/manifests
```

### View History

```bash
# Show sync history
argocd app history my-app

# Output:
# ID  DATE                    REVISION
# 1   2024-01-15 10:30:00    abc1234 (HEAD)
# 0   2024-01-14 09:00:00    def5678
```

### Rollback

```bash
# Rollback to previous sync
argocd app rollback my-app

# Rollback to specific history ID
argocd app rollback my-app 0

# Rollback with prune
argocd app rollback my-app --prune
```

---

## Application Management

### Delete Application

```bash
# Delete application (keeps resources in cluster)
argocd app delete my-app

# Delete application and cascade delete resources
argocd app delete my-app --cascade

# Force delete
argocd app delete my-app --cascade --yes
```

### Patch Application

```bash
# Patch application spec
argocd app patch my-app --patch '{"spec":{"syncPolicy":{"automated":{"prune":true}}}}'

# Patch using JSON patch
argocd app patch my-app --patch '[{"op":"replace","path":"/spec/source/targetRevision","value":"v2.0.0"}]' --type json
```

### Set Application Parameters

```bash
# Set Helm parameters
argocd app set my-app --helm-set image.tag=v2.0.0
argocd app set my-app --values values-prod.yaml
argocd app set my-app --helm-set-string annotations."key"="value"

# Set Kustomize parameters
argocd app set my-app --kustomize-image myapp=myregistry/myapp:v2.0.0
argocd app set my-app --nameprefix prod-
argocd app set my-app --namesuffix -v2

# Set sync policy
argocd app set my-app --sync-policy automated
argocd app set my-app --auto-prune
argocd app set my-app --self-heal

# Change destination
argocd app set my-app --dest-namespace production
argocd app set my-app --dest-server https://prod-cluster.example.com
```

### Refresh Application

```bash
# Refresh app (soft refresh - check Git for changes)
argocd app get my-app --refresh

# Hard refresh (clear cache)
argocd app get my-app --hard-refresh
```

---

## Resource Commands

### List Resources

```bash
# List application resources
argocd app resources my-app

# Output format
argocd app resources my-app -o yaml
argocd app resources my-app -o json
```

### View Resource Manifests

```bash
# Get live manifest from cluster
argocd app manifests my-app

# Get desired manifest from Git
argocd app manifests my-app --source git

# Get specific resource
argocd app manifests my-app --resource apps:Deployment:my-app
```

### Logs

```bash
# View logs for specific resource
argocd app logs my-app --name my-app-pod-xyz

# View logs for deployment
argocd app logs my-app --group apps --kind Deployment --name my-app

# Follow logs
argocd app logs my-app --name my-app-pod-xyz --follow

# Tail logs
argocd app logs my-app --name my-app-pod-xyz --tail 100
```

### Actions

```bash
# Perform actions on resources
argocd app actions list my-app --kind Deployment

# Restart deployment
argocd app actions run my-app restart --kind Deployment --resource-name my-app
```

---

## Project Commands

### List Projects

```bash
argocd proj list
```

### Create Project

```bash
# Create project
argocd proj create production \
  --description "Production applications" \
  --src https://github.com/org/* \
  --dest https://kubernetes.default.svc,production

# Create from YAML
argocd proj create -f project.yaml
```

### Get Project

```bash
argocd proj get production
```

### Configure Project

```bash
# Add source repository
argocd proj add-source production https://github.com/org/new-repo

# Add destination
argocd proj add-destination production https://prod-cluster,production-namespace

# Allow cluster resource
argocd proj allow-cluster-resource production "" Namespace

# Deny namespace resource
argocd proj deny-namespace-resource production "" Secret
```

### Delete Project

```bash
argocd proj delete production
```

---

## Repository Commands

### Add Repository

```bash
# Add HTTPS repository
argocd repo add https://github.com/org/repo.git \
  --username git \
  --password $GITHUB_TOKEN

# Add SSH repository
argocd repo add git@github.com:org/repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa

# Add Helm repository
argocd repo add https://charts.helm.sh/stable \
  --type helm \
  --name stable
```

### List Repositories

```bash
argocd repo list
```

### Remove Repository

```bash
argocd repo rm https://github.com/org/repo.git
```

---

## Cluster Commands

### Add Cluster

```bash
# Add cluster from kubeconfig context
argocd cluster add my-context

# Add with specific name
argocd cluster add my-context --name production-cluster

# Add in-cluster
argocd cluster add in-cluster --in-cluster
```

### List Clusters

```bash
argocd cluster list
```

### Get Cluster

```bash
argocd cluster get https://kubernetes.default.svc
```

### Remove Cluster

```bash
argocd cluster rm https://prod-cluster.example.com
```

---

## Account Commands

### Account Management

```bash
# List accounts
argocd account list

# Get current user info
argocd account get-user-info

# Update password
argocd account update-password

# Generate auth token
argocd account generate-token --account admin

# Check permissions
argocd account can-i get applications '*/*'
argocd account can-i sync applications 'my-project/*'
```

---

## Common Command Patterns

### Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMMON COMMANDS CHEATSHEET                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  LOGIN & CONTEXT                                                    │
│  argocd login <server> --insecure                                  │
│  argocd context                                                    │
│                                                                     │
│  APPLICATION LIFECYCLE                                              │
│  argocd app create <name> --repo <url> --path <path> \             │
│    --dest-server <server> --dest-namespace <ns>                    │
│  argocd app list                                                   │
│  argocd app get <name>                                             │
│  argocd app sync <name>                                            │
│  argocd app delete <name> --cascade                                │
│                                                                     │
│  SYNC OPERATIONS                                                    │
│  argocd app sync <name> --dry-run                                  │
│  argocd app sync <name> --prune --force                            │
│  argocd app sync <name> --revision <commit/tag/branch>             │
│                                                                     │
│  DEBUGGING                                                          │
│  argocd app diff <name>                                            │
│  argocd app history <name>                                         │
│  argocd app rollback <name>                                        │
│  argocd app logs <name> --name <pod>                               │
│                                                                     │
│  CONFIGURATION                                                      │
│  argocd app set <name> --helm-set key=value                        │
│  argocd app set <name> --sync-policy automated                     │
│  argocd repo add <url> --username <user> --password <pass>         │
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
│  The ArgoCD CLI provides complete control over:                     │
│  • Applications (create, sync, delete, rollback)                   │
│  • Projects (RBAC boundaries)                                      │
│  • Repositories (source configuration)                             │
│  • Clusters (target clusters)                                      │
│  • Accounts (user management)                                      │
│                                                                     │
│  Key Commands to Remember:                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ argocd login <server>          # Connect to ArgoCD          │   │
│  │ argocd app create <name> ...   # Create application         │   │
│  │ argocd app sync <name>         # Deploy changes             │   │
│  │ argocd app get <name>          # Check status               │   │
│  │ argocd app diff <name>         # View differences           │   │
│  │ argocd app rollback <name>     # Revert changes             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Next: Create and manage ArgoCD Applications                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
