# Helm Fundamentals

## What is Helm?

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HELM OVERVIEW                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Helm is the package manager for Kubernetes                        │
│                                                                     │
│  Key Concepts:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Chart        │ Package of Kubernetes resources              │   │
│  │ Release      │ Instance of a chart running in cluster       │   │
│  │ Repository   │ Collection of charts                         │   │
│  │ Values       │ Configuration for chart customization        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Why Helm?                                                          │
│  • Simplifies Kubernetes deployments                               │
│  • Enables reproducible installations                              │
│  • Manages complex applications                                    │
│  • Supports versioning and rollbacks                               │
│  • Promotes configuration reuse                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Helm Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM V3 ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐                                                   │
│  │   Helm CLI   │                                                   │
│  │              │                                                   │
│  └──────┬───────┘                                                   │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │    Chart     │    │    Values    │    │  Templates   │         │
│  │  Repository  │    │    Files     │    │              │         │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘         │
│         │                   │                   │                   │
│         └───────────────────┼───────────────────┘                   │
│                             │                                       │
│                             ▼                                       │
│                    ┌──────────────┐                                 │
│                    │   Rendered   │                                 │
│                    │  Manifests   │                                 │
│                    └──────┬───────┘                                 │
│                           │                                         │
│                           ▼                                         │
│                    ┌──────────────┐                                 │
│                    │  Kubernetes  │                                 │
│                    │     API      │                                 │
│                    └──────────────┘                                 │
│                                                                     │
│  Note: Helm v3 removed Tiller (server-side component)              │
│  Charts are rendered client-side and applied directly              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Installation

### Installing Helm

```bash
# macOS
brew install helm

# Windows (Chocolatey)
choco install kubernetes-helm

# Windows (Scoop)
scoop install helm

# Linux (Snap)
sudo snap install helm --classic

# Linux (Script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

### Helm Configuration

```bash
# Helm uses kubeconfig by default
# Same as kubectl configuration

# Set custom kubeconfig
export KUBECONFIG=/path/to/kubeconfig

# Helm cache and config directories
# Linux/macOS: ~/.cache/helm, ~/.config/helm
# Windows: %APPDATA%\helm

# View Helm environment
helm env
```

---

## Basic Commands

### Repository Management

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# Update repositories
helm repo update

# List repositories
helm repo list

# Search for charts
helm search repo nginx
helm search repo bitnami/nginx --versions

# Search Artifact Hub
helm search hub wordpress

# Remove repository
helm repo remove stable
```

### Chart Operations

```bash
# Install a chart
helm install my-release bitnami/nginx

# Install with custom name generation
helm install bitnami/nginx --generate-name

# Install in specific namespace
helm install my-release bitnami/nginx -n production

# Install with custom values
helm install my-release bitnami/nginx --set service.type=LoadBalancer

# Install from local chart
helm install my-release ./my-chart

# Install with values file
helm install my-release bitnami/nginx -f values.yaml

# Dry run (preview)
helm install my-release bitnami/nginx --dry-run

# Wait for resources to be ready
helm install my-release bitnami/nginx --wait --timeout 5m
```

### Release Management

```bash
# List releases
helm list
helm list -A  # All namespaces
helm list -n production  # Specific namespace

# Get release status
helm status my-release

# Get release history
helm history my-release

# Upgrade a release
helm upgrade my-release bitnami/nginx --set replicaCount=3

# Upgrade or install
helm upgrade --install my-release bitnami/nginx

# Rollback to previous revision
helm rollback my-release

# Rollback to specific revision
helm rollback my-release 2

# Uninstall a release
helm uninstall my-release

# Uninstall keeping history
helm uninstall my-release --keep-history
```

### Chart Inspection

```bash
# Show chart information
helm show chart bitnami/nginx

# Show default values
helm show values bitnami/nginx

# Show README
helm show readme bitnami/nginx

# Show all info
helm show all bitnami/nginx

# Download chart locally
helm pull bitnami/nginx
helm pull bitnami/nginx --untar

# Get rendered manifests
helm get manifest my-release

# Get values used in release
helm get values my-release

# Get all release info
helm get all my-release
```

---

## Chart Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CHART DIRECTORY STRUCTURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  my-chart/                                                          │
│  ├── Chart.yaml           # Chart metadata                          │
│  ├── values.yaml          # Default configuration values            │
│  ├── charts/              # Dependency charts                       │
│  ├── templates/           # Kubernetes manifest templates           │
│  │   ├── deployment.yaml                                            │
│  │   ├── service.yaml                                               │
│  │   ├── ingress.yaml                                               │
│  │   ├── _helpers.tpl     # Template helpers                        │
│  │   ├── NOTES.txt        # Post-install notes                      │
│  │   └── tests/           # Test templates                          │
│  │       └── test-connection.yaml                                   │
│  ├── crds/                # Custom Resource Definitions             │
│  ├── .helmignore          # Files to ignore when packaging          │
│  └── README.md            # Chart documentation                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
type: application  # or 'library'
version: 1.0.0
appVersion: "2.0.0"

# Optional fields
keywords:
  - web
  - application
home: https://example.com
sources:
  - https://github.com/example/my-app
maintainers:
  - name: John Doe
    email: john@example.com
    url: https://example.com
icon: https://example.com/icon.png
deprecated: false
annotations:
  category: WebApplication

# Dependencies
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
# values.yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  name: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```

---

## Common Use Cases

### Installing Multiple Releases

```bash
# Different environments
helm install app-dev ./my-app -f values-dev.yaml -n dev
helm install app-staging ./my-app -f values-staging.yaml -n staging
helm install app-prod ./my-app -f values-prod.yaml -n production

# Multiple instances
helm install redis-cache bitnami/redis --set architecture=standalone
helm install redis-session bitnami/redis --set architecture=standalone
```

### Upgrading Applications

```bash
# Simple upgrade
helm upgrade my-release bitnami/nginx

# Upgrade with new values
helm upgrade my-release bitnami/nginx -f new-values.yaml

# Upgrade with version
helm upgrade my-release bitnami/nginx --version 15.0.0

# Atomic upgrade (rollback on failure)
helm upgrade my-release bitnami/nginx --atomic

# Force resource updates
helm upgrade my-release bitnami/nginx --force
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM BEST PRACTICES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Installation:                                                      │
│  ✓ Always use --dry-run first                                      │
│  ✓ Use namespaces to organize releases                             │
│  ✓ Use meaningful release names                                    │
│  ✓ Pin chart versions in production                                │
│                                                                     │
│  Values:                                                            │
│  ✓ Use values files instead of --set for complex values            │
│  ✓ Keep environment-specific values separate                       │
│  ✓ Document all values in values.yaml                              │
│  ✓ Use sensible defaults                                           │
│                                                                     │
│  Upgrades:                                                          │
│  ✓ Review changes with --dry-run --debug                           │
│  ✓ Use --atomic for production upgrades                            │
│  ✓ Test upgrades in non-production first                           │
│  ✓ Keep release history for rollbacks                              │
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
│  Key Concepts:                                                      │
│  • Chart - Package of Kubernetes resources                         │
│  • Release - Running instance of a chart                           │
│  • Repository - Collection of charts                               │
│  • Values - Configuration customization                            │
│                                                                     │
│  Essential Commands:                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm repo add    │ Add chart repository                     │   │
│  │ helm search      │ Find charts                              │   │
│  │ helm install     │ Install a chart                          │   │
│  │ helm upgrade     │ Upgrade a release                        │   │
│  │ helm rollback    │ Rollback to previous version             │   │
│  │ helm uninstall   │ Remove a release                         │   │
│  │ helm list        │ List releases                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Helm v3 Benefits:                                                  │
│  • No Tiller (improved security)                                   │
│  • Uses Kubernetes RBAC                                            │
│  • Release stored as secrets                                       │
│  • Improved upgrade strategy                                       │
│                                                                     │
│  Next: Learn about Helm chart development                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
