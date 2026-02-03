# Kernel & Hardware

## What is the Kernel?

The kernel is the **core component** of Linux that acts as a bridge between hardware and software. It manages:
- Process scheduling
- Memory management
- Device drivers
- System calls
- File systems

## Kernel Architecture

```
┌────────────────────────────────────────────┐
│              User Space                     │
│   (Applications, Shell, Libraries)          │
├────────────────────────────────────────────┤
│            System Call Interface            │
├────────────────────────────────────────────┤
│                Kernel Space                 │
│  ┌──────────┬──────────┬──────────┐        │
│  │ Process  │  Memory  │   VFS    │        │
│  │ Manager  │ Manager  │          │        │
│  ├──────────┼──────────┼──────────┤        │
│  │ Network  │  Device  │   IPC    │        │
│  │  Stack   │ Drivers  │          │        │
│  └──────────┴──────────┴──────────┘        │
├────────────────────────────────────────────┤
│              Hardware Layer                 │
│    (CPU, RAM, Disk, Network, Devices)       │
└────────────────────────────────────────────┘
```

## Kernel Types

| Type | Description | Example |
|------|-------------|---------|
| Monolithic | All services in kernel space | Linux |
| Microkernel | Minimal kernel, services in user space | MINIX |
| Hybrid | Mix of both | Windows NT |

## Kernel Version

```bash
# Check kernel version
uname -r
# Output: 5.15.0-56-generic

# Detailed kernel info
uname -a

# Kernel version breakdown
# 5.15.0-56-generic
# │ │  │ │  └── Distribution specific
# │ │  │ └───── Patch level
# │ │  └─────── Minor version
# │ └────────── Major version
# └──────────── Kernel version
```

## Kernel Modules

Modules are pieces of code that can be loaded/unloaded into the kernel on demand (like drivers).

```bash
# List loaded modules
lsmod

# Get module info
modinfo <module_name>
modinfo ext4

# Load a module
sudo modprobe <module_name>

# Remove a module
sudo modprobe -r <module_name>

# Load module at boot - add to:
/etc/modules
# or
/etc/modules-load.d/*.conf
```

## Hardware Information Commands

### CPU Information

```bash
# CPU details
lscpu

# Processor info
cat /proc/cpuinfo

# Number of CPUs
nproc

# CPU summary
lscpu | grep -E "^CPU\(s\)|Model name|Thread|Core"
```

### Memory Information

```bash
# Memory usage
free -h

# Detailed memory info
cat /proc/meminfo

# Memory hardware info
sudo dmidecode -t memory
```

### Disk Information

```bash
# List block devices
lsblk

# Disk partitions
fdisk -l

# Disk usage
df -h

# Disk hardware
sudo hdparm -I /dev/sda
```

### PCI Devices

```bash
# List PCI devices (network cards, graphics, etc.)
lspci

# Verbose output
lspci -v

# Show kernel drivers
lspci -k
```

### USB Devices

```bash
# List USB devices
lsusb

# Verbose output
lsusb -v

# USB tree view
lsusb -t
```

### General Hardware

```bash
# All hardware info
sudo lshw

# Short summary
sudo lshw -short

# HTML report
sudo lshw -html > hardware.html

# DMI/SMBIOS info
sudo dmidecode

# Specific type (system, baseboard, processor, memory)
sudo dmidecode -t system
```

## Important Directories

| Directory | Content |
|-----------|---------|
| `/proc` | Virtual filesystem for process and kernel info |
| `/sys` | Virtual filesystem for device and driver info |
| `/dev` | Device files |
| `/lib/modules` | Kernel modules |
| `/boot` | Kernel and boot files |

## Common /proc Files

```bash
# CPU info
cat /proc/cpuinfo

# Memory info
cat /proc/meminfo

# Kernel version
cat /proc/version

# Mounted filesystems
cat /proc/mounts

# Uptime in seconds
cat /proc/uptime

# Load average
cat /proc/loadavg

# Running processes
ls /proc | grep -E "^[0-9]+$"
```

## Common /sys Directories

```bash
# Block devices
/sys/block/

# Bus information
/sys/bus/

# Device classes
/sys/class/

# Network devices
/sys/class/net/

# Power management
/sys/power/
```

## Kernel Parameters (sysctl)

```bash
# View all parameters
sysctl -a

# View specific parameter
sysctl net.ipv4.ip_forward

# Set parameter temporarily
sudo sysctl -w net.ipv4.ip_forward=1

# Set permanently - edit:
/etc/sysctl.conf
# or
/etc/sysctl.d/*.conf

# Apply changes
sudo sysctl -p
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `uname -r` | Kernel version |
| `lsmod` | List modules |
| `modprobe` | Load module |
| `lscpu` | CPU info |
| `lspci` | PCI devices |
| `lsusb` | USB devices |
| `lsblk` | Block devices |
| `lshw` | All hardware |
| `dmidecode` | BIOS/hardware |
| `free -h` | Memory usage |

---

**Previous:** [01-introduction.md](01-introduction.md) | **Next:** [03-boot-sequence-runlevels.md](03-boot-sequence-runlevels.md)
