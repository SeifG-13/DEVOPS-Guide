# Helm Upgrades and Rollbacks

## Upgrade Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM UPGRADES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What happens during upgrade:                                       │
│  • Helm compares new release with current                          │
│  • Calculates differences in resources                             │
│  • Applies changes via Kubernetes API                              │
│  • Stores release history                                          │
│                                                                     │
│  Upgrade Flow:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Pre-upgrade hooks ──▶ Resource updates ──▶ Post-upgrade     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Release History:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Revision 1  │  Revision 2  │  Revision 3  │  Current       │   │
│  │ (install)   │  (upgrade)   │  (upgrade)   │  (latest)      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Basic Upgrades

```bash
# Simple upgrade (same chart, new values)
helm upgrade my-release bitnami/nginx --set replicaCount=5

# Upgrade with values file
helm upgrade my-release bitnami/nginx -f values-production.yaml

# Upgrade to specific chart version
helm upgrade my-release bitnami/nginx --version 15.0.0

# Upgrade or install if not exists
helm upgrade --install my-release bitnami/nginx

# Upgrade from local chart
helm upgrade my-release ./my-chart

# Reuse existing values + override specific ones
helm upgrade my-release bitnami/nginx --reuse-values --set image.tag=1.22
```

---

## Upgrade Strategies

### Atomic Upgrades

```bash
# Atomic: rollback on failure
helm upgrade my-release bitnami/nginx --atomic

# Atomic with timeout
helm upgrade my-release bitnami/nginx --atomic --timeout 5m

# What --atomic does:
# 1. Attempts the upgrade
# 2. Waits for resources to be ready
# 3. If timeout or failure, rolls back to previous version
```

### Wait for Ready

```bash
# Wait for resources to be ready
helm upgrade my-release bitnami/nginx --wait

# Wait with timeout
helm upgrade my-release bitnami/nginx --wait --timeout 10m

# Wait for specific resources
helm upgrade my-release bitnami/nginx --wait-for-jobs
```

### Dry Run

```bash
# Preview upgrade (no changes applied)
helm upgrade my-release bitnami/nginx --dry-run

# Dry run with debug output
helm upgrade my-release bitnami/nginx --dry-run --debug

# Show diff (with helm-diff plugin)
helm diff upgrade my-release bitnami/nginx -f values.yaml
```

---

## Upgrade Options

```bash
# Force resource update (delete and recreate)
helm upgrade my-release bitnami/nginx --force

# Cleanup on failure
helm upgrade my-release bitnami/nginx --cleanup-on-fail

# Reset values (don't reuse)
helm upgrade my-release bitnami/nginx --reset-values

# Reset then merge new values
helm upgrade my-release bitnami/nginx --reset-then-reuse-values -f new-values.yaml

# Skip CRDs
helm upgrade my-release bitnami/nginx --skip-crds

# Disable hooks
helm upgrade my-release bitnami/nginx --no-hooks

# Create namespace if needed
helm upgrade --install my-release bitnami/nginx \
  --namespace production \
  --create-namespace

# Set release description
helm upgrade my-release bitnami/nginx --description "Updated to v15.0.0"
```

---

## Rollback Operations

### Basic Rollback

```bash
# View release history
helm history my-release

# Output:
# REVISION  STATUS      CHART           DESCRIPTION
# 1         superseded  nginx-14.0.0    Install complete
# 2         superseded  nginx-14.1.0    Upgrade complete
# 3         deployed    nginx-15.0.0    Upgrade complete

# Rollback to previous revision
helm rollback my-release

# Rollback to specific revision
helm rollback my-release 2

# Rollback with wait
helm rollback my-release 2 --wait

# Rollback with timeout
helm rollback my-release 2 --timeout 5m

# Skip hooks during rollback
helm rollback my-release 2 --no-hooks

# Cleanup on failure
helm rollback my-release 2 --cleanup-on-fail
```

### Rollback Scenarios

```bash
# After failed upgrade
helm upgrade my-release bitnami/nginx --version 15.0.0
# ERROR: upgrade failed
helm rollback my-release

# To known good version
helm history my-release
helm rollback my-release 5  # Last known working revision

# Emergency rollback
helm rollback my-release 1 --force --no-hooks
```

---

## Release History Management

```bash
# View history
helm history my-release

# View with more details
helm history my-release --max 10

# Get specific revision manifest
helm get manifest my-release --revision 2

# Get values from revision
helm get values my-release --revision 2

# Compare revisions (with helm-diff plugin)
helm diff revision my-release 2 3
```

### History Cleanup

```bash
# Limit history during upgrade (keep last 10)
helm upgrade my-release bitnami/nginx --history-max 10

# Delete release keeping history
helm uninstall my-release --keep-history

# Purge all history
helm uninstall my-release
```

---

## Upgrade Patterns

### Blue-Green Deployment

```yaml
# values-blue.yaml
deployment:
  name: app-blue
  labels:
    version: blue

# values-green.yaml
deployment:
  name: app-green
  labels:
    version: green
```

```bash
# Deploy blue
helm install app-blue ./my-app -f values-blue.yaml

# Deploy green (new version)
helm install app-green ./my-app -f values-green.yaml

# Switch traffic (update service selector)
kubectl patch svc my-app -p '{"spec":{"selector":{"version":"green"}}}'

# Cleanup old version
helm uninstall app-blue
```

### Canary Deployment

```yaml
# values-canary.yaml
replicaCount: 1
podLabels:
  track: canary

# values-stable.yaml
replicaCount: 9
podLabels:
  track: stable
```

```bash
# Deploy stable
helm install app-stable ./my-app -f values-stable.yaml

# Deploy canary (10% traffic)
helm install app-canary ./my-app -f values-canary.yaml

# If successful, promote canary
helm upgrade app-stable ./my-app -f values-canary.yaml --set replicaCount=10

# Remove canary
helm uninstall app-canary
```

---

## Handling Breaking Changes

### Pre-Upgrade Checks

```yaml
# templates/pre-upgrade-check.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-pre-upgrade
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: check
          image: bitnami/kubectl
          command: ["/bin/sh", "-c"]
          args:
            - |
              # Check API versions
              kubectl api-versions | grep -q "networking.k8s.io/v1" || exit 1

              # Check for deprecated resources
              # ... additional checks

              echo "Pre-upgrade checks passed"
```

### Migration Job

```yaml
# templates/migration.yaml
{{- if .Values.migration.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate.sh"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-db
                  key: url
{{- end }}
```

---

## Troubleshooting Upgrades

```bash
# Check release status
helm status my-release

# Get release notes
helm get notes my-release

# View all resources
helm get all my-release

# Debug failed upgrade
helm upgrade my-release bitnami/nginx --dry-run --debug

# Check Kubernetes events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check pod status
kubectl get pods -l app.kubernetes.io/instance=my-release

# Check pod logs
kubectl logs -l app.kubernetes.io/instance=my-release --tail=100

# Describe failing pod
kubectl describe pod <pod-name>
```

### Common Issues

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMMON UPGRADE ISSUES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Resource Conflicts:                                                │
│  • Immutable field changed → Use --force or recreate               │
│  • Resource already exists → Check ownership labels                │
│                                                                     │
│  Timeout Issues:                                                    │
│  • Pod not ready → Check probes, resources                         │
│  • Slow startup → Increase --timeout                               │
│                                                                     │
│  Hook Failures:                                                     │
│  • Pre-upgrade job fails → Fix and retry                           │
│  • Job not cleaned up → Manual cleanup needed                      │
│                                                                     │
│  Value Issues:                                                      │
│  • Missing required values → Check schema                          │
│  • Type mismatch → Use --set-string                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                UPGRADE/ROLLBACK BEST PRACTICES                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Before Upgrade:                                                    │
│  ✓ Review changes with --dry-run                                   │
│  ✓ Use helm diff to see differences                                │
│  ✓ Test in staging first                                           │
│  ✓ Backup if needed                                                │
│                                                                     │
│  During Upgrade:                                                    │
│  ✓ Use --atomic for production                                     │
│  ✓ Set appropriate --timeout                                       │
│  ✓ Monitor deployment progress                                     │
│  ✓ Keep release history                                            │
│                                                                     │
│  After Upgrade:                                                     │
│  ✓ Verify application health                                       │
│  ✓ Check logs for errors                                           │
│  ✓ Run smoke tests                                                 │
│  ✓ Document changes                                                │
│                                                                     │
│  Rollback Strategy:                                                 │
│  ✓ Know your rollback targets                                      │
│  ✓ Test rollback procedures                                        │
│  ✓ Have runbooks ready                                             │
│  ✓ Maintain sufficient history                                     │
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
│  Upgrade Commands:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm upgrade        │ Update release                        │   │
│  │ --atomic            │ Rollback on failure                   │   │
│  │ --wait              │ Wait for ready                        │   │
│  │ --reuse-values      │ Keep existing values                  │   │
│  │ --dry-run           │ Preview changes                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Rollback Commands:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm rollback       │ Revert to previous                    │   │
│  │ helm history        │ View revisions                        │   │
│  │ helm get manifest   │ View revision manifests               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Flags:                                                         │
│  • --atomic: Safe upgrades with auto-rollback                      │
│  • --timeout: Control wait time                                    │
│  • --force: Recreate resources                                     │
│  • --history-max: Limit stored revisions                           │
│                                                                     │
│  Next: Learn about Helm in CI/CD pipelines                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
