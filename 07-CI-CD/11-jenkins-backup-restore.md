# Jenkins Backup and Restore

## What to Backup

```
┌─────────────────────────────────────────────────────────────────┐
│                 Jenkins Backup Components                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   JENKINS_HOME (/var/lib/jenkins)                               │
│   │                                                              │
│   ├── config.xml              ← Global configuration           │
│   ├── credentials.xml         ← Credentials (encrypted)        │
│   ├── secrets/                ← Encryption keys                │
│   │   ├── master.key                                           │
│   │   ├── hudson.util.Secret                                   │
│   │   └── initialAdminPassword                                 │
│   │                                                              │
│   ├── jobs/                   ← Job configurations             │
│   │   └── my-job/                                              │
│   │       ├── config.xml                                       │
│   │       └── builds/         ← Build history                  │
│   │                                                              │
│   ├── nodes/                  ← Agent configurations           │
│   ├── plugins/                ← Installed plugins              │
│   ├── users/                  ← User configurations            │
│   └── userContent/            ← User-uploaded files            │
│                                                                  │
│   Critical to backup:                                           │
│   • config.xml, credentials.xml                                 │
│   • secrets/ directory                                         │
│   • jobs/ directory                                            │
│   • plugins/ directory                                          │
│   • nodes/ directory                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Manual Backup

### Basic Backup Script

```bash
#!/bin/bash
# backup-jenkins.sh

JENKINS_HOME="/var/lib/jenkins"
BACKUP_DIR="/backup/jenkins"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/jenkins_backup_${DATE}.tar.gz"

# Create backup directory
mkdir -p ${BACKUP_DIR}

# Stop Jenkins (optional but recommended for consistency)
# systemctl stop jenkins

# Create backup
tar -czf ${BACKUP_FILE} \
    --exclude="${JENKINS_HOME}/workspace" \
    --exclude="${JENKINS_HOME}/caches" \
    --exclude="${JENKINS_HOME}/.cache" \
    --exclude="${JENKINS_HOME}/war" \
    --exclude="${JENKINS_HOME}/logs" \
    -C $(dirname ${JENKINS_HOME}) \
    $(basename ${JENKINS_HOME})

# Start Jenkins
# systemctl start jenkins

# Clean old backups (keep last 7)
find ${BACKUP_DIR} -name "jenkins_backup_*.tar.gz" -mtime +7 -delete

echo "Backup created: ${BACKUP_FILE}"
```

### Backup Critical Files Only

```bash
#!/bin/bash
# backup-jenkins-config.sh

JENKINS_HOME="/var/lib/jenkins"
BACKUP_DIR="/backup/jenkins"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_DIR}/config_${DATE}

# Backup global config
cp ${JENKINS_HOME}/config.xml ${BACKUP_DIR}/config_${DATE}/
cp ${JENKINS_HOME}/credentials.xml ${BACKUP_DIR}/config_${DATE}/

# Backup secrets
cp -r ${JENKINS_HOME}/secrets ${BACKUP_DIR}/config_${DATE}/

# Backup job configs only (not build history)
find ${JENKINS_HOME}/jobs -name "config.xml" -exec cp --parents {} ${BACKUP_DIR}/config_${DATE}/ \;

# Backup node configs
cp -r ${JENKINS_HOME}/nodes ${BACKUP_DIR}/config_${DATE}/

# Backup user configs
cp -r ${JENKINS_HOME}/users ${BACKUP_DIR}/config_${DATE}/

# Backup plugins list
ls ${JENKINS_HOME}/plugins/*.jpi | xargs -I {} basename {} .jpi > ${BACKUP_DIR}/config_${DATE}/plugins.txt

# Create archive
tar -czf ${BACKUP_DIR}/jenkins_config_${DATE}.tar.gz -C ${BACKUP_DIR} config_${DATE}
rm -rf ${BACKUP_DIR}/config_${DATE}

echo "Config backup created: ${BACKUP_DIR}/jenkins_config_${DATE}.tar.gz"
```

## ThinBackup Plugin

### Installation and Configuration

```
Manage Jenkins → Plugins → Available → Search "ThinBackup"
Install and restart Jenkins

Manage Jenkins → ThinBackup → Settings

Backup directory: /backup/jenkins/thinbackup
Backup schedule for full backups: H 2 * * 0  (Weekly Sunday 2 AM)
Backup schedule for differential backups: H 2 * * 1-6  (Daily except Sunday)
Max number of backup sets: 10
Files excluded from backup: (leave default)
Backup build results: ✓
Backup plugins archives: ✓
Backup 'userContents': ✓
Backup next build number: ✓
Backup only builds marked to keep: □
Clean up differential backups: ✓
Move old backups to ZIP files: ✓
```

### ThinBackup Manual Operations

```
Manage Jenkins → ThinBackup

Actions:
• Backup Now - Trigger immediate backup
• Restore - Restore from backup
• Settings - Configure backup settings
```

### ThinBackup Directory Structure

```
/backup/jenkins/thinbackup/
├── FULL-2024-01-01_02-00/
│   ├── config.xml
│   ├── credentials.xml
│   ├── jobs/
│   │   └── my-job/
│   │       └── config.xml
│   ├── plugins/
│   └── ...
├── DIFF-2024-01-02_02-00/
│   └── (changed files only)
├── DIFF-2024-01-03_02-00/
└── ...
```

## Filesystem Snapshot Plugin

### Installation

```
Manage Jenkins → Plugins → Available → Search "Filesystem SCM"
Also search for "SCM Sync Configuration Plugin" (for config sync)
```

### Filesystem Snapshot for Backup

```bash
#!/bin/bash
# Filesystem snapshot backup using LVM

JENKINS_HOME="/var/lib/jenkins"
VG="vg_jenkins"
LV="lv_jenkins"
SNAPSHOT_SIZE="5G"
BACKUP_DIR="/backup/jenkins"
DATE=$(date +%Y%m%d_%H%M%S)

# Create LVM snapshot
lvcreate -L ${SNAPSHOT_SIZE} -s -n ${LV}_snap /dev/${VG}/${LV}

# Mount snapshot
mkdir -p /mnt/jenkins_snap
mount /dev/${VG}/${LV}_snap /mnt/jenkins_snap

# Backup from snapshot
tar -czf ${BACKUP_DIR}/jenkins_snapshot_${DATE}.tar.gz \
    -C /mnt/jenkins_snap .

# Cleanup
umount /mnt/jenkins_snap
lvremove -f /dev/${VG}/${LV}_snap

echo "Snapshot backup created: ${BACKUP_DIR}/jenkins_snapshot_${DATE}.tar.gz"
```

## Restore Procedures

### Full Restore

```bash
#!/bin/bash
# restore-jenkins.sh

BACKUP_FILE=$1
JENKINS_HOME="/var/lib/jenkins"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file.tar.gz>"
    exit 1
fi

# Stop Jenkins
sudo systemctl stop jenkins

# Backup current state (just in case)
sudo mv ${JENKINS_HOME} ${JENKINS_HOME}.old.$(date +%Y%m%d)

# Create fresh Jenkins home
sudo mkdir -p ${JENKINS_HOME}

# Restore from backup
sudo tar -xzf ${BACKUP_FILE} -C $(dirname ${JENKINS_HOME})

# Fix permissions
sudo chown -R jenkins:jenkins ${JENKINS_HOME}

# Start Jenkins
sudo systemctl start jenkins

echo "Restore complete. Check Jenkins status."
```

### Restore Using ThinBackup

```
Manage Jenkins → ThinBackup → Restore

1. Select backup set to restore
2. Optionally select "Restore plugins"
3. Click "Restore"
4. Restart Jenkins: Manage Jenkins → Reload Configuration from Disk
   Or: sudo systemctl restart jenkins
```

### Restore Specific Jobs

```bash
#!/bin/bash
# restore-job.sh

BACKUP_DIR="/backup/jenkins"
JOB_NAME=$1
JENKINS_HOME="/var/lib/jenkins"

# Extract job config from backup
tar -xzf ${BACKUP_DIR}/jenkins_backup_latest.tar.gz \
    -C /tmp \
    jenkins/jobs/${JOB_NAME}/config.xml

# Copy to Jenkins
sudo cp /tmp/jenkins/jobs/${JOB_NAME}/config.xml \
    ${JENKINS_HOME}/jobs/${JOB_NAME}/

sudo chown jenkins:jenkins ${JENKINS_HOME}/jobs/${JOB_NAME}/config.xml

# Reload job configuration
curl -X POST http://localhost:8080/job/${JOB_NAME}/reload

echo "Job ${JOB_NAME} restored"
```

## Automated Backup with Cron

### Setup Cron Job

```bash
# Edit crontab
sudo crontab -e

# Add backup schedule
# Full backup every Sunday at 2 AM
0 2 * * 0 /opt/scripts/backup-jenkins-full.sh >> /var/log/jenkins-backup.log 2>&1

# Incremental backup daily at 2 AM (except Sunday)
0 2 * * 1-6 /opt/scripts/backup-jenkins-incremental.sh >> /var/log/jenkins-backup.log 2>&1
```

### Backup to S3

```bash
#!/bin/bash
# backup-jenkins-s3.sh

JENKINS_HOME="/var/lib/jenkins"
BACKUP_DIR="/tmp/jenkins_backup"
DATE=$(date +%Y%m%d_%H%M%S)
S3_BUCKET="s3://my-backups/jenkins"

# Create local backup
mkdir -p ${BACKUP_DIR}
tar -czf ${BACKUP_DIR}/jenkins_${DATE}.tar.gz \
    --exclude="${JENKINS_HOME}/workspace" \
    --exclude="${JENKINS_HOME}/caches" \
    -C $(dirname ${JENKINS_HOME}) \
    $(basename ${JENKINS_HOME})

# Upload to S3
aws s3 cp ${BACKUP_DIR}/jenkins_${DATE}.tar.gz ${S3_BUCKET}/

# Cleanup local
rm -rf ${BACKUP_DIR}

# Remove old backups from S3 (keep last 30)
aws s3 ls ${S3_BUCKET}/ | sort | head -n -30 | awk '{print $4}' | \
    xargs -I {} aws s3 rm ${S3_BUCKET}/{}

echo "Backup uploaded to S3: ${S3_BUCKET}/jenkins_${DATE}.tar.gz"
```

## Disaster Recovery

### Recovery Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│                 Disaster Recovery Checklist                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Preparation                                                   │
│   □ Document Jenkins version                                    │
│   □ List all installed plugins with versions                    │
│   □ Document system configuration                               │
│   □ Test restore procedure regularly                            │
│   □ Store backups offsite                                       │
│                                                                  │
│   Recovery Steps                                                │
│   1. □ Install Jenkins (same version)                          │
│   2. □ Stop Jenkins service                                     │
│   3. □ Restore JENKINS_HOME from backup                        │
│   4. □ Restore secrets/ directory                              │
│   5. □ Fix file permissions                                     │
│   6. □ Start Jenkins                                           │
│   7. □ Verify plugins load correctly                           │
│   8. □ Verify jobs are present                                 │
│   9. □ Verify credentials work                                 │
│   10.□ Test a sample build                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Export Plugin List

```bash
# Export installed plugins
java -jar jenkins-cli.jar -s http://localhost:8080/ list-plugins | \
    awk '{print $1":"$NF}' > plugins.txt

# Install plugins from list
while read plugin; do
    java -jar jenkins-cli.jar -s http://localhost:8080/ \
        install-plugin ${plugin%:*}
done < plugins.txt
```

## Backup Verification

```bash
#!/bin/bash
# verify-backup.sh

BACKUP_FILE=$1
TEMP_DIR="/tmp/backup_verify_$$"

mkdir -p ${TEMP_DIR}

# Extract backup
tar -xzf ${BACKUP_FILE} -C ${TEMP_DIR}

# Verify critical files
CRITICAL_FILES=(
    "jenkins/config.xml"
    "jenkins/credentials.xml"
    "jenkins/secrets/master.key"
)

echo "Verifying backup: ${BACKUP_FILE}"

for file in "${CRITICAL_FILES[@]}"; do
    if [ -f "${TEMP_DIR}/${file}" ]; then
        echo "✓ ${file} exists"
    else
        echo "✗ ${file} MISSING!"
    fi
done

# Count jobs
JOB_COUNT=$(find ${TEMP_DIR}/jenkins/jobs -name "config.xml" | wc -l)
echo "Jobs found: ${JOB_COUNT}"

# Cleanup
rm -rf ${TEMP_DIR}
```

## Quick Reference

### Backup Locations

| Component | Path |
|-----------|------|
| Global config | `$JENKINS_HOME/config.xml` |
| Credentials | `$JENKINS_HOME/credentials.xml` |
| Secrets | `$JENKINS_HOME/secrets/` |
| Jobs | `$JENKINS_HOME/jobs/` |
| Plugins | `$JENKINS_HOME/plugins/` |
| Users | `$JENKINS_HOME/users/` |

### Backup Commands

| Task | Command |
|------|---------|
| Full backup | `tar -czf backup.tar.gz $JENKINS_HOME` |
| Restore | `tar -xzf backup.tar.gz -C /` |
| Reload config | `curl -X POST http://jenkins/reload` |

### ThinBackup Schedule

| Type | Cron | Description |
|------|------|-------------|
| Full | `H 2 * * 0` | Weekly Sunday |
| Diff | `H 2 * * 1-6` | Daily Mon-Sat |

---

**Previous:** [10-jenkins-build-agents.md](10-jenkins-build-agents.md) | **Next:** [12-github-actions-basics.md](12-github-actions-basics.md)
