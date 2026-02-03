# Docker Storage

## Storage Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Storage Architecture                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Container                                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Container Layer (R/W)  ◄── Writable, ephemeral         │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  Image Layer 4 (R/O)                                    │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  Image Layer 3 (R/O)                                    │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  Image Layer 2 (R/O)    ◄── Read-only, shared           │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  Image Layer 1 (R/O)                                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Storage Driver manages layers (overlay2, devicemapper, etc.) │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Storage Drivers

### Available Drivers

| Driver | Description | Backing Filesystem |
|--------|-------------|-------------------|
| `overlay2` | Modern, recommended | xfs, ext4 |
| `fuse-overlayfs` | Rootless mode | Any |
| `btrfs` | Btrfs features | btrfs |
| `zfs` | ZFS features | zfs |
| `devicemapper` | Older, LVM-based | direct-lvm |
| `vfs` | No copy-on-write | Any |

### Check Current Driver

```bash
# Check storage driver
docker info | grep "Storage Driver"

# Detailed storage info
docker info --format '{{.Driver}}'

# Full driver info
docker system info | grep -A 10 "Storage Driver"
```

### Configure Storage Driver

```bash
# In /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}

# Restart Docker
sudo systemctl restart docker
```

## overlay2 Driver (Recommended)

### How overlay2 Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    overlay2 Filesystem                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Merged View (what container sees)                             │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  /etc/nginx/nginx.conf  (from lower)                    │   │
│   │  /var/log/nginx.log     (from upper - container)        │   │
│   │  /app/index.html        (from lower)                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                         │                                       │
│           ┌─────────────┴─────────────┐                        │
│           │                           │                         │
│   ┌───────▼───────┐          ┌───────▼───────┐                 │
│   │  Upper Dir    │          │  Lower Dirs   │                 │
│   │  (writable)   │          │  (read-only)  │                 │
│   │               │          │  Image layers │                 │
│   │  Container    │          │               │                 │
│   │  changes      │          │  Layer 3      │                 │
│   └───────────────┘          │  Layer 2      │                 │
│                              │  Layer 1      │                 │
│                              └───────────────┘                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### overlay2 Configuration

```bash
# daemon.json configuration
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.size=10G"
  ]
}
```

## Docker Data Location

### Default Location

```bash
# Default: /var/lib/docker
ls /var/lib/docker/

# Contents:
# - containers/   Container data
# - image/        Image layers
# - volumes/      Named volumes
# - network/      Network config
# - overlay2/     Storage driver data
# - tmp/          Temporary files
```

### Change Data Location

```bash
# Method 1: daemon.json
{
  "data-root": "/mnt/docker-data"
}

# Method 2: Symlink
sudo systemctl stop docker
sudo mv /var/lib/docker /mnt/docker-data
sudo ln -s /mnt/docker-data /var/lib/docker
sudo systemctl start docker
```

## Disk Usage

### Check Disk Usage

```bash
# Overall disk usage
docker system df

# Detailed breakdown
docker system df -v

# Example output:
# TYPE            TOTAL   ACTIVE  SIZE      RECLAIMABLE
# Images          10      5       5.3GB     2.1GB (40%)
# Containers      8       3       500MB     200MB (40%)
# Local Volumes   5       3       1GB       400MB (40%)
# Build Cache     20      0       2GB       2GB (100%)
```

### Clean Up Disk Space

```bash
# Remove unused data
docker system prune

# Remove all unused data (including volumes)
docker system prune -a --volumes

# Remove specific types
docker image prune          # Unused images
docker container prune      # Stopped containers
docker volume prune         # Unused volumes
docker network prune        # Unused networks
docker builder prune        # Build cache

# Remove with filters
docker image prune -a --filter "until=24h"
docker container prune --filter "until=24h"
```

## Container Storage

### Container Layer

```bash
# View container storage
docker inspect --format '{{.GraphDriver.Data}}' container_name

# View container size
docker ps -s

# SIZE column shows:
# - Virtual size: total image + container layer
# - Size: container layer only
```

### Copy-on-Write (CoW)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Copy-on-Write                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   When container modifies a file from image layer:              │
│                                                                  │
│   1. File is copied to container's writable layer               │
│   2. Modification happens on the copy                           │
│   3. Original file in image layer unchanged                     │
│                                                                  │
│   Image Layer                  Container Layer                   │
│   ┌──────────────┐            ┌──────────────┐                  │
│   │ file.txt (v1)│───copy────►│ file.txt (v2)│                  │
│   │  (read-only) │            │  (modified)  │                  │
│   └──────────────┘            └──────────────┘                  │
│                                                                  │
│   Container sees file.txt (v2) from its layer                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Image Storage

### Layer Storage

```bash
# List image layers
docker history nginx

# View layer details
docker inspect nginx | jq '.[0].RootFS.Layers'

# Layer storage location
ls /var/lib/docker/overlay2/
```

### Shared Layers

```bash
# Multiple containers/images share common layers
# Example: Two containers from same image

docker run -d --name web1 nginx
docker run -d --name web2 nginx

# Both share all image layers
# Only container layers are separate
# Saves significant disk space
```

## Storage Performance

### Best Practices

```bash
# 1. Use overlay2 driver
{
  "storage-driver": "overlay2"
}

# 2. Use SSD for /var/lib/docker

# 3. Limit container layer size
docker run --storage-opt size=10G nginx

# 4. Use volumes for heavy I/O
docker run -v data:/var/lib/mysql mysql

# 5. Minimize image layers
# Combine RUN commands
```

### Performance Comparison

| Storage Type | Read | Write | Use Case |
|--------------|------|-------|----------|
| Image layers | Fast | N/A (read-only) | Application code |
| Container layer | Medium | Medium | Temporary data |
| Volumes | Fast | Fast | Persistent data |
| Bind mounts | Fast | Fast | Development |

## Quotas and Limits

### Container Storage Limit

```bash
# Limit container writable layer size
# (requires overlay2 with xfs backing filesystem)
docker run --storage-opt size=5G nginx
```

### Configure Quotas

```bash
# daemon.json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.size=10G"
  ]
}
```

## Troubleshooting Storage

### Common Issues

```bash
# Issue: No space left on device
docker system prune -a --volumes

# Issue: Slow container performance
# Check if heavy writes to container layer
docker diff container_name

# Issue: Large image sizes
docker history --no-trunc image_name

# Issue: Storage driver not supported
docker info | grep "Storage Driver"
cat /proc/filesystems
```

### Checking Storage Health

```bash
# Check overlay mounts
mount | grep overlay

# Check filesystem
df -h /var/lib/docker

# Check inodes
df -i /var/lib/docker

# Docker storage info
docker info | grep -A 20 "Storage Driver"
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker system df` | Disk usage |
| `docker system df -v` | Detailed disk usage |
| `docker system prune` | Clean up |
| `docker info` | Storage driver info |
| `docker ps -s` | Container sizes |
| `docker history` | Image layers |
| `docker diff` | Container changes |

---

**Previous:** [11-networking.md](11-networking.md) | **Next:** [13-volumes.md](13-volumes.md)
