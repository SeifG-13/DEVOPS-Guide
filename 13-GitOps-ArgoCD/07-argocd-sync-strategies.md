# ArgoCD Sync Strategies

## Understanding Sync in ArgoCD

Sync is the process of applying the desired state (from Git) to the actual state (in the cluster).

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNC CONCEPT                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐                           ┌─────────────┐         │
│  │   DESIRED   │         SYNC              │   ACTUAL    │         │
│  │   STATE     │────────────────────────▶  │   STATE     │         │
│  │   (Git)     │   Apply manifests         │  (Cluster)  │         │
│  └─────────────┘                           └─────────────┘         │
│                                                                     │
│  Sync Process:                                                      │
│  1. Fetch manifests from Git                                       │
│  2. Compare with live cluster state                                │
│  3. Calculate differences (diff)                                   │
│  4. Apply changes to cluster                                       │
│  5. Monitor health status                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Manual vs Automated Sync

### Manual Sync Policy

```yaml
spec:
  syncPolicy: {}  # Empty or omitted = manual sync
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MANUAL SYNC                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Characteristics:                                                   │
│  • User must trigger sync manually                                 │
│  • More control over when changes are applied                      │
│  • Suitable for production environments                            │
│  • Requires human intervention                                     │
│                                                                     │
│  Trigger sync:                                                      │
│  • Via UI: Click "SYNC" button                                     │
│  • Via CLI: argocd app sync my-app                                 │
│  • Via API: POST /api/v1/applications/{name}/sync                  │
│                                                                     │
│  Use cases:                                                         │
│  • Production deployments requiring approval                       │
│  • Sensitive environments                                          │
│  • Scheduled maintenance windows                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Automated Sync Policy

```yaml
spec:
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Revert manual cluster changes
      allowEmpty: false # Don't sync if no resources
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTOMATED SYNC                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Characteristics:                                                   │
│  • ArgoCD automatically syncs when Git changes                     │
│  • True GitOps: Git is always the source of truth                  │
│  • Faster deployments                                              │
│  • Requires trust in CI/CD pipeline                                │
│                                                                     │
│  Automated Sync Options:                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ prune      │ Delete resources removed from Git            │    │
│  │ selfHeal   │ Revert manual changes in cluster             │    │
│  │ allowEmpty │ Sync even if no resources generated          │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Use cases:                                                         │
│  • Development environments                                        │
│  • Staging environments                                            │
│  • Microservices with frequent updates                             │
│  • Environments with strong CI/CD pipelines                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Prune and Self-Heal

### Prune

```yaml
spec:
  syncPolicy:
    automated:
      prune: true  # Enable pruning
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRUNING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  WITHOUT PRUNE:                                                     │
│  ┌─────────────┐         ┌─────────────┐                           │
│  │ Git         │         │ Cluster     │                           │
│  │ - app-v2    │  sync   │ - app-v2    │                           │
│  │             │────────▶│ - app-v1    │ ← Orphaned!               │
│  └─────────────┘         └─────────────┘                           │
│                                                                     │
│  WITH PRUNE:                                                        │
│  ┌─────────────┐         ┌─────────────┐                           │
│  │ Git         │         │ Cluster     │                           │
│  │ - app-v2    │  sync   │ - app-v2    │                           │
│  │             │────────▶│             │ ← app-v1 deleted          │
│  └─────────────┘         └─────────────┘                           │
│                                                                     │
│  Prune deletes resources that:                                      │
│  • Were previously managed by ArgoCD                               │
│  • No longer exist in Git                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Self-Heal

```yaml
spec:
  syncPolicy:
    automated:
      selfHeal: true  # Enable self-healing
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SELF-HEAL                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Scenario: Someone manually changes cluster                         │
│                                                                     │
│  WITHOUT SELF-HEAL:                                                 │
│  ┌─────────────┐         ┌─────────────┐                           │
│  │ Git         │         │ Cluster     │                           │
│  │ replicas: 3 │         │ replicas: 5 │ ← Manual change stays     │
│  └─────────────┘         └─────────────┘                           │
│  Status: OutOfSync (drift detected but not corrected)              │
│                                                                     │
│  WITH SELF-HEAL:                                                    │
│  ┌─────────────┐         ┌─────────────┐                           │
│  │ Git         │  heal   │ Cluster     │                           │
│  │ replicas: 3 │────────▶│ replicas: 3 │ ← Reverted to Git state  │
│  └─────────────┘         └─────────────┘                           │
│  Status: Synced (automatic drift correction)                       │
│                                                                     │
│  Self-heal timing: Default is 5 seconds after drift detected       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Sync Options

### Available Sync Options

```yaml
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
      - PrunePropagationPolicy=foreground
      - Replace=true
      - ServerSideApply=true
      - FailOnSharedResource=true
      - RespectIgnoreDifferences=true
```

### Sync Options Explained

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNC OPTIONS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CreateNamespace=true                                               │
│  ─────────────────────                                              │
│  Automatically create namespace if it doesn't exist                │
│                                                                     │
│  PruneLast=true                                                     │
│  ─────────────────                                                  │
│  Prune resources after all other sync operations complete          │
│  Useful for dependencies (delete old after new is ready)           │
│                                                                     │
│  ApplyOutOfSyncOnly=true                                            │
│  ────────────────────────                                           │
│  Only sync resources that are out of sync (faster syncs)           │
│                                                                     │
│  PrunePropagationPolicy=foreground|background|orphan                │
│  ───────────────────────────────────────────────────                │
│  • foreground: Wait for dependents to be deleted                   │
│  • background: Delete immediately, GC cleans dependents            │
│  • orphan: Leave dependents (not deleted)                          │
│                                                                     │
│  Replace=true                                                       │
│  ─────────────                                                      │
│  Use kubectl replace instead of apply (destructive)                │
│  Needed for immutable fields                                       │
│                                                                     │
│  ServerSideApply=true                                               │
│  ─────────────────────                                              │
│  Use server-side apply (K8s 1.22+)                                 │
│  Better conflict detection, field management                       │
│                                                                     │
│  FailOnSharedResource=true                                          │
│  ──────────────────────────                                         │
│  Fail sync if resource is managed by another application           │
│                                                                     │
│  RespectIgnoreDifferences=true                                      │
│  ──────────────────────────────                                     │
│  Apply ignoreDifferences during sync comparison                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Per-Resource Sync Options

```yaml
# Add annotation to specific resources
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true,PruneLast=true
```

---

## Sync Waves

Sync waves control the order in which resources are synced.

```yaml
# Resource with sync wave annotation
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # Sync first
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # Sync second (default)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # Sync third
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNC WAVES                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Sync Order (lower numbers first):                                  │
│                                                                     │
│  Wave -5    Wave -1     Wave 0      Wave 1      Wave 5             │
│    │          │           │           │           │                 │
│    ▼          ▼           ▼           ▼           ▼                 │
│  ┌────┐    ┌────┐      ┌────┐      ┌────┐      ┌────┐             │
│  │CRDs│───▶│ NS │─────▶│Conf│─────▶│Apps│─────▶│Jobs│             │
│  └────┘    └────┘      └────┘      └────┘      └────┘             │
│                                                                     │
│  Use cases:                                                         │
│  • Create CRDs before resources that use them                      │
│  • Create namespaces before resources in them                      │
│  • Create ConfigMaps/Secrets before Deployments                    │
│  • Run database migrations before app deployment                   │
│  • Run cleanup jobs after all resources deployed                   │
│                                                                     │
│  Default wave: 0 (if no annotation)                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Sync Hooks

Hooks are special resources that run at specific points during sync.

### Hook Types

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync  # Hook phase
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migration
          image: my-app:latest
          command: ["./migrate.sh"]
      restartPolicy: Never
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HOOK PHASES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ PreSync  │──▶│   Sync   │──▶│ PostSync │──▶│SyncFail  │        │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘        │
│                                                                     │
│  PreSync                                                            │
│  ────────                                                           │
│  Run BEFORE sync begins                                            │
│  • Database migrations                                             │
│  • Backup operations                                               │
│  • Pre-deployment validation                                       │
│                                                                     │
│  Sync                                                               │
│  ────                                                               │
│  Run WITH normal resources                                         │
│  • Additional setup tasks                                          │
│                                                                     │
│  PostSync                                                           │
│  ────────                                                           │
│  Run AFTER sync completes successfully                             │
│  • Notifications                                                   │
│  • Cache warming                                                   │
│  • Integration tests                                               │
│                                                                     │
│  SyncFail                                                           │
│  ────────                                                           │
│  Run ONLY if sync fails                                            │
│  • Cleanup on failure                                              │
│  • Rollback operations                                             │
│  • Alert notifications                                             │
│                                                                     │
│  Skip                                                               │
│  ────                                                               │
│  Resource is skipped during sync (for manual management)           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Hook Delete Policies

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HOOK DELETE POLICIES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  HookSucceeded                                                      │
│  ─────────────                                                      │
│  Delete hook resource after it succeeds                            │
│                                                                     │
│  HookFailed                                                         │
│  ──────────                                                         │
│  Delete hook resource after it fails                               │
│                                                                     │
│  BeforeHookCreation                                                 │
│  ─────────────────                                                  │
│  Delete existing hook before creating new one                      │
│  (Useful for Jobs that can't be updated)                           │
│                                                                     │
│  Multiple policies (comma-separated):                              │
│  argocd.argoproj.io/hook-delete-policy: HookSucceeded,HookFailed   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Complete Hook Example

```yaml
# PreSync: Database migration
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
    argocd.argoproj.io/sync-wave: "-1"
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp/migrations:v1.0
          command: ["./run-migrations.sh"]
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
      restartPolicy: Never
  backoffLimit: 3
---
# PostSync: Notify Slack
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-deployment
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: notify
          image: curlimages/curl:latest
          command:
            - curl
            - -X
            - POST
            - -d
            - '{"text":"Deployment completed successfully!"}'
            - $(SLACK_WEBHOOK_URL)
          envFrom:
            - secretRef:
                name: slack-credentials
      restartPolicy: Never
---
# SyncFail: Cleanup on failure
apiVersion: batch/v1
kind: Job
metadata:
  name: cleanup-on-failure
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: cleanup
          image: myapp/cleanup:latest
          command: ["./rollback.sh"]
      restartPolicy: Never
```

---

## Sync Windows

Control when applications can sync (maintenance windows).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  syncWindows:
    # Allow sync during business hours
    - kind: allow
      schedule: '0 9-17 * * 1-5'  # Mon-Fri 9am-5pm
      duration: 8h
      applications:
        - '*'
      namespaces:
        - '*'

    # Deny sync during weekends
    - kind: deny
      schedule: '0 0 * * 0,6'  # Sat-Sun
      duration: 24h
      applications:
        - 'production-*'

    # Manual sync only window
    - kind: allow
      schedule: '0 2 * * *'  # 2am daily
      duration: 2h
      applications:
        - 'critical-app'
      manualSync: true
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNC WINDOWS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Window Types:                                                      │
│  • allow: Permit sync during window                                │
│  • deny: Block sync during window                                  │
│                                                                     │
│  Schedule Format (Cron):                                            │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ ┌───────── minute (0 - 59)                                    │ │
│  │ │ ┌─────── hour (0 - 23)                                      │ │
│  │ │ │ ┌───── day of month (1 - 31)                              │ │
│  │ │ │ │ ┌─── month (1 - 12)                                     │ │
│  │ │ │ │ │ ┌─ day of week (0 - 6) (Sun=0)                        │ │
│  │ │ │ │ │ │                                                      │ │
│  │ * * * * *                                                      │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Options:                                                           │
│  • manualSync: Allow only manual sync during window                │
│  • applications: List of app patterns                              │
│  • namespaces: List of namespace patterns                          │
│  • clusters: List of cluster patterns                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Retry Strategy

Configure automatic retry on sync failure.

```yaml
spec:
  syncPolicy:
    retry:
      limit: 5  # Number of retries
      backoff:
        duration: 5s      # Initial backoff
        factor: 2         # Multiplier per retry
        maxDuration: 3m   # Max backoff duration
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RETRY BACKOFF                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  With duration=5s, factor=2, maxDuration=3m, limit=5:              │
│                                                                     │
│  Attempt 1: Immediate                                              │
│  Attempt 2: Wait 5s                                                │
│  Attempt 3: Wait 10s                                               │
│  Attempt 4: Wait 20s                                               │
│  Attempt 5: Wait 40s                                               │
│                                                                     │
│  Total max wait: 75 seconds (capped at maxDuration per wait)       │
│                                                                     │
│  Use cases:                                                         │
│  • Transient network issues                                        │
│  • Cluster API temporarily unavailable                             │
│  • CRD not yet installed                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Sync Strategies Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNC STRATEGY PATTERNS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Development Environment:                                           │
│  syncPolicy:                                                        │
│    automated:                                                       │
│      prune: true                                                   │
│      selfHeal: true                                                │
│    syncOptions:                                                     │
│      - CreateNamespace=true                                        │
│                                                                     │
│  Staging Environment:                                               │
│  syncPolicy:                                                        │
│    automated:                                                       │
│      prune: true                                                   │
│      selfHeal: true                                                │
│    retry:                                                          │
│      limit: 3                                                      │
│                                                                     │
│  Production Environment:                                            │
│  syncPolicy:                                                        │
│    automated:                                                       │
│      prune: false    # Manual prune                                │
│      selfHeal: true                                                │
│    syncOptions:                                                     │
│      - ApplyOutOfSyncOnly=true                                     │
│      - PruneLast=true                                              │
│                                                                     │
│  Critical Production:                                               │
│  syncPolicy: {}  # Manual sync only                                │
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
│  Sync Policy Options:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ automated.prune    │ Delete resources not in Git            │   │
│  │ automated.selfHeal │ Revert manual cluster changes          │   │
│  │ syncOptions        │ CreateNamespace, Replace, etc.         │   │
│  │ retry              │ Automatic retry on failure             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Sync Ordering:                                                     │
│  • Sync Waves: Order resources by wave number                      │
│  • Sync Hooks: PreSync, Sync, PostSync, SyncFail                   │
│                                                                     │
│  Access Control:                                                    │
│  • Sync Windows: Allow/deny sync during time periods               │
│                                                                     │
│  Best Practices:                                                    │
│  • Use automated sync for non-production                           │
│  • Enable selfHeal for drift prevention                            │
│  • Use hooks for migrations and notifications                      │
│  • Configure sync windows for production                           │
│                                                                     │
│  Next: Learn about ArgoCD Projects for RBAC                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
