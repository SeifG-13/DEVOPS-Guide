# Package Management

## Package Managers Overview

| Distribution | Package Format | Low-Level Tool | High-Level Tool |
|--------------|----------------|----------------|-----------------|
| Debian/Ubuntu | .deb | dpkg | apt, apt-get |
| RHEL/CentOS/Fedora | .rpm | rpm | yum, dnf |
| Arch | .pkg.tar.xz | pacman | pacman |
| Alpine | .apk | apk | apk |

## APT (Debian/Ubuntu)

### apt vs apt-get

- `apt` - User-friendly, progress bar, recommended for interactive use
- `apt-get` - Scriptable, stable interface, better for scripts

### Update Package Lists

```bash
# Update package lists
sudo apt update

# Upgrade installed packages
sudo apt upgrade

# Full upgrade (handles dependencies)
sudo apt full-upgrade

# Update and upgrade together
sudo apt update && sudo apt upgrade -y
```

### Installing Packages

```bash
# Install package
sudo apt install nginx

# Install multiple packages
sudo apt install nginx php mysql-server

# Install without prompts
sudo apt install -y nginx

# Install specific version
sudo apt install nginx=1.18.0-0ubuntu1

# Reinstall package
sudo apt reinstall nginx

# Install from .deb file
sudo apt install ./package.deb
sudo dpkg -i package.deb
```

### Removing Packages

```bash
# Remove package (keep config)
sudo apt remove nginx

# Remove package and config
sudo apt purge nginx

# Remove with dependencies
sudo apt autoremove nginx

# Remove unused dependencies
sudo apt autoremove
```

### Searching Packages

```bash
# Search packages
apt search nginx

# Show package info
apt show nginx

# List installed packages
apt list --installed

# List upgradable packages
apt list --upgradable

# Check if package is installed
dpkg -l | grep nginx
apt list nginx
```

### Package Information

```bash
# Show package details
apt show nginx

# Show dependencies
apt depends nginx

# Show reverse dependencies
apt rdepends nginx

# Show package files
dpkg -L nginx

# Find which package owns file
dpkg -S /usr/sbin/nginx
```

### Cache Management

```bash
# Clean package cache
sudo apt clean

# Remove old cached packages
sudo apt autoclean

# Show cache statistics
apt-cache stats
```

### Repository Management

```bash
# List repositories
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# Add repository
sudo add-apt-repository ppa:ondrej/php
sudo add-apt-repository "deb http://repo.example.com/ubuntu focal main"

# Remove repository
sudo add-apt-repository --remove ppa:ondrej/php

# Add GPG key
curl -fsSL https://example.com/key.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/example.gpg
```

## dpkg (Low-Level Debian)

```bash
# Install .deb package
sudo dpkg -i package.deb

# Remove package
sudo dpkg -r package

# Purge package (with config)
sudo dpkg -P package

# List installed packages
dpkg -l

# List files in package
dpkg -L nginx

# Find package owning file
dpkg -S /path/to/file

# Show package status
dpkg -s nginx

# Extract package without installing
dpkg -x package.deb /destination/

# Reconfigure package
sudo dpkg-reconfigure package

# Fix broken dependencies
sudo apt --fix-broken install
sudo dpkg --configure -a
```

## YUM/DNF (RHEL/CentOS/Fedora)

DNF is the modern replacement for YUM (Fedora, RHEL 8+).

### Basic Operations

```bash
# Update package lists and upgrade
sudo yum update          # or
sudo dnf update

# Check for updates
yum check-update
dnf check-update

# Install package
sudo yum install nginx
sudo dnf install nginx

# Install without prompts
sudo yum install -y nginx

# Remove package
sudo yum remove nginx
sudo dnf remove nginx

# Reinstall package
sudo yum reinstall nginx
```

### Searching Packages

```bash
# Search packages
yum search nginx
dnf search nginx

# Show package info
yum info nginx
dnf info nginx

# List installed packages
yum list installed
dnf list installed

# List available packages
yum list available
dnf list available

# Find which package provides file
yum provides /usr/sbin/nginx
dnf provides /usr/sbin/nginx
```

### Groups

```bash
# List groups
yum grouplist
dnf grouplist

# Install group
sudo yum groupinstall "Development Tools"
sudo dnf groupinstall "Development Tools"

# Remove group
sudo yum groupremove "Development Tools"
```

### Repository Management

```bash
# List repositories
yum repolist
dnf repolist

# List all repos (enabled and disabled)
yum repolist all

# Enable/disable repo
sudo yum-config-manager --enable repo_name
sudo yum-config-manager --disable repo_name

# Add repository
sudo yum-config-manager --add-repo https://example.com/repo.repo

# Install EPEL repository
sudo yum install epel-release
```

### Cache Management

```bash
# Clean cache
sudo yum clean all
sudo dnf clean all

# Make cache
sudo yum makecache
sudo dnf makecache

# Clean specific
sudo yum clean packages
sudo yum clean metadata
```

## RPM (Low-Level RHEL)

```bash
# Install RPM
sudo rpm -ivh package.rpm

# Upgrade RPM
sudo rpm -Uvh package.rpm

# Remove package
sudo rpm -e package

# Query installed packages
rpm -qa

# Query specific package
rpm -q nginx

# Query package info
rpm -qi nginx

# Query package files
rpm -ql nginx

# Find package owning file
rpm -qf /path/to/file

# Query RPM file
rpm -qip package.rpm

# Verify package
rpm -V nginx
```

## Common Tasks

### Check Available Updates

```bash
# Debian/Ubuntu
apt list --upgradable

# RHEL/CentOS
yum check-update
```

### Security Updates Only

```bash
# Ubuntu
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades

# RHEL/CentOS
sudo yum update --security
```

### Hold Package Version

```bash
# Debian/Ubuntu
sudo apt-mark hold nginx
sudo apt-mark unhold nginx
apt-mark showhold

# RHEL/CentOS (using versionlock plugin)
sudo yum install yum-plugin-versionlock
sudo yum versionlock nginx
sudo yum versionlock list
sudo yum versionlock delete nginx
```

### Download Without Installing

```bash
# Debian/Ubuntu
apt download nginx

# RHEL/CentOS
yumdownloader nginx
dnf download nginx
```

### List Package Dependencies

```bash
# Debian/Ubuntu
apt depends nginx
apt-cache depends nginx

# RHEL/CentOS
yum deplist nginx
dnf repoquery --requires nginx
```

## Quick Reference

### Debian/Ubuntu (apt)

| Task | Command |
|------|---------|
| Update lists | `apt update` |
| Upgrade | `apt upgrade` |
| Install | `apt install pkg` |
| Remove | `apt remove pkg` |
| Purge | `apt purge pkg` |
| Search | `apt search pkg` |
| Info | `apt show pkg` |
| List installed | `apt list --installed` |
| Autoremove | `apt autoremove` |
| Clean | `apt clean` |

### RHEL/CentOS (yum/dnf)

| Task | Command |
|------|---------|
| Update | `yum update` |
| Install | `yum install pkg` |
| Remove | `yum remove pkg` |
| Search | `yum search pkg` |
| Info | `yum info pkg` |
| List installed | `yum list installed` |
| Provides | `yum provides file` |
| Clean | `yum clean all` |
| Repos | `yum repolist` |

---

**Previous:** [14-service-management-systemd.md](14-service-management-systemd.md) | **Next:** [16-networking-commands.md](16-networking-commands.md)
