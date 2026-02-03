# Storage Management

## Storage Concepts

### Storage Types

| Type | Description | Connection | Shared |
|------|-------------|------------|--------|
| **DAS** | Direct Attached Storage | Direct to server | No |
| **NAS** | Network Attached Storage | Network (NFS/SMB) | Yes |
| **SAN** | Storage Area Network | Fiber/iSCSI | Yes |

### DAS vs NAS vs SAN

```
DAS (Direct Attached Storage)
┌─────────┐     ┌─────────┐
│ Server  │────►│  Disk   │
└─────────┘     └─────────┘
Direct connection, no network

NAS (Network Attached Storage)
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Server  │◄───►│ Network │◄───►│   NAS   │
└─────────┘     └─────────┘     └─────────┘
File-level access over network (NFS/SMB)

SAN (Storage Area Network)
┌─────────┐     ┌─────────────┐     ┌─────────┐
│ Server  │◄───►│ SAN Fabric  │◄───►│ Storage │
└─────────┘     │(Fiber/iSCSI)│     │  Array  │
                └─────────────┘     └─────────┘
Block-level access, dedicated network
```

## Disk Information

### View Disks and Partitions

```bash
# List block devices
lsblk

# List with filesystem info
lsblk -f

# List all disks
fdisk -l
sudo fdisk -l

# Detailed disk info
sudo hdparm -I /dev/sda

# SCSI devices
lsscsi

# Disk serial, model
sudo smartctl -i /dev/sda
```

### Disk Usage

```bash
# Filesystem disk usage
df -h

# Inode usage
df -i

# Directory size
du -sh /path/to/dir

# Largest directories
du -h /path | sort -hr | head -20

# Exclude patterns
du -sh --exclude="*.log" /var
```

## Partitioning

### Partition Types

| Type | Description | Max Size | Max Partitions |
|------|-------------|----------|----------------|
| MBR | Master Boot Record | 2 TB | 4 primary |
| GPT | GUID Partition Table | 9.4 ZB | 128+ |

### fdisk (MBR/GPT)

```bash
# Start fdisk
sudo fdisk /dev/sdb

# fdisk commands:
# m - Help menu
# p - Print partitions
# n - New partition
# d - Delete partition
# t - Change partition type
# w - Write changes
# q - Quit without saving

# Example: Create new partition
sudo fdisk /dev/sdb
# n (new)
# p (primary)
# 1 (partition number)
# Enter (default first sector)
# +10G (size)
# w (write)
```

### parted (GPT recommended)

```bash
# Start parted
sudo parted /dev/sdb

# Print partitions
print

# Create GPT label
mklabel gpt

# Create partition
mkpart primary ext4 0% 50%
mkpart primary ext4 50% 100%

# Resize partition
resizepart 1 100%

# Delete partition
rm 1

# Quit
quit
```

### gdisk (GPT specific)

```bash
# Similar to fdisk but for GPT
sudo gdisk /dev/sdb
```

## Filesystems

### Common Filesystems

| Filesystem | Description | Max File | Max Volume |
|------------|-------------|----------|------------|
| ext4 | Standard Linux | 16 TB | 1 EB |
| xfs | High performance | 8 EB | 8 EB |
| btrfs | Modern, snapshots | 16 EB | 16 EB |
| ntfs | Windows | 16 EB | 256 TB |
| vfat | USB/FAT32 | 4 GB | 2 TB |

### Create Filesystem

```bash
# ext4
sudo mkfs.ext4 /dev/sdb1

# xfs
sudo mkfs.xfs /dev/sdb1

# With label
sudo mkfs.ext4 -L "data" /dev/sdb1

# Force (overwrite existing)
sudo mkfs.ext4 -F /dev/sdb1
```

### Check and Repair

```bash
# Check filesystem
sudo fsck /dev/sdb1
sudo fsck.ext4 /dev/sdb1

# Auto-repair
sudo fsck -y /dev/sdb1

# Check xfs
sudo xfs_repair /dev/sdb1
```

## Mounting

### Manual Mount

```bash
# Mount partition
sudo mount /dev/sdb1 /mnt/data

# Mount with options
sudo mount -o rw,noexec /dev/sdb1 /mnt/data

# Mount read-only
sudo mount -o ro /dev/sdb1 /mnt/data

# Mount ISO
sudo mount -o loop image.iso /mnt/iso

# Mount NFS
sudo mount -t nfs server:/share /mnt/nfs

# Mount SMB/CIFS
sudo mount -t cifs //server/share /mnt/smb -o user=username

# Unmount
sudo umount /mnt/data

# Force unmount
sudo umount -f /mnt/data

# Lazy unmount
sudo umount -l /mnt/data
```

### Permanent Mount (/etc/fstab)

```bash
# /etc/fstab format:
# device    mount-point    type    options    dump    fsck

# Examples:
/dev/sdb1    /data    ext4    defaults    0    2
UUID=abc123  /backup  xfs     defaults    0    2
/dev/sdc1    /media   ext4    noauto,user 0    0

# NFS mount
server:/share    /mnt/nfs    nfs    defaults    0    0

# SMB mount
//server/share   /mnt/smb    cifs   credentials=/etc/samba/creds    0    0
```

### Get UUID

```bash
# List UUIDs
blkid
lsblk -f

# Specific device
blkid /dev/sdb1
```

### Mount Options

| Option | Description |
|--------|-------------|
| `defaults` | rw, suid, dev, exec, auto, nouser, async |
| `ro` | Read-only |
| `rw` | Read-write |
| `noexec` | No execution of binaries |
| `nosuid` | Ignore SUID bits |
| `nodev` | Ignore device files |
| `noatime` | Don't update access time |
| `auto` | Mount at boot |
| `noauto` | Don't mount at boot |
| `user` | Allow users to mount |

## LVM - Logical Volume Manager

### LVM Concepts

```
Physical Volumes (PV) → Volume Groups (VG) → Logical Volumes (LV)

┌──────────────────────────────────────────┐
│            Volume Group (VG)              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │   LV1   │ │   LV2   │ │   LV3   │     │
│  │  /home  │ │  /var   │ │  /data  │     │
│  └─────────┘ └─────────┘ └─────────┘     │
├──────────────────────────────────────────┤
│  PV: /dev/sda2   PV: /dev/sdb1           │
└──────────────────────────────────────────┘
```

### Create LVM

```bash
# 1. Create Physical Volumes
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdc1

# View PVs
sudo pvs
sudo pvdisplay

# 2. Create Volume Group
sudo vgcreate myvg /dev/sdb1 /dev/sdc1

# View VGs
sudo vgs
sudo vgdisplay

# 3. Create Logical Volume
sudo lvcreate -L 50G -n mylv myvg
sudo lvcreate -l 100%FREE -n mylv myvg  # Use all space

# View LVs
sudo lvs
sudo lvdisplay

# 4. Create filesystem
sudo mkfs.ext4 /dev/myvg/mylv

# 5. Mount
sudo mount /dev/myvg/mylv /mnt/data
```

### Extend LVM

```bash
# Add disk to VG
sudo pvcreate /dev/sdd1
sudo vgextend myvg /dev/sdd1

# Extend LV
sudo lvextend -L +50G /dev/myvg/mylv
sudo lvextend -l +100%FREE /dev/myvg/mylv

# Resize filesystem
sudo resize2fs /dev/myvg/mylv        # ext4
sudo xfs_growfs /mnt/data            # xfs
```

### Reduce LVM

```bash
# Unmount first
sudo umount /mnt/data

# Check filesystem
sudo e2fsck -f /dev/myvg/mylv

# Resize filesystem
sudo resize2fs /dev/myvg/mylv 40G

# Reduce LV
sudo lvreduce -L 40G /dev/myvg/mylv

# Remount
sudo mount /dev/myvg/mylv /mnt/data
```

### Remove LVM

```bash
# Unmount
sudo umount /mnt/data

# Remove LV
sudo lvremove /dev/myvg/mylv

# Remove VG
sudo vgremove myvg

# Remove PV
sudo pvremove /dev/sdb1
```

## NFS - Network File System

### NFS Server Setup

```bash
# Install NFS server
sudo apt install nfs-kernel-server   # Debian/Ubuntu
sudo yum install nfs-utils           # RHEL/CentOS

# Configure exports
sudo vim /etc/exports

# /etc/exports format:
/data    192.168.1.0/24(rw,sync,no_subtree_check)
/backup  *(ro,sync)
/home    192.168.1.100(rw,no_root_squash)

# Export options:
# rw - Read-write
# ro - Read-only
# sync - Synchronous writes
# async - Asynchronous writes
# no_root_squash - Allow root access
# no_subtree_check - Disable subtree checking

# Apply exports
sudo exportfs -a

# Show exports
sudo exportfs -v

# Start NFS
sudo systemctl enable --now nfs-server
```

### NFS Client Setup

```bash
# Install NFS client
sudo apt install nfs-common          # Debian/Ubuntu
sudo yum install nfs-utils           # RHEL/CentOS

# Show available exports
showmount -e server_ip

# Mount NFS
sudo mount -t nfs server:/data /mnt/nfs

# Mount in fstab
server:/data    /mnt/nfs    nfs    defaults    0    0
```

## Swap Space

### Create Swap

```bash
# Create swap partition
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2

# Create swap file
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Add to fstab
/swapfile    none    swap    sw    0    0

# Check swap
swapon --show
free -h
```

### Manage Swap

```bash
# Enable swap
sudo swapon /dev/sdb2
sudo swapon -a        # All in fstab

# Disable swap
sudo swapoff /dev/sdb2
sudo swapoff -a       # All

# Set swappiness
sudo sysctl vm.swappiness=10
# Permanent: add to /etc/sysctl.conf
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `lsblk` | List block devices | `lsblk -f` |
| `fdisk` | Partition disk | `fdisk /dev/sdb` |
| `parted` | Partition (GPT) | `parted /dev/sdb` |
| `mkfs` | Create filesystem | `mkfs.ext4 /dev/sdb1` |
| `mount` | Mount filesystem | `mount /dev/sdb1 /mnt` |
| `umount` | Unmount | `umount /mnt` |
| `df` | Disk free space | `df -h` |
| `du` | Directory size | `du -sh /path` |
| `blkid` | Show UUIDs | `blkid` |
| `pvcreate` | Create PV | `pvcreate /dev/sdb1` |
| `vgcreate` | Create VG | `vgcreate myvg /dev/sdb1` |
| `lvcreate` | Create LV | `lvcreate -L 10G -n lv myvg` |
| `lvextend` | Extend LV | `lvextend -L +5G /dev/vg/lv` |

---

**Previous:** [17-ssh-scp.md](17-ssh-scp.md) | **Next:** [19-cron-scheduling.md](19-cron-scheduling.md)
