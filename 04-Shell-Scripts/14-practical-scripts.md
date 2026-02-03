# Practical Shell Scripts

## Backup Scripts

### Simple File Backup

```bash
#!/bin/bash
# backup-files.sh - Backup important files

set -euo pipefail

SOURCE_DIR="${1:-$HOME/documents}"
BACKUP_DIR="${2:-/backup}"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${DATE}.tar.gz"

# Ensure backup directory exists
mkdir -p "$BACKUP_DIR"

# Create backup
echo "Backing up $SOURCE_DIR..."
tar -czf "$BACKUP_DIR/$BACKUP_NAME" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

# Show result
echo "Backup created: $BACKUP_DIR/$BACKUP_NAME"
ls -lh "$BACKUP_DIR/$BACKUP_NAME"

# Remove backups older than 7 days
echo "Cleaning old backups..."
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +7 -delete
```

### Database Backup

```bash
#!/bin/bash
# db-backup.sh - MySQL database backup

set -euo pipefail

DB_HOST="${DB_HOST:-localhost}"
DB_USER="${DB_USER:-root}"
DB_PASS="${DB_PASS:-}"
DB_NAME="${1:-}"
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)

usage() {
    echo "Usage: $0 <database_name>"
    exit 1
}

[ -z "$DB_NAME" ] && usage

mkdir -p "$BACKUP_DIR"

BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

echo "Backing up database: $DB_NAME"

mysqldump -h "$DB_HOST" -u "$DB_USER" ${DB_PASS:+-p"$DB_PASS"} \
    --single-transaction \
    --routines \
    --triggers \
    "$DB_NAME" | gzip > "$BACKUP_FILE"

echo "Backup completed: $BACKUP_FILE"
ls -lh "$BACKUP_FILE"

# Keep only last 30 days
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime +30 -delete
```

### Incremental Backup with rsync

```bash
#!/bin/bash
# incremental-backup.sh - Rsync incremental backup

set -euo pipefail

SOURCE="/home"
DEST="/backup/incremental"
LATEST="$DEST/latest"
DATE=$(date +%Y%m%d_%H%M%S)
CURRENT="$DEST/$DATE"

mkdir -p "$DEST"

# Create incremental backup using hard links
if [ -d "$LATEST" ]; then
    rsync -avh --delete \
        --link-dest="$LATEST" \
        "$SOURCE/" "$CURRENT/"
else
    rsync -avh --delete \
        "$SOURCE/" "$CURRENT/"
fi

# Update latest symlink
rm -f "$LATEST"
ln -s "$CURRENT" "$LATEST"

echo "Incremental backup completed: $CURRENT"

# Remove backups older than 30 days
find "$DEST" -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;
```

## System Monitoring

### System Health Check

```bash
#!/bin/bash
# health-check.sh - System health monitoring

check_disk() {
    echo "=== Disk Usage ==="
    df -h | awk '$5+0 > 80 {print "WARNING: " $0}'
    df -h | grep -v "^Filesystem"
    echo
}

check_memory() {
    echo "=== Memory Usage ==="
    free -h
    echo

    # Check if memory usage > 90%
    mem_used=$(free | awk '/^Mem:/ {printf "%.0f", $3/$2 * 100}')
    if [ "$mem_used" -gt 90 ]; then
        echo "WARNING: Memory usage is ${mem_used}%"
    fi
    echo
}

check_cpu() {
    echo "=== CPU Load ==="
    uptime
    echo

    # Check if load average > number of CPUs
    cpus=$(nproc)
    load=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)
    if [ "$(echo "$load > $cpus" | bc)" -eq 1 ]; then
        echo "WARNING: Load average ($load) exceeds CPU count ($cpus)"
    fi
    echo
}

check_services() {
    echo "=== Service Status ==="
    services=("sshd" "nginx" "docker")

    for service in "${services[@]}"; do
        if systemctl is-active --quiet "$service" 2>/dev/null; then
            echo "[OK] $service is running"
        else
            echo "[FAIL] $service is not running"
        fi
    done
    echo
}

check_network() {
    echo "=== Network Connectivity ==="
    hosts=("8.8.8.8" "google.com")

    for host in "${hosts[@]}"; do
        if ping -c 1 -W 2 "$host" &>/dev/null; then
            echo "[OK] Can reach $host"
        else
            echo "[FAIL] Cannot reach $host"
        fi
    done
    echo
}

# Run all checks
check_disk
check_memory
check_cpu
check_services
check_network
```

### Process Monitor

```bash
#!/bin/bash
# process-monitor.sh - Monitor specific processes

PROCESS_NAME="${1:-nginx}"
CHECK_INTERVAL=5
LOG_FILE="/var/log/process-monitor.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $*" | tee -a "$LOG_FILE"
}

monitor_process() {
    while true; do
        if ! pgrep -x "$PROCESS_NAME" > /dev/null; then
            log "WARNING: $PROCESS_NAME is not running!"

            # Attempt restart
            log "Attempting to restart $PROCESS_NAME..."
            systemctl restart "$PROCESS_NAME" 2>/dev/null || \
                service "$PROCESS_NAME" restart 2>/dev/null || \
                log "ERROR: Failed to restart $PROCESS_NAME"
        fi

        # Log process stats if running
        if pgrep -x "$PROCESS_NAME" > /dev/null; then
            pid=$(pgrep -x "$PROCESS_NAME" | head -1)
            cpu=$(ps -p $pid -o %cpu= 2>/dev/null | xargs)
            mem=$(ps -p $pid -o %mem= 2>/dev/null | xargs)
            log "INFO: $PROCESS_NAME (PID: $pid) - CPU: ${cpu}%, MEM: ${mem}%"
        fi

        sleep "$CHECK_INTERVAL"
    done
}

log "Starting process monitor for: $PROCESS_NAME"
monitor_process
```

## Log Management

### Log Rotation Script

```bash
#!/bin/bash
# log-rotate.sh - Manual log rotation

LOG_DIR="/var/log/myapp"
MAX_SIZE=$((10 * 1024 * 1024))  # 10MB
MAX_FILES=5

rotate_log() {
    local log_file="$1"
    local base_name=$(basename "$log_file")

    # Remove oldest
    rm -f "${log_file}.${MAX_FILES}.gz"

    # Rotate existing
    for ((i=MAX_FILES-1; i>=1; i--)); do
        if [ -f "${log_file}.${i}.gz" ]; then
            mv "${log_file}.${i}.gz" "${log_file}.$((i+1)).gz"
        fi
    done

    # Compress current and start new
    if [ -f "$log_file" ]; then
        gzip -c "$log_file" > "${log_file}.1.gz"
        > "$log_file"
        echo "Rotated: $log_file"
    fi
}

# Find and rotate large logs
find "$LOG_DIR" -name "*.log" -type f | while read -r log; do
    size=$(stat -c%s "$log" 2>/dev/null || stat -f%z "$log" 2>/dev/null)
    if [ "$size" -gt "$MAX_SIZE" ]; then
        rotate_log "$log"
    fi
done
```

### Log Analysis Script

```bash
#!/bin/bash
# log-analyze.sh - Analyze web server logs

LOG_FILE="${1:-/var/log/nginx/access.log}"

[ ! -f "$LOG_FILE" ] && echo "File not found: $LOG_FILE" && exit 1

echo "=== Log Analysis: $LOG_FILE ==="
echo "Generated: $(date)"
echo

echo "--- Request Summary ---"
echo "Total requests: $(wc -l < "$LOG_FILE")"
echo

echo "--- Top 10 IPs ---"
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10
echo

echo "--- HTTP Status Codes ---"
awk '{print $9}' "$LOG_FILE" | sort | uniq -c | sort -rn
echo

echo "--- Top 10 Requested URLs ---"
awk '{print $7}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10
echo

echo "--- Requests per Hour ---"
awk '{print substr($4, 2, 14)}' "$LOG_FILE" | sort | uniq -c
echo

echo "--- Top User Agents ---"
awk -F'"' '{print $6}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -5
```

## Automated Tasks

### Deployment Script

```bash
#!/bin/bash
# deploy.sh - Simple deployment script

set -euo pipefail

APP_NAME="myapp"
DEPLOY_DIR="/var/www/$APP_NAME"
REPO_URL="git@github.com:user/$APP_NAME.git"
BRANCH="${1:-main}"
BACKUP_DIR="/backup/deployments"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

backup_current() {
    if [ -d "$DEPLOY_DIR" ]; then
        local backup_name="${APP_NAME}_$(date +%Y%m%d_%H%M%S).tar.gz"
        log "Backing up current deployment..."
        tar -czf "$BACKUP_DIR/$backup_name" -C "$(dirname "$DEPLOY_DIR")" "$(basename "$DEPLOY_DIR")"
        log "Backup created: $BACKUP_DIR/$backup_name"
    fi
}

deploy() {
    log "Starting deployment of $APP_NAME from branch: $BRANCH"

    # Create backup
    mkdir -p "$BACKUP_DIR"
    backup_current

    # Clone/pull repository
    if [ -d "$DEPLOY_DIR/.git" ]; then
        log "Updating existing deployment..."
        cd "$DEPLOY_DIR"
        git fetch origin
        git checkout "$BRANCH"
        git pull origin "$BRANCH"
    else
        log "Fresh deployment..."
        rm -rf "$DEPLOY_DIR"
        git clone -b "$BRANCH" "$REPO_URL" "$DEPLOY_DIR"
        cd "$DEPLOY_DIR"
    fi

    # Install dependencies
    if [ -f "package.json" ]; then
        log "Installing Node.js dependencies..."
        npm ci --production
    fi

    if [ -f "requirements.txt" ]; then
        log "Installing Python dependencies..."
        pip install -r requirements.txt
    fi

    # Run migrations if needed
    if [ -f "migrate.sh" ]; then
        log "Running migrations..."
        ./migrate.sh
    fi

    # Restart service
    log "Restarting service..."
    systemctl restart "$APP_NAME" || service "$APP_NAME" restart

    log "Deployment completed successfully!"
}

# Rollback function
rollback() {
    local backup=$(ls -t "$BACKUP_DIR"/${APP_NAME}_*.tar.gz 2>/dev/null | head -1)

    if [ -z "$backup" ]; then
        log "No backup found for rollback"
        exit 1
    fi

    log "Rolling back to: $backup"
    rm -rf "$DEPLOY_DIR"
    mkdir -p "$DEPLOY_DIR"
    tar -xzf "$backup" -C "$(dirname "$DEPLOY_DIR")"
    systemctl restart "$APP_NAME"
    log "Rollback completed"
}

case "${2:-deploy}" in
    deploy)   deploy ;;
    rollback) rollback ;;
    *)        echo "Usage: $0 [branch] [deploy|rollback]" ;;
esac
```

### Cron Job Manager

```bash
#!/bin/bash
# cron-manager.sh - Manage cron jobs

CRON_DIR="/etc/cron.d"
APP_NAME="myapp"
CRON_FILE="$CRON_DIR/$APP_NAME"

add_job() {
    local schedule="$1"
    local command="$2"
    local job_line="$schedule root $command"

    if grep -qF "$command" "$CRON_FILE" 2>/dev/null; then
        echo "Job already exists"
        return 1
    fi

    echo "$job_line" >> "$CRON_FILE"
    echo "Job added: $job_line"
}

remove_job() {
    local pattern="$1"

    if [ -f "$CRON_FILE" ]; then
        grep -v "$pattern" "$CRON_FILE" > "$CRON_FILE.tmp"
        mv "$CRON_FILE.tmp" "$CRON_FILE"
        echo "Job(s) matching '$pattern' removed"
    fi
}

list_jobs() {
    if [ -f "$CRON_FILE" ]; then
        echo "=== Current cron jobs for $APP_NAME ==="
        cat "$CRON_FILE"
    else
        echo "No cron jobs configured"
    fi
}

case "$1" in
    add)
        [ $# -lt 3 ] && echo "Usage: $0 add 'schedule' 'command'" && exit 1
        add_job "$2" "$3"
        ;;
    remove)
        [ -z "$2" ] && echo "Usage: $0 remove 'pattern'" && exit 1
        remove_job "$2"
        ;;
    list)
        list_jobs
        ;;
    *)
        echo "Usage: $0 {add|remove|list}"
        ;;
esac
```

## Menu-Driven Scripts

### Interactive Menu

```bash
#!/bin/bash
# menu.sh - Interactive menu system

show_menu() {
    clear
    echo "========================================"
    echo "       System Administration Menu       "
    echo "========================================"
    echo "1. Show System Information"
    echo "2. Show Disk Usage"
    echo "3. Show Memory Usage"
    echo "4. Show Running Processes"
    echo "5. Show Network Connections"
    echo "6. Manage Services"
    echo "7. User Management"
    echo "8. Exit"
    echo "========================================"
}

show_system_info() {
    echo "=== System Information ==="
    echo "Hostname: $(hostname)"
    echo "OS: $(uname -s)"
    echo "Kernel: $(uname -r)"
    echo "Architecture: $(uname -m)"
    echo "Uptime: $(uptime -p 2>/dev/null || uptime)"
    read -p "Press Enter to continue..."
}

manage_services() {
    echo "=== Service Management ==="
    echo "1. List all services"
    echo "2. Start a service"
    echo "3. Stop a service"
    echo "4. Restart a service"
    echo "5. Back to main menu"

    read -p "Select option: " opt
    case $opt in
        1) systemctl list-units --type=service ;;
        2) read -p "Service name: " svc; systemctl start "$svc" ;;
        3) read -p "Service name: " svc; systemctl stop "$svc" ;;
        4) read -p "Service name: " svc; systemctl restart "$svc" ;;
        5) return ;;
    esac
    read -p "Press Enter to continue..."
}

user_management() {
    echo "=== User Management ==="
    echo "1. List users"
    echo "2. Add user"
    echo "3. Delete user"
    echo "4. Change password"
    echo "5. Back to main menu"

    read -p "Select option: " opt
    case $opt in
        1) cut -d: -f1 /etc/passwd ;;
        2) read -p "Username: " user; useradd "$user" ;;
        3) read -p "Username: " user; userdel "$user" ;;
        4) read -p "Username: " user; passwd "$user" ;;
        5) return ;;
    esac
    read -p "Press Enter to continue..."
}

# Main loop
while true; do
    show_menu
    read -p "Select option [1-8]: " choice

    case $choice in
        1) show_system_info ;;
        2) df -h; read -p "Press Enter..." ;;
        3) free -h; read -p "Press Enter..." ;;
        4) ps aux | head -20; read -p "Press Enter..." ;;
        5) netstat -tuln 2>/dev/null || ss -tuln; read -p "Press Enter..." ;;
        6) manage_services ;;
        7) user_management ;;
        8) echo "Goodbye!"; exit 0 ;;
        *) echo "Invalid option" ;;
    esac
done
```

## Utility Scripts

### File Organizer

```bash
#!/bin/bash
# file-organizer.sh - Organize files by type

SOURCE_DIR="${1:-.}"
ORGANIZE_BY="${2:-extension}"  # extension or date

organize_by_extension() {
    local dir="$1"

    find "$dir" -maxdepth 1 -type f | while read -r file; do
        ext="${file##*.}"
        [ "$ext" = "$file" ] && ext="no_extension"

        mkdir -p "$dir/$ext"
        mv "$file" "$dir/$ext/"
        echo "Moved: $file -> $dir/$ext/"
    done
}

organize_by_date() {
    local dir="$1"

    find "$dir" -maxdepth 1 -type f | while read -r file; do
        date_dir=$(date -r "$file" +%Y/%m 2>/dev/null || stat -c %y "$file" | cut -d' ' -f1 | cut -d'-' -f1,2 | tr '-' '/')

        mkdir -p "$dir/$date_dir"
        mv "$file" "$dir/$date_dir/"
        echo "Moved: $file -> $dir/$date_dir/"
    done
}

case "$ORGANIZE_BY" in
    extension) organize_by_extension "$SOURCE_DIR" ;;
    date) organize_by_date "$SOURCE_DIR" ;;
    *) echo "Usage: $0 [directory] [extension|date]" ;;
esac
```

### Bulk File Renamer

```bash
#!/bin/bash
# bulk-rename.sh - Bulk rename files

PATTERN="${1:-}"
REPLACEMENT="${2:-}"
DIR="${3:-.}"
DRY_RUN="${DRY_RUN:-true}"

usage() {
    cat << EOF
Usage: $0 <pattern> <replacement> [directory]

Environment:
    DRY_RUN=false   Actually rename files (default: true)

Examples:
    $0 .jpeg .jpg                  # Replace .jpeg with .jpg
    $0 "photo_" "vacation_" ./pics # Rename prefix
    DRY_RUN=false $0 .jpeg .jpg    # Actually perform rename
EOF
    exit 1
}

[ -z "$PATTERN" ] || [ -z "$REPLACEMENT" ] && usage

find "$DIR" -maxdepth 1 -type f -name "*${PATTERN}*" | while read -r file; do
    dir=$(dirname "$file")
    base=$(basename "$file")
    new_name="${base//$PATTERN/$REPLACEMENT}"

    if [ "$DRY_RUN" = "true" ]; then
        echo "[DRY RUN] Would rename: $file -> $dir/$new_name"
    else
        mv "$file" "$dir/$new_name"
        echo "Renamed: $file -> $dir/$new_name"
    fi
done
```

## Quick Reference

| Script Type | Key Features |
|-------------|--------------|
| Backup | tar, gzip, rsync, find for cleanup |
| Monitoring | loops, conditionals, systemctl |
| Log Management | awk, sed, grep, rotation |
| Deployment | git, npm/pip, service restart |
| Menu | while loop, case statement, read |
| Utilities | find, file operations, text processing |

---

**Previous:** [13-process-management.md](13-process-management.md) | **Next:** [15-best-practices.md](15-best-practices.md)
