# Service Management with systemd

## What is systemd?

systemd is the init system and service manager for modern Linux distributions. It manages:
- System startup and services
- Device management
- Logging
- User sessions
- Timers (cron replacement)

## systemctl - Main Command

### Service Control

```bash
# Start service
sudo systemctl start nginx

# Stop service
sudo systemctl stop nginx

# Restart service
sudo systemctl restart nginx

# Reload config without restart
sudo systemctl reload nginx

# Reload or restart
sudo systemctl reload-or-restart nginx

# Check status
systemctl status nginx

# Enable at boot
sudo systemctl enable nginx

# Disable at boot
sudo systemctl disable nginx

# Enable and start
sudo systemctl enable --now nginx

# Disable and stop
sudo systemctl disable --now nginx

# Check if enabled
systemctl is-enabled nginx

# Check if active
systemctl is-active nginx
```

### Listing Services

```bash
# List all units
systemctl list-units

# List all services
systemctl list-units --type=service

# List running services
systemctl list-units --type=service --state=running

# List failed services
systemctl list-units --state=failed
systemctl --failed

# List all installed unit files
systemctl list-unit-files

# List enabled services
systemctl list-unit-files --state=enabled
```

### Service Information

```bash
# Show service status
systemctl status nginx

# Show service properties
systemctl show nginx

# Show specific property
systemctl show nginx -p MainPID
systemctl show nginx -p ActiveState

# Show dependencies
systemctl list-dependencies nginx
systemctl list-dependencies --reverse nginx

# Show what wants to start the unit
systemctl list-dependencies --reverse --all nginx
```

## Unit Files

### Unit File Locations

| Location | Purpose |
|----------|---------|
| `/lib/systemd/system/` | Installed by packages |
| `/etc/systemd/system/` | Administrator-created (highest priority) |
| `/run/systemd/system/` | Runtime units |

### Unit File Structure

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=My Application
Documentation=https://docs.myapp.com
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Unit Sections

#### [Unit] Section

```ini
[Unit]
Description=Service description
Documentation=man:nginx(8)
After=network.target            # Start after these units
Before=other.service            # Start before these units
Requires=dependency.service     # Hard dependency
Wants=optional.service          # Soft dependency
Conflicts=conflicting.service   # Cannot run together
```

#### [Service] Section

```ini
[Service]
Type=simple                     # Main process type
ExecStart=/path/to/command      # Start command
ExecStartPre=/path/to/pre       # Before ExecStart
ExecStartPost=/path/to/post     # After ExecStart
ExecReload=/path/to/reload      # Reload command
ExecStop=/path/to/stop          # Stop command
Restart=on-failure              # Restart policy
RestartSec=10                   # Delay before restart
User=username                   # Run as user
Group=groupname                 # Run as group
WorkingDirectory=/path          # Working directory
Environment=VAR=value           # Environment variables
EnvironmentFile=/path/to/env    # Environment file
```

#### Service Types

| Type | Description |
|------|-------------|
| `simple` | Default. ExecStart is main process |
| `forking` | Process forks. Parent exits, child continues |
| `oneshot` | Process exits after completion |
| `notify` | Like simple, but sends notification when ready |
| `dbus` | Like simple, but acquires D-Bus name |
| `idle` | Like simple, but delayed until all jobs done |

#### Restart Policies

| Policy | Description |
|--------|-------------|
| `no` | Never restart |
| `on-success` | Restart on clean exit |
| `on-failure` | Restart on non-zero exit, signal, timeout |
| `on-abnormal` | Restart on signal, timeout |
| `on-watchdog` | Restart on watchdog timeout |
| `on-abort` | Restart on unclean signal |
| `always` | Always restart |

#### [Install] Section

```ini
[Install]
WantedBy=multi-user.target      # Enable in this target
RequiredBy=other.service        # Required by other service
Alias=myapp.service             # Alias name
```

### Creating Custom Service

```bash
# Create service file
sudo vim /etc/systemd/system/myapp.service

# Example: Node.js application
[Unit]
Description=My Node.js App
After=network.target

[Service]
Type=simple
User=node
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/node /var/www/myapp/app.js
Restart=on-failure
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target

# Reload daemon
sudo systemctl daemon-reload

# Enable and start
sudo systemctl enable --now myapp
```

### Modifying Existing Services

```bash
# Create override directory
sudo systemctl edit nginx

# This creates:
# /etc/systemd/system/nginx.service.d/override.conf

# Add your overrides
[Service]
LimitNOFILE=65536

# Reload
sudo systemctl daemon-reload
sudo systemctl restart nginx

# View effective configuration
systemctl cat nginx
```

## journalctl - View Logs

```bash
# View all logs
journalctl

# View boot logs
journalctl -b
journalctl -b -1         # Previous boot

# View service logs
journalctl -u nginx

# Follow logs (like tail -f)
journalctl -f
journalctl -u nginx -f

# Since time
journalctl --since "2024-01-01"
journalctl --since "1 hour ago"
journalctl --since "today"

# Until time
journalctl --until "2024-01-01 12:00"

# Priority level
journalctl -p err        # Errors and above
journalctl -p warning    # Warnings and above

# Show recent entries
journalctl -n 50         # Last 50 lines

# Output formats
journalctl -o json       # JSON format
journalctl -o short      # Default
journalctl -o verbose    # Full details

# Disk usage
journalctl --disk-usage

# Clean old logs
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=100M
```

### Log Priorities

| Priority | Level |
|----------|-------|
| 0 | emerg |
| 1 | alert |
| 2 | crit |
| 3 | err |
| 4 | warning |
| 5 | notice |
| 6 | info |
| 7 | debug |

## Targets (Runlevels)

```bash
# View current target
systemctl get-default

# Set default target
sudo systemctl set-default multi-user.target
sudo systemctl set-default graphical.target

# Switch target immediately
sudo systemctl isolate multi-user.target
sudo systemctl isolate rescue.target

# List targets
systemctl list-units --type=target

# Common targets
# poweroff.target    - Shutdown
# rescue.target      - Single user mode
# multi-user.target  - CLI multi-user
# graphical.target   - GUI
# reboot.target      - Reboot
```

## Timers (Cron Alternative)

### Timer Unit File

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=daily
# Or specific time:
# OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

### Associated Service

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

### Managing Timers

```bash
# List timers
systemctl list-timers

# Enable timer
sudo systemctl enable --now backup.timer

# Check timer status
systemctl status backup.timer
```

## System Control

```bash
# Reboot
sudo systemctl reboot

# Shutdown
sudo systemctl poweroff

# Suspend
sudo systemctl suspend

# Hibernate
sudo systemctl hibernate

# Halt
sudo systemctl halt
```

## Analyzing Boot

```bash
# Boot time analysis
systemd-analyze

# Blame (slow services)
systemd-analyze blame

# Critical chain
systemd-analyze critical-chain

# Plot boot sequence
systemd-analyze plot > boot.svg
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `systemctl start service` | Start service |
| `systemctl stop service` | Stop service |
| `systemctl restart service` | Restart service |
| `systemctl reload service` | Reload config |
| `systemctl status service` | Check status |
| `systemctl enable service` | Enable at boot |
| `systemctl disable service` | Disable at boot |
| `systemctl is-active service` | Check if running |
| `systemctl is-enabled service` | Check if enabled |
| `systemctl list-units` | List all units |
| `systemctl daemon-reload` | Reload unit files |
| `journalctl -u service` | View service logs |
| `journalctl -f` | Follow logs |

---

**Previous:** [13-process-management.md](13-process-management.md) | **Next:** [15-package-management.md](15-package-management.md)
