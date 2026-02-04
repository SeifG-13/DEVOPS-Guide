# Kubernetes Jobs and CronJobs

## What is a Job?

A Job creates pods that run until successful completion.

```
┌─────────────────────────────────────────────────────────────────┐
│                       Job Concept                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Deployment/ReplicaSet              Job                        │
│   ┌─────────────────────┐           ┌─────────────────────┐    │
│   │    Running Pods     │           │   Pods Run to       │    │
│   │    (continuous)     │           │   Completion        │    │
│   │                     │           │                     │    │
│   │  Always maintain    │           │  Run once or        │    │
│   │  desired replicas   │           │  until success      │    │
│   └─────────────────────┘           └─────────────────────┘    │
│                                                                  │
│   Job Use Cases:                                                │
│   • Database migrations                                         │
│   • Batch processing                                            │
│   • One-time scripts                                            │
│   • Data exports                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Creating a Job

### Basic Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

```bash
# Create Job
kubectl apply -f job.yaml

# Check Job status
kubectl get jobs

# View Job pods
kubectl get pods -l job-name=pi-calculation

# Get Job logs
kubectl logs job/pi-calculation

# Delete Job
kubectl delete job pi-calculation
```

### Job with Multiple Completions

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 5      # Run 5 pods to completion
  parallelism: 2      # Run 2 pods at a time
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo Processing item $JOB_COMPLETION_INDEX && sleep 5"]
      restartPolicy: Never
```

```
┌─────────────────────────────────────────────────────────────────┐
│              Job Parallelism and Completions                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   completions: 5, parallelism: 2                                │
│                                                                  │
│   Time ──────────────────────────────────────────────────►      │
│                                                                  │
│   t0:  [Pod 0] [Pod 1]                                          │
│   t1:  [Done ] [Done ] [Pod 2] [Pod 3]                         │
│   t2:              [Done ] [Done ] [Pod 4]                      │
│   t3:                          [Done ]                          │
│                                                                  │
│   Total: 5 successful completions                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Job Configuration

### Completion Modes

```yaml
# Non-indexed (default)
spec:
  completions: 3
  completionMode: NonIndexed

# Indexed (each pod gets unique index)
spec:
  completions: 3
  completionMode: Indexed  # Pods get JOB_COMPLETION_INDEX env var
```

### Backoff Policy

```yaml
spec:
  backoffLimit: 6           # Max retries before marking job failed
  activeDeadlineSeconds: 600  # Max time for job to run
```

### TTL for Finished Jobs

```yaml
spec:
  ttlSecondsAfterFinished: 100  # Delete job 100s after completion
```

### Pod Failure Policy

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob
      onExitCodes:
        containerName: main
        operator: In
        values: [42]
    - action: Ignore
      onPodConditions:
      - type: DisruptionTarget
```

## Practical Job Examples

### Database Migration

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:latest
        command: ["./migrate.sh"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
      restartPolicy: OnFailure
  backoffLimit: 3
```

### Data Export Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: export-data
spec:
  template:
    spec:
      containers:
      - name: exporter
        image: data-exporter:v1
        args: ["--output", "/data/export.csv"]
        volumeMounts:
        - name: data-volume
          mountPath: /data
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: export-pvc
      restartPolicy: Never
  backoffLimit: 2
  ttlSecondsAfterFinished: 3600
```

### Parallel Processing

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-processor
spec:
  completions: 10
  parallelism: 5
  completionMode: Indexed
  template:
    spec:
      containers:
      - name: processor
        image: processor:v1
        command:
        - /bin/sh
        - -c
        - |
          echo "Processing partition $JOB_COMPLETION_INDEX"
          ./process.sh --partition=$JOB_COMPLETION_INDEX
      restartPolicy: Never
```

## CronJobs

CronJobs create Jobs on a schedule.

```
┌─────────────────────────────────────────────────────────────────┐
│                     CronJob Concept                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CronJob: backup-db                                            │
│   Schedule: "0 2 * * *" (2 AM daily)                            │
│                                                                  │
│   Day 1      Day 2      Day 3      Day 4                        │
│   02:00      02:00      02:00      02:00                        │
│     │          │          │          │                          │
│     ▼          ▼          ▼          ▼                          │
│   ┌────┐    ┌────┐    ┌────┐    ┌────┐                         │
│   │Job1│    │Job2│    │Job3│    │Job4│                         │
│   └────┘    └────┘    └────┘    └────┘                         │
│                                                                  │
│   Each Job creates Pods that run to completion                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Cron Schedule Format

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
* * * * *
```

| Schedule | Description |
|----------|-------------|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 0 * * *` | Every day at midnight |
| `0 2 * * *` | Every day at 2 AM |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 0 1 * *` | First day of every month |

### Basic CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo", "Hello from CronJob"]
          restartPolicy: OnFailure
```

### Complete CronJob Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-db
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  timeZone: "America/New_York"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backup/db-$(date +%Y%m%d).sql
            env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            - name: DB_NAME
              value: "mydb"
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
      backoffLimit: 2
```

## CronJob Configuration

### Concurrency Policy

```yaml
spec:
  concurrencyPolicy: Allow    # Allow concurrent Jobs (default)
  # concurrencyPolicy: Forbid   # Skip new Job if previous still running
  # concurrencyPolicy: Replace  # Replace current Job with new one
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  Concurrency Policies                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Allow:                                                        │
│   Time: ──[Job1]──[Job2]──[Job3]──                             │
│                 ↑ Jobs can overlap                              │
│                                                                  │
│   Forbid:                                                       │
│   Time: ──[Job1]──────────[Job2]──                             │
│                   ↑ Job2 skipped (Job1 still running)           │
│                                                                  │
│   Replace:                                                      │
│   Time: ──[Job1]──                                              │
│                └─X─[Job2]──                                     │
│                   ↑ Job1 stopped, Job2 started                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### History Limits

```yaml
spec:
  successfulJobsHistoryLimit: 3  # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1      # Keep last 1 failed job
```

### Starting Deadline

```yaml
spec:
  startingDeadlineSeconds: 200  # Must start within 200s of schedule
```

### Suspend CronJob

```yaml
spec:
  suspend: true  # Temporarily pause scheduling
```

```bash
# Suspend via kubectl
kubectl patch cronjob backup-db -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob backup-db -p '{"spec":{"suspend":false}}'
```

## Managing Jobs and CronJobs

```bash
# Jobs
kubectl get jobs
kubectl describe job pi-calculation
kubectl logs job/pi-calculation
kubectl delete job pi-calculation

# CronJobs
kubectl get cronjobs
kubectl get cj
kubectl describe cronjob backup-db
kubectl delete cronjob backup-db

# Create Job from CronJob (run now)
kubectl create job manual-backup --from=cronjob/backup-db

# View CronJob's Jobs
kubectl get jobs -l job-name=backup-db

# Suspend CronJob
kubectl patch cronjob backup-db -p '{"spec":{"suspend":true}}'
```

## Quick Reference

### Job Fields

| Field | Description |
|-------|-------------|
| `completions` | Total successful pods needed |
| `parallelism` | Max concurrent pods |
| `backoffLimit` | Max retries |
| `activeDeadlineSeconds` | Max job duration |
| `ttlSecondsAfterFinished` | Auto-delete delay |

### CronJob Fields

| Field | Description |
|-------|-------------|
| `schedule` | Cron expression |
| `timeZone` | Timezone for schedule |
| `concurrencyPolicy` | Allow/Forbid/Replace |
| `suspend` | Pause scheduling |
| `successfulJobsHistoryLimit` | Successful jobs to keep |
| `failedJobsHistoryLimit` | Failed jobs to keep |

### Pod Restart Policies

| Policy | Use |
|--------|-----|
| `Never` | Job - pod never restarts |
| `OnFailure` | Job - restart container on failure |
| `Always` | Not allowed for Jobs |

### Common Commands

| Command | Description |
|---------|-------------|
| `kubectl get jobs` | List jobs |
| `kubectl get cj` | List cronjobs |
| `kubectl create job --from=cronjob/name` | Run cronjob now |
| `kubectl logs job/name` | View job logs |

---

**Previous:** [14-daemonsets.md](14-daemonsets.md) | **Next:** [16-ingress.md](16-ingress.md)
