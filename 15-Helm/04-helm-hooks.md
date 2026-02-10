# Helm Hooks

## Hooks Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HELM HOOKS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What are Hooks?                                                    │
│  • Resources that run at specific points in release lifecycle      │
│  • Enable actions before/after install, upgrade, delete            │
│  • Typically Jobs or Pods that perform operations                  │
│                                                                     │
│  Common Use Cases:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Database Migration  │ Run before upgrade                    │   │
│  │ Backup              │ Run before upgrade/delete             │   │
│  │ Config Validation   │ Run before install                    │   │
│  │ Cache Warmup        │ Run after install                     │   │
│  │ Cleanup             │ Run after delete                      │   │
│  │ Test                │ Run after install to verify           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Hook Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                      HOOK EXECUTION ORDER                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Install Lifecycle:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. pre-install hooks run                                    │   │
│  │ 2. Chart resources installed                                │   │
│  │ 3. post-install hooks run                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Upgrade Lifecycle:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. pre-upgrade hooks run                                    │   │
│  │ 2. Chart resources upgraded                                 │   │
│  │ 3. post-upgrade hooks run                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Delete Lifecycle:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. pre-delete hooks run                                     │   │
│  │ 2. Chart resources deleted                                  │   │
│  │ 3. post-delete hooks run                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Rollback Lifecycle:                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. pre-rollback hooks run                                   │   │
│  │ 2. Chart resources rolled back                              │   │
│  │ 3. post-rollback hooks run                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Hook Types

```yaml
# Available hook types
helm.sh/hook: pre-install      # Before resources installed
helm.sh/hook: post-install     # After resources installed
helm.sh/hook: pre-delete       # Before resources deleted
helm.sh/hook: post-delete      # After resources deleted
helm.sh/hook: pre-upgrade      # Before resources upgraded
helm.sh/hook: post-upgrade     # After resources upgraded
helm.sh/hook: pre-rollback     # Before rollback
helm.sh/hook: post-rollback    # After rollback
helm.sh/hook: test             # For helm test command

# Multiple hooks
helm.sh/hook: pre-install,pre-upgrade
```

---

## Basic Hook Examples

### Pre-Install Hook

```yaml
# templates/pre-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-pre-install
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    metadata:
      name: {{ include "myapp.fullname" . }}-pre-install
    spec:
      restartPolicy: Never
      containers:
        - name: pre-install
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - |
              echo "Running pre-install hook"
              echo "Validating configuration..."
              # Validation logic here
              echo "Pre-install complete"
```

### Database Migration Hook

```yaml
# templates/db-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-db-migrate
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  backoffLimit: 3
  template:
    metadata:
      name: {{ include "myapp.fullname" . }}-db-migrate
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
                  name: {{ include "myapp.fullname" . }}-db
                  key: url
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
```

### Backup Before Upgrade

```yaml
# templates/backup-hook.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-backup
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: backup
          image: postgres:15
          command: ["/bin/sh", "-c"]
          args:
            - |
              pg_dump $DATABASE_URL > /backup/backup-$(date +%Y%m%d%H%M%S).sql
              echo "Backup completed successfully"
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-db
                  key: url
          volumeMounts:
            - name: backup-volume
              mountPath: /backup
      volumes:
        - name: backup-volume
          persistentVolumeClaim:
            claimName: {{ include "myapp.fullname" . }}-backup
```

---

## Hook Weights

```yaml
# Hook weights control execution order
# Lower weights run first, default is 0

# Weight -10: Runs first
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-first
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-10"
# ...

# Weight 0: Runs second (default)
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-second
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
# ...

# Weight 10: Runs last
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-third
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "10"
# ...
```

---

## Hook Delete Policies

```yaml
# Hook deletion policies
# Determines when hook resources are cleaned up

# Delete before creating a new hook (recreate on each run)
"helm.sh/hook-delete-policy": before-hook-creation

# Delete after hook succeeds
"helm.sh/hook-delete-policy": hook-succeeded

# Delete after hook fails
"helm.sh/hook-delete-policy": hook-failed

# Multiple policies
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded

# No policy = hook resources are kept
```

### Delete Policy Examples

```yaml
# Always recreate, keep on failure
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  # Job will be deleted before each run, kept if failed
  # ...

# Clean up only on success
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-validation
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  # Job will be kept if failed (for debugging)
  # ...

# Clean up on both success and failure
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-temp-task
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-delete-policy": hook-succeeded,hook-failed
spec:
  # Job will always be cleaned up
  # ...
```

---

## Test Hooks

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-test-connection
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "myapp.fullname" . }}:{{ .Values.service.port }}']
---
# templates/tests/test-api.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-test-api
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "10"
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: curlimages/curl
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Testing API endpoints..."

          # Health check
          curl -f http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/health || exit 1

          # API endpoint
          curl -f http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/api/v1/status || exit 1

          echo "All tests passed!"
```

```bash
# Run tests
helm test my-release

# Run tests with logs
helm test my-release --logs

# Run tests with timeout
helm test my-release --timeout 5m
```

---

## Advanced Hook Patterns

### Conditional Hooks

```yaml
# Only create hook if enabled
{{- if .Values.migration.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: {{ .Values.migration.command | toJson }}
{{- end }}
```

### Post-Delete Cleanup

```yaml
# templates/post-delete-cleanup.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-cleanup
  annotations:
    "helm.sh/hook": post-delete
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: {{ include "myapp.fullname" . }}-cleanup
      containers:
        - name: cleanup
          image: bitnami/kubectl:latest
          command: ["/bin/sh", "-c"]
          args:
            - |
              echo "Cleaning up PVCs..."
              kubectl delete pvc -l app.kubernetes.io/instance={{ .Release.Name }} -n {{ .Release.Namespace }}

              echo "Cleaning up secrets..."
              kubectl delete secret -l app.kubernetes.io/instance={{ .Release.Name }} -n {{ .Release.Namespace }}

              echo "Cleanup complete"
```

### Init Container as Alternative

```yaml
# Sometimes init containers are better than hooks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      initContainers:
        # Wait for database
        - name: wait-for-db
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - |
              until nc -z {{ .Values.database.host }} {{ .Values.database.port }}; do
                echo "Waiting for database..."
                sleep 2
              done
              echo "Database is ready"

        # Run migrations
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate.sh"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-db
                  key: url

      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          # ...
```

---

## Hook with ServiceAccount

```yaml
# templates/hook-serviceaccount.yaml
{{- if .Values.hooks.migration.enabled }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-20"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-20"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-20"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
subjects:
  - kind: ServiceAccount
    name: {{ include "myapp.fullname" . }}-migration
    namespace: {{ .Release.Namespace }}
roleRef:
  kind: Role
  name: {{ include "myapp.fullname" . }}-migration
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      serviceAccountName: {{ include "myapp.fullname" . }}-migration
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate.sh"]
{{- end }}
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HOOK BEST PRACTICES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Design:                                                            │
│  ✓ Use hooks for one-time operations                               │
│  ✓ Consider init containers for per-pod operations                 │
│  ✓ Make hooks idempotent                                           │
│  ✓ Set appropriate timeouts                                        │
│                                                                     │
│  Error Handling:                                                    │
│  ✓ Set backoffLimit for Job hooks                                  │
│  ✓ Keep failed hooks for debugging                                 │
│  ✓ Log detailed error messages                                     │
│  ✓ Test hook failure scenarios                                     │
│                                                                     │
│  Resource Management:                                               │
│  ✓ Use delete policies appropriately                               │
│  ✓ Set resource limits on hook pods                                │
│  ✓ Clean up hook resources                                         │
│  ✓ Use weights for ordering                                        │
│                                                                     │
│  Security:                                                          │
│  ✓ Use minimal RBAC for hooks                                      │
│  ✓ Don't expose secrets in logs                                    │
│  ✓ Use dedicated service accounts                                  │
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
│  Hook Types:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ pre-install    │ Before install                             │   │
│  │ post-install   │ After install                              │   │
│  │ pre-upgrade    │ Before upgrade                             │   │
│  │ post-upgrade   │ After upgrade                              │   │
│  │ pre-delete     │ Before delete                              │   │
│  │ post-delete    │ After delete                               │   │
│  │ test           │ For helm test                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Annotations:                                                   │
│  • helm.sh/hook - Hook type                                        │
│  • helm.sh/hook-weight - Execution order                           │
│  • helm.sh/hook-delete-policy - Cleanup policy                     │
│                                                                     │
│  Common Patterns:                                                   │
│  • Database migrations (pre-upgrade)                               │
│  • Backups (pre-upgrade/delete)                                    │
│  • Validation (pre-install)                                        │
│  • Cleanup (post-delete)                                           │
│  • Testing (test)                                                  │
│                                                                     │
│  Next: Learn about Helm dependencies                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
