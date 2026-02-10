# Linux Cheat Sheet for DevOps Engineers

## Quick Reference Commands

### File System Navigation
```bash
pwd                     # Print working directory
ls -la                  # List all files with details
cd /path/to/dir         # Change directory
tree -L 2               # Directory tree (2 levels)
```

### File Operations
```bash
cp -r src dest          # Copy recursively
mv old new              # Move/rename
rm -rf dir              # Remove directory (CAUTION!)
touch file.txt          # Create empty file
mkdir -p path/to/dir    # Create nested directories
ln -s target link       # Create symbolic link
```

### File Permissions
```bash
chmod 755 file          # rwxr-xr-x
chmod u+x file          # Add execute for owner
chown user:group file   # Change ownership
umask 022               # Default permission mask
```

### Text Processing
```bash
cat file                # Display file content
head -n 20 file         # First 20 lines
tail -f /var/log/syslog # Follow log in real-time
grep -r "pattern" .     # Recursive search
sed -i 's/old/new/g' f  # Find and replace in-place
awk '{print $1}' file   # Print first column
cut -d: -f1 /etc/passwd # Cut with delimiter
sort | uniq -c          # Count unique occurrences
```

### Process Management
```bash
ps aux                  # All running processes
top / htop              # Interactive process viewer
kill -9 PID             # Force kill process
pkill -f "pattern"      # Kill by pattern
nohup cmd &             # Run in background (survives logout)
jobs / fg / bg          # Job control
```

### Systemd Services
```bash
systemctl start nginx   # Start service
systemctl enable nginx  # Enable on boot
systemctl status nginx  # Check status
journalctl -u nginx -f  # Follow service logs
systemctl daemon-reload # Reload after config change
```

### Networking
```bash
ip addr                 # Show IP addresses
ss -tulnp               # Show listening ports
netstat -tulnp          # Legacy port listing
curl -I url             # HTTP headers only
wget -q -O- url         # Download to stdout
ping -c 4 host          # Test connectivity
traceroute host         # Trace packet route
dig domain              # DNS lookup
nslookup domain         # DNS query
```

### Disk & Storage
```bash
df -h                   # Disk usage (human readable)
du -sh *                # Directory sizes
lsblk                   # List block devices
mount /dev/sda1 /mnt    # Mount filesystem
fdisk -l                # List partitions
```

### Package Management
```bash
# Debian/Ubuntu
apt update && apt upgrade
apt install package
apt remove package
dpkg -l | grep package

# RHEL/CentOS
yum update / dnf update
yum install package
rpm -qa | grep package
```

### SSH & Remote
```bash
ssh user@host           # Connect to remote
ssh -i key.pem user@ip  # Connect with key
scp file user@host:/p   # Copy to remote
rsync -avz src dest     # Sync files
ssh-keygen -t ed25519   # Generate SSH key
ssh-copy-id user@host   # Copy key to remote
```

### Compression
```bash
tar -czvf arch.tar.gz d # Create gzip archive
tar -xzvf arch.tar.gz   # Extract gzip archive
zip -r arch.zip dir     # Create zip
unzip arch.zip          # Extract zip
```

### Cron Jobs
```bash
crontab -e              # Edit cron jobs
crontab -l              # List cron jobs
# Format: m h dom mon dow command
# */5 * * * * /script.sh  # Every 5 minutes
```

---

## Interview Q&A

### Q1: What is the difference between hard link and soft link?
**A:**
- **Hard link**: Points directly to inode; survives original file deletion; cannot cross filesystems; cannot link directories
- **Soft link**: Points to filename; breaks if original deleted; can cross filesystems; can link directories

### Q2: Explain Linux boot process
**A:**
1. **BIOS/UEFI** - Hardware initialization
2. **Bootloader (GRUB)** - Loads kernel
3. **Kernel** - Initializes hardware, mounts root filesystem
4. **Init/Systemd** - First process (PID 1), starts services
5. **Runlevel/Target** - Multi-user or graphical target

### Q3: How do you troubleshoot a server that's running slow?
**A:**
1. `top/htop` - Check CPU and memory usage
2. `iostat` - Check disk I/O
3. `vmstat` - Virtual memory statistics
4. `netstat/ss` - Network connections
5. `dmesg` - Kernel messages
6. `/var/log/` - Check system logs

### Q4: What is the difference between process and thread?
**A:**
- **Process**: Independent execution unit with own memory space
- **Thread**: Lightweight unit within process, shares memory with other threads

### Q5: Explain file permissions 755 vs 644
**A:**
- **755 (rwxr-xr-x)**: Owner full access, others read+execute (for scripts/directories)
- **644 (rw-r--r--)**: Owner read+write, others read only (for regular files)

### Q6: What is swap space?
**A:** Virtual memory on disk used when RAM is full. Acts as overflow for physical memory. Configure with `mkswap` and `swapon`.

### Q7: How do you check which process is using a specific port?
**A:**
```bash
ss -tulnp | grep :80
netstat -tulnp | grep :80
lsof -i :80
fuser 80/tcp
```

### Q8: What is the difference between systemctl and service commands?
**A:**
- `systemctl` - Systemd (modern, more features, parallel startup)
- `service` - SysVinit (legacy, sequential startup)

### Q9: How do you find files larger than 100MB?
**A:**
```bash
find / -type f -size +100M -exec ls -lh {} \;
```

### Q10: Explain /etc/passwd and /etc/shadow
**A:**
- **/etc/passwd**: User account info (readable by all)
- **/etc/shadow**: Encrypted passwords (root only)

---

## Essential Paths for DevOps

| Path | Purpose |
|------|---------|
| `/etc/` | Configuration files |
| `/var/log/` | Log files |
| `/var/run/` | Runtime data, PID files |
| `/tmp/` | Temporary files |
| `/opt/` | Optional/third-party software |
| `/usr/local/bin/` | Local binaries |
| `/proc/` | Virtual filesystem for processes |
| `/sys/` | Virtual filesystem for kernel |

---

## Performance Monitoring Commands

```bash
# CPU
uptime                  # Load averages
mpstat                  # CPU statistics

# Memory
free -h                 # Memory usage
vmstat 1 5              # Virtual memory stats

# Disk
iostat -x 1             # I/O statistics
iotop                   # I/O by process

# Network
iftop                   # Network bandwidth
nethogs                 # Bandwidth by process
```

---

## Best Practices

1. **Never run as root** - Use sudo for privileged operations
2. **Use SSH keys** - Disable password authentication
3. **Automate with scripts** - Bash scripts for repetitive tasks
4. **Monitor logs** - Set up log rotation and centralized logging
5. **Regular backups** - Test restore procedures
6. **Keep systems updated** - Security patches are critical
7. **Document everything** - Maintain runbooks
