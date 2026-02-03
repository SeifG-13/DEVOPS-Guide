# Cron & Scheduling

## Cron Overview

Cron is a time-based job scheduler in Linux. It runs scheduled commands at specified times.

### Cron Components

| Component | Purpose |
|-----------|---------|
| `crond` | Cron daemon (service) |
| `crontab` | User cron tables |
| `/etc/crontab` | System cron table |
| `/etc/cron.d/` | Drop-in cron files |
| `/etc/cron.daily/` | Daily scripts |
| `/etc/cron.hourly/` | Hourly scripts |
| `/etc/cron.weekly/` | Weekly scripts |
| `/etc/cron.monthly/` | Monthly scripts |

## Crontab Syntax

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 7) (0 and 7 = Sunday)
│ │ │ │ │
│ │ │ │ │
* * * * * command to execute
```

### Special Characters

| Character | Meaning | Example |
|-----------|---------|---------|
| `*` | Any value | `* * * * *` (every minute) |
| `,` | Value list | `1,15,30` (at 1, 15, and 30) |
| `-` | Range | `1-5` (1 through 5) |
| `/` | Step | `*/5` (every 5) |

### Examples

```bash
# Every minute
* * * * * /script.sh

# Every 5 minutes
*/5 * * * * /script.sh

# Every hour at minute 0
0 * * * * /script.sh

# Every day at midnight
0 0 * * * /script.sh

# Every day at 2:30 AM
30 2 * * * /script.sh

# Every Monday at 9 AM
0 9 * * 1 /script.sh

# Every weekday at 6 PM
0 18 * * 1-5 /script.sh

# First day of every month at noon
0 12 1 * * /script.sh

# Every 15 minutes during business hours
*/15 9-17 * * 1-5 /script.sh

# Twice a day (8 AM and 8 PM)
0 8,20 * * * /script.sh

# Every Sunday at 4:05 AM
5 4 * * 0 /script.sh

# Every quarter (Jan, Apr, Jul, Oct) on the 1st at 6 AM
0 6 1 1,4,7,10 * /script.sh
```

### Special Strings

```bash
@reboot     # Run once at startup
@yearly     # 0 0 1 1 * (once a year)
@annually   # Same as @yearly
@monthly    # 0 0 1 * * (once a month)
@weekly     # 0 0 * * 0 (once a week)
@daily      # 0 0 * * * (once a day)
@midnight   # Same as @daily
@hourly     # 0 * * * * (once an hour)

# Examples
@reboot /home/user/startup.sh
@daily /usr/local/bin/backup.sh
@weekly /scripts/cleanup.sh
```

## Managing Crontabs

### User Crontab

```bash
# Edit current user's crontab
crontab -e

# List current user's crontab
crontab -l

# Remove current user's crontab
crontab -r

# Remove with confirmation
crontab -i -r

# Edit another user's crontab (as root)
sudo crontab -u username -e

# List another user's crontab
sudo crontab -u username -l
```

### System Crontab

```bash
# System crontab (includes user field)
sudo vim /etc/crontab

# Format (note the username field):
# min hour day month dow user command
0 5 * * * root /scripts/backup.sh

# Drop-in directory
sudo vim /etc/cron.d/myjob
```

### Periodic Directories

```bash
# Place scripts in these directories:
/etc/cron.hourly/   # Runs hourly
/etc/cron.daily/    # Runs daily
/etc/cron.weekly/   # Runs weekly
/etc/cron.monthly/  # Runs monthly

# Scripts must:
# - Be executable
# - NOT have file extensions (.sh)
# - Have proper permissions

# Example:
sudo cp myscript.sh /etc/cron.daily/myscript
sudo chmod +x /etc/cron.daily/myscript
```

## Cron Best Practices

### Environment Variables

```bash
# Cron has minimal environment
# Set variables at top of crontab:

SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=admin@example.com
HOME=/home/user

# Or in script:
#!/bin/bash
export PATH=/usr/local/bin:/usr/bin:/bin
```

### Output Handling

```bash
# Redirect output to file
* * * * * /script.sh >> /var/log/myjob.log 2>&1

# Discard output
* * * * * /script.sh > /dev/null 2>&1

# Email output (if MAILTO set)
MAILTO=admin@example.com
* * * * * /script.sh

# No email
MAILTO=""
* * * * * /script.sh
```

### Locking (Prevent Overlap)

```bash
# Using flock
* * * * * /usr/bin/flock -n /tmp/myjob.lock /path/to/script.sh

# In script:
#!/bin/bash
LOCKFILE=/tmp/myjob.lock
if [ -f "$LOCKFILE" ]; then
    exit 0
fi
trap "rm -f $LOCKFILE" EXIT
touch $LOCKFILE
# ... rest of script
```

### Logging

```bash
# Add timestamps
* * * * * /script.sh 2>&1 | while read line; do echo "$(date): $line"; done >> /var/log/myjob.log

# Or in script:
#!/bin/bash
exec >> /var/log/myjob.log 2>&1
echo "=== $(date) ==="
# ... rest of script
```

## Access Control

```bash
# Allow users to use cron
/etc/cron.allow

# Deny users from using cron
/etc/cron.deny

# If cron.allow exists: only listed users can use cron
# If cron.allow doesn't exist: users not in cron.deny can use cron
# If neither exists: only root can use cron (on some systems)
```

## at - One-Time Jobs

```bash
# Schedule command for later
at 10:00
at> /path/to/script.sh
at> Ctrl+D

# Various time formats
at 10:00 AM
at 10:00 PM
at noon
at midnight
at now + 1 hour
at now + 30 minutes
at now + 2 days
at 10:00 AM tomorrow
at 10:00 AM next week
at 10:00 Jan 15

# From file
at 10:00 -f /path/to/commands.txt

# List pending jobs
atq

# Remove job
atrm <job_number>

# View job details
at -c <job_number>
```

### at Access Control

```bash
/etc/at.allow    # Users allowed to use at
/etc/at.deny     # Users denied from using at
```

## batch - Run When System Load is Low

```bash
# Run when load average drops below 1.5
batch
batch> /path/to/script.sh
batch> Ctrl+D
```

## systemd Timers

### Timer Unit File

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=daily
# Or specific time:
# OnCalendar=*-*-* 02:30:00
# OnCalendar=Mon *-*-* 10:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

### Associated Service

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=root
```

### Timer Calendar Expressions

```bash
# Daily at midnight
OnCalendar=daily

# Weekly on Monday
OnCalendar=weekly

# Monthly on 1st
OnCalendar=monthly

# Specific time
OnCalendar=*-*-* 14:30:00

# Every 15 minutes
OnCalendar=*:0/15

# Weekdays at 9 AM
OnCalendar=Mon..Fri *-*-* 09:00:00

# Multiple times
OnCalendar=*-*-* 08:00:00
OnCalendar=*-*-* 20:00:00
```

### Managing Timers

```bash
# Enable and start timer
sudo systemctl enable --now backup.timer

# List timers
systemctl list-timers

# Check timer status
systemctl status backup.timer

# Manually trigger associated service
sudo systemctl start backup.service

# View timer logs
journalctl -u backup.service
```

## Comparison: Cron vs systemd Timers

| Feature | Cron | systemd Timers |
|---------|------|----------------|
| Dependencies | No | Yes |
| Logging | Manual | journald |
| Randomized delay | No | Yes |
| Persistent | No | Yes |
| Resource limits | No | Yes |
| Boot time execution | Limited | Full support |

## Practical Examples

### Backup Script

```bash
# /etc/cron.d/backup
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin

# Daily backup at 2 AM
0 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

### Log Rotation

```bash
# Weekly log cleanup
0 0 * * 0 root find /var/log -name "*.log" -mtime +30 -delete
```

### Health Check

```bash
# Every 5 minutes
*/5 * * * * root /usr/local/bin/health-check.sh || /usr/local/bin/alert.sh
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `crontab -e` | Edit crontab | `crontab -e` |
| `crontab -l` | List crontab | `crontab -l` |
| `crontab -r` | Remove crontab | `crontab -r` |
| `at` | Schedule one-time job | `at 10:00` |
| `atq` | List pending at jobs | `atq` |
| `atrm` | Remove at job | `atrm 5` |
| `systemctl list-timers` | List systemd timers | `systemctl list-timers` |

---

**Previous:** [18-storage.md](18-storage.md) | **Next:** [20-logs-troubleshooting.md](20-logs-troubleshooting.md)
