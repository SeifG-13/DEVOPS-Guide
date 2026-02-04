# Ansible Ad-Hoc Commands

## What are Ad-Hoc Commands?

Ad-hoc commands are one-liner Ansible commands that perform quick tasks without writing a playbook. They're perfect for:
- Quick system checks
- One-time tasks
- Testing connectivity
- Gathering information
- Emergency fixes

---

## Basic Syntax

```bash
ansible <pattern> -m <module> -a "<arguments>" [options]
```

| Component | Description |
|-----------|-------------|
| `pattern` | Target hosts (all, group name, host name) |
| `-m` | Module to use |
| `-a` | Module arguments |
| `-i` | Inventory file |
| `-u` | Remote user |
| `-b` | Become (sudo) |
| `-k` | Ask for SSH password |
| `-K` | Ask for sudo password |

---

## Connectivity Testing

### Ping Module

```bash
# Test all hosts
ansible all -m ping

# Test specific group
ansible webservers -m ping

# Test specific host
ansible web1.example.com -m ping

# Test with specific inventory
ansible all -i inventory.yml -m ping

# Test with SSH password prompt
ansible all -m ping -k
```

### Expected Output

```
web1.example.com | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

---

## Common Modules

### Command Module (Default)

Executes commands on remote hosts. Does NOT use shell.

```bash
# Default module (command)
ansible all -a "uptime"

# Explicit command module
ansible all -m command -a "df -h"

# Check disk space
ansible all -a "df -h /"

# Check memory
ansible all -a "free -m"

# List processes
ansible all -a "ps aux | head"  # Won't work! No shell
```

### Shell Module

Executes commands through shell (supports pipes, redirects).

```bash
# Using shell features
ansible all -m shell -a "ps aux | grep nginx | wc -l"

# Redirects
ansible all -m shell -a "echo 'test' >> /tmp/test.txt"

# Environment variables
ansible all -m shell -a "echo $HOME"

# Complex commands
ansible all -m shell -a "cat /etc/passwd | grep -v nologin | wc -l"
```

### Raw Module

Executes commands without Python (for bootstrap or network devices).

```bash
# When Python isn't installed
ansible all -m raw -a "apt-get install -y python3"

# Network devices
ansible switches -m raw -a "show version"
```

---

## File Operations

### Copy Module

Copy files to remote hosts.

```bash
# Copy file to remote
ansible all -m copy -a "src=/local/file.txt dest=/remote/file.txt"

# Copy with permissions
ansible all -m copy -a "src=app.conf dest=/etc/app.conf mode=0644 owner=root"

# Copy content directly
ansible all -m copy -a "content='Hello World' dest=/tmp/hello.txt"

# Backup before overwrite
ansible all -m copy -a "src=config.conf dest=/etc/config.conf backup=yes"
```

### Fetch Module

Copy files from remote to local.

```bash
# Fetch file from remote
ansible all -m fetch -a "src=/var/log/syslog dest=/tmp/logs/ flat=yes"

# Fetch with directory structure
ansible all -m fetch -a "src=/etc/hostname dest=/tmp/hostnames/"
```

### File Module

Manage files and directories.

```bash
# Create directory
ansible all -m file -a "path=/opt/app state=directory mode=0755"

# Create empty file
ansible all -m file -a "path=/tmp/test.txt state=touch"

# Delete file
ansible all -m file -a "path=/tmp/test.txt state=absent"

# Create symlink
ansible all -m file -a "src=/etc/nginx dest=/opt/nginx-config state=link"

# Change ownership
ansible all -m file -a "path=/var/www owner=www-data group=www-data recurse=yes"

# Set permissions
ansible all -m file -a "path=/opt/app/script.sh mode=0755"
```

### Lineinfile Module

Manage lines in files.

```bash
# Add line to file
ansible all -m lineinfile -a "path=/etc/hosts line='192.168.1.100 app.local'"

# Replace line matching pattern
ansible all -m lineinfile -a "path=/etc/ssh/sshd_config regexp='^PermitRootLogin' line='PermitRootLogin no'"

# Remove line
ansible all -m lineinfile -a "path=/etc/hosts regexp='app.local' state=absent"

# Insert after pattern
ansible all -m lineinfile -a "path=/etc/fstab insertafter='^# swap' line='/dev/sdb1 none swap sw 0 0'"
```

---

## Package Management

### APT Module (Debian/Ubuntu)

```bash
# Update cache
ansible all -m apt -a "update_cache=yes" -b

# Install package
ansible all -m apt -a "name=nginx state=present" -b

# Install specific version
ansible all -m apt -a "name=nginx=1.18.0-0ubuntu1 state=present" -b

# Install multiple packages
ansible all -m apt -a "name=nginx,curl,vim state=present" -b

# Remove package
ansible all -m apt -a "name=nginx state=absent" -b

# Upgrade all packages
ansible all -m apt -a "upgrade=dist" -b

# Autoremove unused
ansible all -m apt -a "autoremove=yes" -b
```

### YUM/DNF Module (RHEL/CentOS)

```bash
# Install package
ansible all -m yum -a "name=httpd state=present" -b

# Install from URL
ansible all -m yum -a "name=https://example.com/package.rpm state=present" -b

# Remove package
ansible all -m yum -a "name=httpd state=absent" -b

# Update all packages
ansible all -m yum -a "name=* state=latest" -b

# Install group
ansible all -m yum -a "name='@Development Tools' state=present" -b
```

### Pip Module

```bash
# Install Python package
ansible all -m pip -a "name=flask"

# Install specific version
ansible all -m pip -a "name=flask==2.0.0"

# Install from requirements
ansible all -m pip -a "requirements=/opt/app/requirements.txt"

# Install in virtualenv
ansible all -m pip -a "name=flask virtualenv=/opt/venv"
```

---

## Service Management

### Service Module

```bash
# Start service
ansible all -m service -a "name=nginx state=started" -b

# Stop service
ansible all -m service -a "name=nginx state=stopped" -b

# Restart service
ansible all -m service -a "name=nginx state=restarted" -b

# Reload service
ansible all -m service -a "name=nginx state=reloaded" -b

# Enable at boot
ansible all -m service -a "name=nginx enabled=yes" -b

# Check service status
ansible all -m service -a "name=nginx" -b
```

### Systemd Module

```bash
# Start and enable
ansible all -m systemd -a "name=nginx state=started enabled=yes" -b

# Daemon reload
ansible all -m systemd -a "daemon_reload=yes" -b

# Restart with daemon reload
ansible all -m systemd -a "name=myapp state=restarted daemon_reload=yes" -b
```

---

## User and Group Management

### User Module

```bash
# Create user
ansible all -m user -a "name=deploy state=present" -b

# Create with home directory
ansible all -m user -a "name=deploy home=/opt/deploy shell=/bin/bash" -b

# Create with SSH key
ansible all -m user -a "name=deploy generate_ssh_key=yes ssh_key_bits=4096" -b

# Add to groups
ansible all -m user -a "name=deploy groups=sudo,docker append=yes" -b

# Set password (encrypted)
ansible all -m user -a "name=deploy password={{ 'password' | password_hash('sha512') }}" -b

# Remove user
ansible all -m user -a "name=olduser state=absent remove=yes" -b
```

### Group Module

```bash
# Create group
ansible all -m group -a "name=developers state=present" -b

# Create with GID
ansible all -m group -a "name=developers gid=1500 state=present" -b

# Remove group
ansible all -m group -a "name=oldgroup state=absent" -b
```

### Authorized Key Module

```bash
# Add SSH key
ansible all -m authorized_key -a "user=deploy key='ssh-rsa AAAA... user@host'" -b

# Add from file
ansible all -m authorized_key -a "user=deploy key=\"{{ lookup('file', '~/.ssh/id_rsa.pub') }}\"" -b

# Remove key
ansible all -m authorized_key -a "user=deploy key='ssh-rsa AAAA...' state=absent" -b
```

---

## System Information

### Setup Module (Gather Facts)

```bash
# Gather all facts
ansible all -m setup

# Filter specific facts
ansible all -m setup -a "filter=ansible_os_family"
ansible all -m setup -a "filter=ansible_distribution*"
ansible all -m setup -a "filter=ansible_memory_mb"
ansible all -m setup -a "filter=ansible_processor*"
ansible all -m setup -a "filter=ansible_default_ipv4"

# Save facts to file
ansible all -m setup --tree /tmp/facts
```

### Common Fact Filters

| Filter | Description |
|--------|-------------|
| `ansible_hostname` | Short hostname |
| `ansible_fqdn` | Fully qualified domain name |
| `ansible_distribution` | OS distribution (Ubuntu, CentOS) |
| `ansible_distribution_version` | OS version |
| `ansible_os_family` | OS family (Debian, RedHat) |
| `ansible_kernel` | Kernel version |
| `ansible_architecture` | CPU architecture |
| `ansible_processor_cores` | Number of CPU cores |
| `ansible_memtotal_mb` | Total memory in MB |
| `ansible_default_ipv4` | Default IPv4 info |

---

## Privilege Escalation

### Become Options

```bash
# Run as root (sudo)
ansible all -m apt -a "name=nginx state=present" -b

# Specify become method
ansible all -m command -a "whoami" -b --become-method=sudo

# Specify become user
ansible all -m command -a "whoami" -b --become-user=postgres

# Ask for sudo password
ansible all -m command -a "whoami" -b -K
```

---

## Parallelism and Performance

### Fork Control

```bash
# Default is 5 forks
ansible all -m ping

# Increase parallelism
ansible all -m ping -f 20

# Decrease for sensitive operations
ansible all -m service -a "name=nginx state=restarted" -b -f 1
```

### Async Tasks

```bash
# Run async (background) with timeout
ansible all -m command -a "sleep 60" -B 120 -P 0

# -B: timeout in seconds
# -P: poll interval (0 = fire and forget)

# Check async job status
ansible all -m async_status -a "jid=123456789"
```

---

## Output and Verbosity

### Verbose Modes

```bash
# Normal output
ansible all -m ping

# Verbose
ansible all -m ping -v

# More verbose (shows task results)
ansible all -m ping -vv

# Very verbose (shows connection info)
ansible all -m ping -vvv

# Debug (shows everything)
ansible all -m ping -vvvv
```

### Output Formats

```bash
# JSON output (default)
ansible all -m ping

# One-line output
ansible all -m ping -o

# YAML callback plugin
ANSIBLE_STDOUT_CALLBACK=yaml ansible all -m ping

# Minimal callback
ANSIBLE_STDOUT_CALLBACK=minimal ansible all -m ping
```

---

## Practical Examples

### System Health Check

```bash
# Check uptime
ansible all -a "uptime"

# Check disk space
ansible all -a "df -h /"

# Check memory
ansible all -m shell -a "free -m | grep Mem"

# Check running services
ansible all -m shell -a "systemctl list-units --state=running | head -20" -b

# Check load average
ansible all -a "cat /proc/loadavg"
```

### Quick Fixes

```bash
# Clear disk space
ansible all -m shell -a "apt clean && apt autoremove -y" -b

# Kill process
ansible all -m shell -a "pkill -f 'stuck_process'" -b

# Restart failed service
ansible webservers -m service -a "name=nginx state=restarted" -b

# Flush iptables
ansible all -m shell -a "iptables -F" -b

# Sync time
ansible all -m shell -a "ntpdate pool.ntp.org" -b
```

### Information Gathering

```bash
# Get OS info
ansible all -m setup -a "filter=ansible_distribution*"

# List installed packages
ansible all -m shell -a "dpkg -l | wc -l"

# Check open ports
ansible all -m shell -a "ss -tlnp" -b

# Get IP addresses
ansible all -m setup -a "filter=ansible_all_ipv4_addresses"
```

---

## Summary

| Task | Command |
|------|---------|
| **Test connectivity** | `ansible all -m ping` |
| **Run command** | `ansible all -a "uptime"` |
| **Run shell command** | `ansible all -m shell -a "cmd"` |
| **Copy file** | `ansible all -m copy -a "src=x dest=y"` |
| **Install package** | `ansible all -m apt -a "name=pkg" -b` |
| **Start service** | `ansible all -m service -a "name=svc state=started" -b` |
| **Create user** | `ansible all -m user -a "name=user" -b` |
| **Gather facts** | `ansible all -m setup` |
| **Increase verbosity** | Add `-v`, `-vv`, `-vvv` |
| **Use sudo** | Add `-b` flag |

Ad-hoc commands are powerful for quick tasks, but for complex or repeatable automation, use playbooks.
