# Docker Volumes

## Why Use Volumes?

Container data is ephemeral - when a container is removed, its data is lost. Volumes provide persistent storage.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Data Persistence Options                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without Volume              With Volume                        │
│   ┌─────────────┐            ┌─────────────┐                    │
│   │  Container  │            │  Container  │                    │
│   │  ┌───────┐  │            │  ┌───────┐  │                    │
│   │  │ Data  │  │            │  │ Data  │  │                    │
│   │  └───────┘  │            │  └───┬───┘  │                    │
│   └──────┬──────┘            └──────┼──────┘                    │
│          │                          │                            │
│          ▼                          ▼                            │
│   ┌─────────────┐            ┌─────────────┐                    │
│   │  Deleted    │            │   Volume    │                    │
│   │  with       │            │  (persists) │                    │
│   │  container  │            │             │                    │
│   └─────────────┘            └─────────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Storage Options

| Type | Managed By | Location | Use Case |
|------|------------|----------|----------|
| Volumes | Docker | /var/lib/docker/volumes | Production data |
| Bind Mounts | User | Anywhere on host | Development |
| tmpfs | Docker | Host memory | Sensitive/temporary |

## Docker Volumes

### Create Volumes

```bash
# Create named volume
docker volume create mydata

# Create with options
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.1,rw \
    --opt device=:/path/to/dir \
    nfs_volume

# Create with labels
docker volume create --label env=prod mydata
```

### List Volumes

```bash
# List all volumes
docker volume ls

# Filter volumes
docker volume ls -f name=my
docker volume ls -f driver=local
docker volume ls -f dangling=true
docker volume ls -f label=env=prod

# Format output
docker volume ls --format "table {{.Name}}\t{{.Driver}}\t{{.Mountpoint}}"
```

### Inspect Volumes

```bash
# Inspect volume
docker volume inspect mydata

# Get specific field
docker volume inspect --format '{{.Mountpoint}}' mydata

# Output example:
# {
#     "Name": "mydata",
#     "Driver": "local",
#     "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
#     "Labels": {},
#     "Scope": "local"
# }
```

### Remove Volumes

```bash
# Remove specific volume
docker volume rm mydata

# Remove multiple volumes
docker volume rm vol1 vol2 vol3

# Remove all unused volumes
docker volume prune

# Force remove (even if in use - dangerous!)
docker volume rm -f mydata
```

## Using Volumes

### Mount Volume to Container

```bash
# Using -v flag
docker run -v mydata:/app/data nginx

# Using --mount flag (more explicit)
docker run --mount source=mydata,target=/app/data nginx

# Volume options with --mount
docker run --mount \
    type=volume,\
    source=mydata,\
    target=/app/data,\
    readonly \
    nginx

# Auto-create volume
docker run -v newvolume:/data nginx  # Creates if doesn't exist
```

### -v vs --mount

| Feature | `-v` | `--mount` |
|---------|------|-----------|
| Syntax | Compact | Verbose, explicit |
| Auto-create | Yes | Error if not exists |
| Recommended | Development | Production |
| Options | Limited | Full options |

```bash
# -v syntax
docker run -v mydata:/app/data:ro nginx

# --mount syntax (equivalent)
docker run --mount type=volume,source=mydata,target=/app/data,readonly nginx
```

## Bind Mounts

Mount host directory into container.

```bash
# Using -v
docker run -v /host/path:/container/path nginx

# Using --mount
docker run --mount type=bind,source=/host/path,target=/container/path nginx

# Read-only bind mount
docker run -v /host/path:/container/path:ro nginx
docker run --mount type=bind,source=/host/path,target=/container/path,readonly nginx

# Current directory
docker run -v $(pwd):/app node npm start

# Windows path
docker run -v "C:\Users\data":/app nginx
```

### Bind Mount Use Cases

```bash
# Development: live code reload
docker run -v $(pwd)/src:/app/src node npm run dev

# Configuration files
docker run -v /host/config/nginx.conf:/etc/nginx/nginx.conf:ro nginx

# Logs
docker run -v /host/logs:/var/log/nginx nginx

# Sharing data between host and container
docker run -v /host/shared:/shared ubuntu
```

## tmpfs Mounts

Store data in host memory (not persisted).

```bash
# Using --tmpfs
docker run --tmpfs /app/cache nginx

# Using --mount
docker run --mount type=tmpfs,target=/app/cache nginx

# With options
docker run --mount type=tmpfs,target=/app/cache,tmpfs-size=100m,tmpfs-mode=1770 nginx

# Multiple tmpfs
docker run --tmpfs /tmp --tmpfs /run nginx
```

### tmpfs Use Cases

- Sensitive data (secrets, tokens)
- Session data
- Cache that doesn't need persistence
- Performance-critical temporary data

## Volume Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    Storage Types Comparison                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Volumes               Bind Mounts            tmpfs             │
│   ┌──────────┐         ┌──────────┐          ┌──────────┐      │
│   │Container │         │Container │          │Container │      │
│   │   /data  │         │   /data  │          │   /data  │      │
│   └────┬─────┘         └────┬─────┘          └────┬─────┘      │
│        │                    │                     │             │
│        ▼                    ▼                     ▼             │
│   ┌──────────┐         ┌──────────┐          ┌──────────┐      │
│   │ Docker   │         │   Host   │          │   Host   │      │
│   │ Area     │         │Filesystem│          │  Memory  │      │
│   │          │         │          │          │          │      │
│   │/var/lib/ │         │/host/path│          │ (tmpfs)  │      │
│   │docker/   │         │          │          │          │      │
│   │volumes/  │         │          │          │          │      │
│   └──────────┘         └──────────┘          └──────────┘      │
│                                                                  │
│   Best for:            Best for:              Best for:         │
│   - Production data    - Development          - Secrets         │
│   - Database storage   - Config files         - Temp data       │
│   - Shared data        - Source code          - Cache           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Volume Drivers

### Local Driver (Default)

```bash
# Default local driver
docker volume create --driver local mydata

# NFS mount
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.1,rw \
    --opt device=:/export/data \
    nfs_data
```

### Third-Party Drivers

```bash
# Install plugin (example: REX-Ray)
docker plugin install rexray/ebs

# Create volume with plugin
docker volume create -d rexray/ebs --name ebs_vol

# Common plugins:
# - rexray/ebs (AWS EBS)
# - rexray/s3fs (AWS S3)
# - vieux/sshfs (SSH)
# - local-persist
```

## Practical Examples

### Database with Volume

```bash
# PostgreSQL
docker run -d \
    --name postgres \
    -e POSTGRES_PASSWORD=secret \
    -v postgres_data:/var/lib/postgresql/data \
    postgres:15

# MySQL
docker run -d \
    --name mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -v mysql_data:/var/lib/mysql \
    mysql:8

# MongoDB
docker run -d \
    --name mongo \
    -v mongo_data:/data/db \
    mongo:6
```

### Sharing Data Between Containers

```bash
# Create shared volume
docker volume create shared_data

# Writer container
docker run -d \
    --name writer \
    -v shared_data:/data \
    alpine sh -c "while true; do date >> /data/log.txt; sleep 5; done"

# Reader container
docker run -it \
    --name reader \
    -v shared_data:/data:ro \
    alpine tail -f /data/log.txt
```

### Backup and Restore

```bash
# Backup volume to tar
docker run --rm \
    -v mydata:/source:ro \
    -v $(pwd):/backup \
    alpine tar cvf /backup/mydata_backup.tar -C /source .

# Restore from tar
docker run --rm \
    -v mydata:/target \
    -v $(pwd):/backup:ro \
    alpine sh -c "cd /target && tar xvf /backup/mydata_backup.tar"

# Backup to remote (with compression)
docker run --rm \
    -v mydata:/source:ro \
    alpine tar czf - -C /source . | ssh user@host "cat > /backup/mydata.tar.gz"
```

### Development Setup

```bash
# Node.js development
docker run -it \
    -v $(pwd):/app \
    -v /app/node_modules \
    -w /app \
    -p 3000:3000 \
    node:20 npm run dev

# Note: /app/node_modules is anonymous volume
# Prevents overwriting container's node_modules with empty host dir
```

## Volume in Docker Compose

```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro

  app:
    image: myapp
    volumes:
      - ./src:/app/src
      - app_logs:/app/logs

volumes:
  postgres_data:
  app_logs:
    driver: local
    driver_opts:
      type: none
      device: /host/path/logs
      o: bind
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker volume create` | Create volume |
| `docker volume ls` | List volumes |
| `docker volume inspect` | Volume details |
| `docker volume rm` | Remove volume |
| `docker volume prune` | Remove unused |
| `-v name:/path` | Mount volume |
| `-v /host:/container` | Bind mount |
| `--tmpfs /path` | tmpfs mount |
| `--mount type=...` | Explicit mount |

---

**Previous:** [12-storage.md](12-storage.md) | **Next:** [14-docker-compose-basics.md](14-docker-compose-basics.md)
