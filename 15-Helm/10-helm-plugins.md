# Helm Plugins

## Plugins Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM PLUGINS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What are Helm Plugins?                                            │
│  • Extensions that add new commands to Helm                        │
│  • Can be written in any language                                  │
│  • Installed to $HELM_PLUGINS directory                            │
│  • Invoked as helm <plugin-name>                                   │
│                                                                     │
│  Popular Plugins:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm-diff      │ Preview changes before upgrade             │   │
│  │ helm-secrets   │ Manage encrypted secrets                   │   │
│  │ helm-s3        │ S3 as chart repository                     │   │
│  │ helm-git       │ Use git repos as chart source              │   │
│  │ helm-push      │ Push charts to ChartMuseum                 │   │
│  │ helm-unittest  │ Unit test for charts                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Plugin Management

```bash
# List installed plugins
helm plugin list

# Install plugin
helm plugin install <url>

# Install specific version
helm plugin install <url> --version 1.0.0

# Update plugin
helm plugin update <name>

# Uninstall plugin
helm plugin uninstall <name>

# Plugin directory
echo $HELM_PLUGINS
# Default: ~/.local/share/helm/plugins
```

---

## Helm Diff

### Installation and Usage

```bash
# Install
helm plugin install https://github.com/databus23/helm-diff

# Preview upgrade changes
helm diff upgrade my-release bitnami/nginx -f values.yaml

# Show only changes (no context)
helm diff upgrade my-release bitnami/nginx --no-hooks

# Compare releases
helm diff revision my-release 2 3

# Compare with specific values
helm diff upgrade my-release bitnami/nginx \
  -f values.yaml \
  --set replicaCount=5

# Output formats
helm diff upgrade my-release bitnami/nginx --output dyff
helm diff upgrade my-release bitnami/nginx --output simple

# Suppress secrets
helm diff upgrade my-release bitnami/nginx --suppress-secrets

# Dry run in CI
helm diff upgrade my-release bitnami/nginx --detailed-exitcode
# Exit codes: 0=no changes, 1=error, 2=changes detected
```

### Diff in CI/CD

```yaml
# GitHub Actions with diff
- name: Helm diff
  run: |
    helm diff upgrade my-release ./charts/my-app \
      -f values-production.yaml \
      --detailed-exitcode || DIFF_EXIT=$?

    if [ "$DIFF_EXIT" -eq 2 ]; then
      echo "Changes detected, proceeding with upgrade"
    elif [ "$DIFF_EXIT" -eq 0 ]; then
      echo "No changes, skipping upgrade"
      exit 0
    else
      echo "Error occurred"
      exit 1
    fi
```

---

## Helm Secrets

### Installation

```bash
# Install plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Install SOPS (required backend)
brew install sops

# Install age (encryption tool)
brew install age
```

### Configuration

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*secrets.*\.yaml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

  - path_regex: .*production.*\.yaml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    kms: arn:aws:kms:us-east-1:123456789:key/xxxxx

  - path_regex: .*staging.*\.yaml$
    age: age1yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
```

### Usage

```bash
# Create encrypted values file
cat > secrets.yaml << EOF
database:
  password: supersecret
api:
  key: my-api-key
EOF

# Encrypt file
sops -e secrets.yaml > secrets.enc.yaml

# Or use helm secrets
helm secrets encrypt secrets.yaml > secrets.enc.yaml

# Decrypt file
helm secrets decrypt secrets.enc.yaml

# View decrypted content
helm secrets view secrets.enc.yaml

# Edit encrypted file
helm secrets edit secrets.enc.yaml

# Install with encrypted values
helm secrets install my-release ./my-chart -f secrets.enc.yaml

# Upgrade with encrypted values
helm secrets upgrade my-release ./my-chart -f secrets.enc.yaml

# Template with secrets
helm secrets template my-release ./my-chart -f secrets.enc.yaml
```

### Multiple Values Files

```bash
# Combine regular and encrypted values
helm secrets upgrade my-release ./my-chart \
  -f values.yaml \
  -f secrets.enc.yaml \
  -f values-production.yaml
```

---

## Helm Push (ChartMuseum)

```bash
# Install plugin
helm plugin install https://github.com/chartmuseum/helm-push

# Add repository
helm repo add my-charts https://charts.example.com

# Push chart
helm cm-push ./my-chart my-charts

# Push with specific version
helm cm-push ./my-chart my-charts --version 1.0.0

# Push with force (overwrite)
helm cm-push ./my-chart my-charts --force

# Push packaged chart
helm cm-push my-chart-1.0.0.tgz my-charts
```

---

## Helm S3

```bash
# Install plugin
helm plugin install https://github.com/hypnoglow/helm-s3.git

# Initialize S3 bucket
helm s3 init s3://my-helm-charts/stable

# Add S3 repository
helm repo add my-s3 s3://my-helm-charts/stable

# Push chart
helm s3 push ./my-chart-1.0.0.tgz my-s3

# Push with force
helm s3 push ./my-chart-1.0.0.tgz my-s3 --force

# Delete chart
helm s3 delete my-chart --version 1.0.0 my-s3

# Reindex
helm s3 reindex my-s3
```

---

## Helm Git

```bash
# Install plugin
helm plugin install https://github.com/aslafy-z/helm-git

# Use git repo as dependency
# Chart.yaml
dependencies:
  - name: my-lib
    version: "1.0.0"
    repository: "git+https://github.com/myorg/helm-charts@charts/my-lib?ref=v1.0.0"

# SSH authentication
dependencies:
  - name: my-lib
    version: "1.0.0"
    repository: "git+ssh://git@github.com/myorg/helm-charts@charts/my-lib?ref=main"

# With subdirectory
dependencies:
  - name: common
    version: "1.0.0"
    repository: "git+https://github.com/myorg/charts@libs/common?ref=v1.0.0"
```

---

## Helm Unittest

### Installation

```bash
# Install plugin
helm plugin install https://github.com/helm-unittest/helm-unittest

# Run tests
helm unittest ./my-chart

# Run with output format
helm unittest ./my-chart -o junit -f test-results.xml
```

### Writing Tests

```yaml
# tests/deployment_test.yaml
suite: deployment tests
templates:
  - deployment.yaml
tests:
  - it: should have correct replicas
    set:
      replicaCount: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3

  - it: should use correct image
    set:
      image:
        repository: nginx
        tag: "1.21"
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: nginx:1.21

  - it: should have resource limits
    asserts:
      - isNotNull:
          path: spec.template.spec.containers[0].resources.limits

  - it: should render nothing when disabled
    set:
      enabled: false
    asserts:
      - hasDocuments:
          count: 0

  - it: should have correct labels
    release:
      name: my-release
    asserts:
      - equal:
          path: metadata.labels["app.kubernetes.io/name"]
          value: my-chart
      - equal:
          path: metadata.labels["app.kubernetes.io/instance"]
          value: my-release

  - it: should set environment variables
    set:
      env:
        - name: LOG_LEVEL
          value: debug
    asserts:
      - contains:
          path: spec.template.spec.containers[0].env
          content:
            name: LOG_LEVEL
            value: debug
```

### Advanced Tests

```yaml
# tests/ingress_test.yaml
suite: ingress tests
templates:
  - ingress.yaml
tests:
  - it: should not create ingress when disabled
    set:
      ingress.enabled: false
    asserts:
      - hasDocuments:
          count: 0

  - it: should create ingress when enabled
    set:
      ingress:
        enabled: true
        hosts:
          - host: example.com
            paths:
              - path: /
    asserts:
      - hasDocuments:
          count: 1
      - isKind:
          of: Ingress
      - equal:
          path: spec.rules[0].host
          value: example.com

  - it: should configure TLS
    set:
      ingress:
        enabled: true
        tls:
          - secretName: tls-secret
            hosts:
              - example.com
    asserts:
      - equal:
          path: spec.tls[0].secretName
          value: tls-secret
```

---

## Helm Mapkubeapis

```bash
# Install plugin (for deprecated API migration)
helm plugin install https://github.com/helm/helm-mapkubeapis

# Check for deprecated APIs
helm mapkubeapis my-release

# Migrate deprecated APIs
helm mapkubeapis my-release --dry-run
helm mapkubeapis my-release
```

---

## Creating Custom Plugins

### Plugin Structure

```
my-plugin/
├── plugin.yaml       # Plugin metadata
├── bin/
│   └── my-plugin     # Executable script
└── scripts/
    └── helper.sh     # Helper scripts
```

### plugin.yaml

```yaml
# plugin.yaml
name: "my-plugin"
version: "1.0.0"
usage: "My custom plugin"
description: "Does something useful"
command: "$HELM_PLUGIN_DIR/bin/my-plugin"
hooks:
  install: "scripts/install.sh"
  update: "scripts/update.sh"
platformCommand:
  - os: linux
    arch: amd64
    command: "$HELM_PLUGIN_DIR/bin/my-plugin-linux-amd64"
  - os: darwin
    arch: amd64
    command: "$HELM_PLUGIN_DIR/bin/my-plugin-darwin-amd64"
```

### Plugin Script

```bash
#!/bin/bash
# bin/my-plugin

# Access Helm environment
echo "HELM_BIN: $HELM_BIN"
echo "HELM_PLUGIN_NAME: $HELM_PLUGIN_NAME"
echo "HELM_PLUGIN_DIR: $HELM_PLUGIN_DIR"

# Parse arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    --release)
      RELEASE="$2"
      shift 2
      ;;
    --namespace)
      NAMESPACE="$2"
      shift 2
      ;;
    *)
      echo "Unknown option: $1"
      exit 1
      ;;
  esac
done

# Plugin logic
echo "Running my-plugin for release: $RELEASE in namespace: $NAMESPACE"
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PLUGIN BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Selection:                                                         │
│  ✓ Check plugin maintenance status                                 │
│  ✓ Verify compatibility with Helm version                          │
│  ✓ Review security implications                                    │
│  ✓ Test in non-production first                                    │
│                                                                     │
│  Usage:                                                             │
│  ✓ Pin plugin versions in CI/CD                                    │
│  ✓ Document required plugins                                       │
│  ✓ Include plugin setup in onboarding                              │
│  ✓ Keep plugins updated                                            │
│                                                                     │
│  Essential Plugins:                                                 │
│  ✓ helm-diff for safe upgrades                                     │
│  ✓ helm-secrets for encrypted values                               │
│  ✓ helm-unittest for testing                                       │
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
│  Essential Plugins:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm-diff     │ Preview upgrade changes                     │   │
│  │ helm-secrets  │ Encrypted values with SOPS                  │   │
│  │ helm-unittest │ Unit testing for charts                     │   │
│  │ helm-push     │ Push to ChartMuseum                         │   │
│  │ helm-s3       │ S3 as repository                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Plugin Commands:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm plugin list      │ List installed                      │   │
│  │ helm plugin install   │ Install plugin                      │   │
│  │ helm plugin update    │ Update plugin                       │   │
│  │ helm plugin uninstall │ Remove plugin                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Recommended Setup:                                                 │
│  • Install helm-diff for safe upgrades                             │
│  • Use helm-secrets for production secrets                         │
│  • Use helm-unittest for chart testing                             │
│                                                                     │
│  Next: Learn about Helm security                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
