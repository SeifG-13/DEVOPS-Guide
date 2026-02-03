# Introduction to Linux

## What is Linux?

Linux is an open-source, Unix-like operating system kernel created by **Linus Torvalds** in 1991. It forms the foundation of various operating systems called **Linux distributions**.

## Key Characteristics

| Feature | Description |
|---------|-------------|
| Open Source | Free to use, modify, and distribute |
| Multi-user | Multiple users can work simultaneously |
| Multi-tasking | Run multiple processes at the same time |
| Portable | Runs on various hardware platforms |
| Secure | Strong permission and access control system |
| Stable | Known for reliability and uptime |

## Linux vs Unix vs Windows

| Aspect | Linux | Unix | Windows |
|--------|-------|------|---------|
| Cost | Free | Expensive | Licensed |
| Source | Open | Closed | Closed |
| CLI | Bash, Zsh | sh, ksh | CMD, PowerShell |
| File System | ext4, xfs | UFS, ZFS | NTFS |
| Case Sensitive | Yes | Yes | No |

## Popular Linux Distributions

### For Servers (DevOps Focus)
- **Ubuntu Server** - User-friendly, great community support
- **CentOS / Rocky Linux / AlmaLinux** - Enterprise-grade, RHEL-based
- **Debian** - Stable, secure, minimal
- **Amazon Linux** - Optimized for AWS

### For Desktop
- Ubuntu Desktop
- Fedora
- Linux Mint
- Arch Linux

## Why Linux for DevOps?

1. **Server Dominance** - 90%+ of servers run Linux
2. **Cloud Native** - AWS, GCP, Azure primarily use Linux
3. **Container Foundation** - Docker and Kubernetes built on Linux
4. **Automation Friendly** - Powerful CLI and scripting
5. **Free & Customizable** - No licensing costs
6. **Security** - Regular patches, strong permissions
7. **Package Managers** - Easy software installation

## Linux Architecture Overview

```
┌─────────────────────────────────────┐
│           Applications              │
├─────────────────────────────────────┤
│              Shell                  │
├─────────────────────────────────────┤
│         System Libraries            │
├─────────────────────────────────────┤
│             Kernel                  │
├─────────────────────────────────────┤
│            Hardware                 │
└─────────────────────────────────────┘
```

## Accessing Linux

1. **Install on hardware** - Dual boot or dedicated machine
2. **Virtual Machine** - VirtualBox, VMware
3. **Cloud Instance** - AWS EC2, GCP Compute, Azure VM
4. **WSL** - Windows Subsystem for Linux
5. **Containers** - Docker with Linux image

## Basic Terminal Access

```bash
# Open terminal: Ctrl + Alt + T (Ubuntu)

# Check Linux version
cat /etc/os-release

# Check kernel version
uname -r

# Check system info
hostnamectl
```

---

**Next:** [02-kernel-hardware.md](02-kernel-hardware.md)
