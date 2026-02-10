# Helm Troubleshooting

## Common Issues Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM TROUBLESHOOTING                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Common Issue Categories:                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Installation  │ Chart not found, dependency issues          │   │
│  │ Template      │ Rendering errors, invalid YAML              │   │
│  │ Upgrade       │ Resource conflicts, immutable fields        │   │
│  │ Rollback      │ History issues, hooks failing               │   │
│  │ Values        │ Type mismatches, missing required           │   │
│  │ Connection    │ Cluster access, RBAC issues                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Debugging Commands

### Basic Debugging

```bash
# Check Helm version
helm version

# Verify cluster connection
helm list

# Debug template rendering
helm template my-release ./my-chart --debug

# Dry run installation
helm install my-release ./my-chart --dry-run --debug

# Get release status
helm status my-release

# Get release history
helm history my-release

# Get manifest of deployed release
helm get manifest my-release

# Get values used in release
helm get values my-release
helm get values my-release --all  # Include defaults

# Get all release info
helm get all my-release

# Get release notes
helm get notes my-release
```

### Kubernetes Debugging

```bash
# Check pods
kubectl get pods -l app.kubernetes.io/instance=my-release

# Describe failing pod
kubectl describe pod <pod-name>

# Check pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container logs

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check resources created by Helm
kubectl get all -l app.kubernetes.io/instance=my-release
```

---

## Installation Issues

### Chart Not Found

```bash
# Error: chart "myrepo/my-chart" not found
# Solutions:

# Update repositories
helm repo update

# Check repository is added
helm repo list

# Search for chart
helm search repo my-chart

# Check chart name/version
helm search repo myrepo/my-chart --versions
```

### Dependency Issues

```bash
# Error: found in Chart.yaml, but missing in charts/ directory
# Solutions:

# Update dependencies
helm dependency update ./my-chart

# Or build dependencies
helm dependency build ./my-chart

# Check dependency status
helm dependency list ./my-chart
```

### Namespace Issues

```bash
# Error: namespace "production" not found
# Solutions:

# Create namespace
kubectl create namespace production

# Or use --create-namespace
helm install my-release ./my-chart -n production --create-namespace
```

---

## Template Errors

### YAML Syntax Errors

```bash
# Error: YAML parse error
# Debug with:
helm template my-release ./my-chart --debug 2>&1 | head -50

# Common causes:
# - Incorrect indentation
# - Missing quotes around strings with special chars
# - Invalid YAML in values
```

### Template Function Errors

```yaml
# Error: function "xyz" not defined
# Common solutions:

# Wrong:
{{ .Values.name | uppercase }}
# Correct:
{{ .Values.name | upper }}

# Wrong:
{{ if .Values.enabled = true }}
# Correct:
{{ if eq .Values.enabled true }}

# Wrong:
{{ include "myhelper" }}
# Correct:
{{ include "myhelper" . }}
```

### Nil Pointer Errors

```yaml
# Error: nil pointer evaluating interface {}
# Solution: Use default or check existence

# Wrong:
{{ .Values.config.nested.value }}

# Correct:
{{ .Values.config.nested.value | default "" }}

# Or with if:
{{- if .Values.config }}
{{- if .Values.config.nested }}
{{ .Values.config.nested.value }}
{{- end }}
{{- end }}

# Or with with:
{{- with .Values.config }}
{{- with .nested }}
{{ .value }}
{{- end }}
{{- end }}
```

---

## Upgrade Issues

### Immutable Field Changed

```bash
# Error: field is immutable
# Common fields: selector, clusterIP, volumeClaimTemplates

# Solution 1: Delete and reinstall
helm uninstall my-release
helm install my-release ./my-chart

# Solution 2: Use --force (recreates resources)
helm upgrade my-release ./my-chart --force

# Solution 3: Manually delete the resource
kubectl delete deployment my-release-app
helm upgrade my-release ./my-chart
```

### Resource Already Exists

```bash
# Error: cannot re-use a name that is still in use
# Or: resource already exists and is not managed by Helm

# Check existing resources
kubectl get all -l app.kubernetes.io/name=my-app

# Solution 1: Adopt existing resources
kubectl annotate <resource> meta.helm.sh/release-name=my-release
kubectl annotate <resource> meta.helm.sh/release-namespace=default
kubectl label <resource> app.kubernetes.io/managed-by=Helm

# Solution 2: Delete conflicting resources
kubectl delete <resource-type> <resource-name>
```

### Failed Upgrade with Pending Status

```bash
# Error: another operation (install/upgrade/rollback) is in progress
# Check release status
helm status my-release

# If stuck in pending-upgrade:
# Option 1: Wait for timeout
# Option 2: Rollback
helm rollback my-release

# If rollback fails, force it:
kubectl delete secret -l owner=helm,name=my-release,status=pending-upgrade
helm upgrade my-release ./my-chart
```

---

## Rollback Issues

### No Revision to Rollback

```bash
# Error: no revision to roll back to
# Check history
helm history my-release

# If history is empty, you can't rollback
# Prevention: Don't use --keep-history with uninstall
```

### Hook Failures During Rollback

```bash
# Error: pre-rollback hook failed
# Skip hooks if needed
helm rollback my-release 1 --no-hooks

# Or cleanup failed hooks
kubectl delete job -l helm.sh/hook=pre-rollback
helm rollback my-release 1
```

---

## Values Issues

### Type Mismatch

```bash
# Error: cannot convert string to int
# Solution: Force correct type

# Use --set-string for strings
helm install my-release ./my-chart --set-string image.tag="1.0"

# Use proper typing in values
helm install my-release ./my-chart --set replicaCount=3  # int
helm install my-release ./my-chart --set-string name="myapp"  # string
```

### Required Value Missing

```yaml
# In templates:
{{ required "image.repository is required" .Values.image.repository }}

# Error: image.repository is required
# Solution: Provide the value
helm install my-release ./my-chart --set image.repository=nginx
```

### Complex Values with --set

```bash
# Setting nested values
helm install my-release ./my-chart \
  --set resources.limits.memory=256Mi \
  --set resources.limits.cpu=100m

# Setting arrays
helm install my-release ./my-chart \
  --set ingress.hosts[0].host=example.com \
  --set ingress.hosts[0].paths[0].path=/

# Escaping special characters
helm install my-release ./my-chart \
  --set annotations."prometheus\.io/scrape"=true

# Setting JSON values
helm install my-release ./my-chart \
  --set-json 'env=[{"name":"A","value":"B"}]'
```

---

## Connection Issues

### Cluster Connection Failed

```bash
# Error: Kubernetes cluster unreachable
# Check kubectl connection
kubectl cluster-info

# Check kubeconfig
kubectl config view
kubectl config current-context

# Set correct context
kubectl config use-context my-cluster

# Specify kubeconfig explicitly
export KUBECONFIG=/path/to/kubeconfig
helm list
```

### RBAC Permission Denied

```bash
# Error: forbidden: User cannot list/create/delete
# Check permissions
kubectl auth can-i create deployments
kubectl auth can-i list secrets

# Use correct service account
helm install my-release ./my-chart --set serviceAccount.name=helm-deployer
```

---

## Release State Issues

### Orphaned Release

```bash
# Release exists but resources were deleted manually
# Check release secrets
kubectl get secrets -l owner=helm,name=my-release

# Option 1: Upgrade (will recreate resources)
helm upgrade my-release ./my-chart

# Option 2: Uninstall and reinstall
helm uninstall my-release
helm install my-release ./my-chart
```

### Corrupted Release History

```bash
# View release secrets
kubectl get secrets -l owner=helm,name=my-release

# Delete corrupted release
kubectl delete secret sh.helm.release.v1.my-release.v1

# Reinstall
helm install my-release ./my-chart
```

---

## Troubleshooting Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                 TROUBLESHOOTING WORKFLOW                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Identify the Issue                                              │
│     └─ Check error message carefully                               │
│     └─ helm status my-release                                      │
│     └─ kubectl get events                                          │
│                                                                     │
│  2. Debug Templates                                                 │
│     └─ helm template --debug                                       │
│     └─ helm install --dry-run --debug                              │
│     └─ Check YAML validity                                         │
│                                                                     │
│  3. Check Resources                                                 │
│     └─ kubectl get pods                                            │
│     └─ kubectl describe pod <name>                                 │
│     └─ kubectl logs <pod>                                          │
│                                                                     │
│  4. Review Configuration                                            │
│     └─ helm get values my-release                                  │
│     └─ helm get manifest my-release                                │
│     └─ Compare with expected values                                │
│                                                                     │
│  5. Resolution                                                      │
│     └─ Fix values/templates                                        │
│     └─ Upgrade or rollback                                         │
│     └─ If needed, uninstall and reinstall                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Error Messages

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMMON ERRORS AND SOLUTIONS                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Error: UPGRADE FAILED: another operation in progress              │
│  → Wait or rollback, then retry                                    │
│                                                                     │
│  Error: cannot re-use a name that is still in use                  │
│  → Uninstall existing release first                                │
│                                                                     │
│  Error: release has no deployed releases                           │
│  → First installation failed, check events                         │
│                                                                     │
│  Error: timed out waiting for the condition                        │
│  → Increase --timeout, check pod status                            │
│                                                                     │
│  Error: YAML parse error                                            │
│  → Check template syntax and indentation                           │
│                                                                     │
│  Error: field is immutable                                          │
│  → Use --force or delete resource manually                         │
│                                                                     │
│  Error: forbidden: User cannot                                      │
│  → Check RBAC permissions                                          │
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
│  Key Debug Commands:                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm template --debug  │ Debug template rendering           │   │
│  │ helm status            │ Check release status               │   │
│  │ helm get manifest      │ View deployed manifests            │   │
│  │ helm get values        │ View used values                   │   │
│  │ helm history           │ View release history               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Quick Fixes:                                                       │
│  • --force: Recreate resources                                     │
│  • --no-hooks: Skip problematic hooks                              │
│  • --dry-run: Test without applying                                │
│  • --timeout: Increase wait time                                   │
│                                                                     │
│  Prevention:                                                        │
│  • Always use --dry-run first                                      │
│  • Use helm diff for upgrades                                      │
│  • Keep release history                                            │
│  • Test in staging first                                           │
│                                                                     │
│  Next: Learn about Helm with Kustomize                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
