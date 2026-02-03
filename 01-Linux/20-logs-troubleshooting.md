# Logs & Troubleshooting

## Log Locations

### Important Log Files

| Log File | Purpose |
|----------|---------|
| `/var/log/syslog` | General system log (Debian/Ubuntu) |
| `/var/log/messages` | General system log (RHEL/CentOS) |
| `/var/log/auth.log` | Authentication logs (Debian/Ubuntu) |
| `/var/log/secure` | Authentication logs (RHEL/CentOS) |
| `/var/log/kern.log` | Kernel logs |
| `/var/log/dmesg` | Kernel ring buffer |
| `/var/log/boot.log` | Boot messages |
| `/var/log/cron` | Cron job logs |
| `/var/log/maillog` | Mail server logs |
| `/var/log/httpd/` | Apache logs (RHEL) |
| `/var/log/apache2/` | Apache logs (Debian) |
| `/var/log/nginx/` | Nginx logs |
| `/var/log/mysql/` | MySQL logs |
| `/var/log/audit/` | SELinux/audit logs |
| `/var/log/faillog` | Failed login attempts |
| `/var/log/lastlog` | Last login info |
| `/var/log/wtmp` | Login history |
| `/var/log/btmp` | Bad login attempts |

## Viewing Logs

### Basic Log Viewing

```bash
# View entire log
cat /var/log/syslog

# Page through log
less /var/log/syslog

# View last lines
tail /var/log/syslog
tail -n 100 /var/log/syslog

# Follow log in real-time
tail -f /var/log/syslog

# Follow multiple logs
tail -f /var/log/syslog /var/log/auth.log

# View first lines
head -n 50 /var/log/syslog
```

### Search in Logs

```bash
# Search for pattern
grep "error" /var/log/syslog
grep -i "error" /var/log/syslog   # Case insensitive

# Search with context
grep -A 5 "error" /var/log/syslog   # 5 lines after
grep -B 5 "error" /var/log/syslog   # 5 lines before
grep -C 5 "error" /var/log/syslog   # 5 lines around

# Search in all logs
grep -r "error" /var/log/

# Count occurrences
grep -c "error" /var/log/syslog

# Search multiple patterns
grep -E "error|warning|critical" /var/log/syslog
```

### Filter by Time

```bash
# View logs from specific time
awk '/Mar 15 10:/' /var/log/syslog

# Between times
awk '/Mar 15 10:/,/Mar 15 11:/' /var/log/syslog

# Last hour (requires timestamps)
grep "$(date -d '1 hour ago' '+%b %d %H')" /var/log/syslog
```

## journalctl (systemd)

### Basic Usage

```bash
# All logs
journalctl

# Follow (like tail -f)
journalctl -f

# Reverse order (newest first)
journalctl -r

# No pager (direct output)
journalctl --no-pager

# Last N lines
journalctl -n 50
```

### Filter by Unit/Service

```bash
# Specific service
journalctl -u nginx
journalctl -u sshd

# Multiple services
journalctl -u nginx -u php-fpm

# Follow service logs
journalctl -u nginx -f
```

### Filter by Time

```bash
# Since boot
journalctl -b
journalctl -b -1     # Previous boot

# Since time
journalctl --since "2024-01-15"
journalctl --since "2024-01-15 10:00:00"
journalctl --since "1 hour ago"
journalctl --since "today"
journalctl --since "yesterday"

# Until time
journalctl --until "2024-01-15 12:00:00"

# Time range
journalctl --since "2024-01-15 10:00" --until "2024-01-15 12:00"
```

### Filter by Priority

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

```bash
# Errors only
journalctl -p err

# Warnings and above
journalctl -p warning

# Specific priority
journalctl -p 3

# Range
journalctl -p 0..4
```

### Other Filters

```bash
# By PID
journalctl _PID=1234

# By user
journalctl _UID=1000

# By executable
journalctl /usr/bin/nginx

# Kernel messages
journalctl -k
journalctl --dmesg
```

### Output Formats

```bash
# JSON output
journalctl -o json
journalctl -o json-pretty

# Short format
journalctl -o short

# Verbose
journalctl -o verbose

# Show only message
journalctl -o cat
```

### Journal Maintenance

```bash
# Disk usage
journalctl --disk-usage

# Clean by time
sudo journalctl --vacuum-time=7d

# Clean by size
sudo journalctl --vacuum-size=500M

# Rotate logs
sudo journalctl --rotate

# Verify integrity
journalctl --verify
```

## dmesg - Kernel Messages

```bash
# View kernel messages
dmesg

# Follow (like tail -f)
dmesg -w

# Human-readable timestamps
dmesg -T

# Colorized output
dmesg --color=always

# Filter by facility
dmesg --facility=kern

# Filter by level
dmesg --level=err,warn

# Clear ring buffer
sudo dmesg -c
```

## Log Rotation

### logrotate Configuration

```bash
# Main config
/etc/logrotate.conf

# Application-specific
/etc/logrotate.d/

# Manual rotation
sudo logrotate /etc/logrotate.conf
sudo logrotate -f /etc/logrotate.d/nginx  # Force
```

### logrotate Config Example

```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily                   # Rotate daily
    rotate 7                # Keep 7 rotations
    compress                # Compress old logs
    delaycompress           # Delay compression by 1 rotation
    missingok               # Don't error if log missing
    notifempty              # Don't rotate if empty
    create 0640 root adm    # Create new log with permissions
    sharedscripts           # Run scripts once for all logs
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

## Troubleshooting Techniques

### System Information

```bash
# System overview
hostnamectl
uname -a

# Uptime and load
uptime

# Memory usage
free -h

# Disk usage
df -h

# Inode usage
df -i
```

### Process Troubleshooting

```bash
# High CPU processes
ps aux --sort=-%cpu | head

# High memory processes
ps aux --sort=-%mem | head

# Process tree
pstree -p

# Open files by process
lsof -p <pid>

# Network connections by process
lsof -i -P -n

# Strace (system calls)
strace -p <pid>
strace command
```

### Network Troubleshooting

```bash
# Check interfaces
ip addr
ip link

# Check routing
ip route

# Check connectivity
ping -c 4 8.8.8.8
ping -c 4 google.com

# DNS resolution
nslookup google.com
dig google.com

# Check ports
ss -tulpn
netstat -tulpn

# Test specific port
nc -zv hostname 80
telnet hostname 80

# Trace route
traceroute google.com
mtr google.com
```

### Disk Troubleshooting

```bash
# Disk usage
df -h

# Find large files
find / -type f -size +100M 2>/dev/null

# Find large directories
du -h / | sort -hr | head -20

# Check for disk errors
sudo smartctl -a /dev/sda

# Check filesystem
sudo fsck -n /dev/sda1
```

### Service Troubleshooting

```bash
# Check service status
systemctl status nginx

# Check if service is active
systemctl is-active nginx

# Check if service is enabled
systemctl is-enabled nginx

# View service logs
journalctl -u nginx -n 50

# Follow service logs
journalctl -u nginx -f

# Check failed services
systemctl --failed
```

### Boot Troubleshooting

```bash
# Boot time analysis
systemd-analyze

# Slow services
systemd-analyze blame

# Critical chain
systemd-analyze critical-chain

# Boot logs
journalctl -b
journalctl -b -1  # Previous boot

# Kernel messages
dmesg | less
```

### Common Issues and Solutions

| Issue | Commands to Check | Common Solutions |
|-------|------------------|------------------|
| High CPU | `top`, `htop`, `ps aux --sort=-%cpu` | Kill process, optimize code |
| High Memory | `free -h`, `ps aux --sort=-%mem` | Kill process, add swap |
| Disk Full | `df -h`, `du -sh /*` | Remove files, extend disk |
| Can't SSH | `systemctl status sshd`, firewall | Start sshd, check firewall |
| Service Down | `systemctl status <service>` | Restart, check logs |
| Network Issues | `ip addr`, `ping`, `ss -tulpn` | Check config, firewall |
| Permission Denied | `ls -la`, `namei -l` | Fix permissions, ownership |

### Debug Mode

```bash
# Bash script debug
bash -x script.sh

# Set debug in script
set -x  # Enable
set +x  # Disable

# Verbose mode
bash -v script.sh
```

## Monitoring Commands

```bash
# Real-time system monitor
top
htop

# I/O monitoring
iotop

# Network monitoring
iftop
nethogs

# System activity
vmstat 1
iostat 1

# Memory info
free -h
cat /proc/meminfo

# Watch command
watch -n 1 'ps aux --sort=-%cpu | head'
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `tail -f` | Follow log | `tail -f /var/log/syslog` |
| `journalctl` | View systemd logs | `journalctl -u nginx -f` |
| `dmesg` | Kernel messages | `dmesg -T` |
| `grep` | Search logs | `grep error /var/log/syslog` |
| `last` | Login history | `last` |
| `lastb` | Failed logins | `lastb` |
| `who` | Current users | `who` |
| `uptime` | System uptime | `uptime` |
| `free` | Memory usage | `free -h` |
| `df` | Disk usage | `df -h` |
| `top` | Process monitor | `top` |
| `ss` | Network sockets | `ss -tulpn` |

---

**Previous:** [19-cron-scheduling.md](19-cron-scheduling.md)

---

## Congratulations!

You've completed the Linux section of the DevOps roadmap. You should now have a solid foundation in:

- Linux fundamentals and architecture
- File system and permissions
- User and process management
- Networking and security
- Storage and logging
- Troubleshooting techniques

**Next Topic:** [02-Networking](../02-Networking/)
