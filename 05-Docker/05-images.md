# Docker Images

## What is a Docker Image?

A Docker image is a read-only template containing instructions for creating a container. Images are built in layers, with each layer representing a set of filesystem changes.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Docker Image Structure                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Application Code (Layer 5)                 │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │              Dependencies (Layer 4)                     │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │              Runtime Environment (Layer 3)              │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │              System Libraries (Layer 2)                 │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │              Base OS (Layer 1)                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Each layer is:                                                │
│   • Read-only                                                   │
│   • Cached and reusable                                         │
│   • Identified by SHA256 hash                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Image Naming Convention

```
┌─────────────────────────────────────────────────────────────────┐
│                    Image Name Format                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [registry/][repository/]name[:tag][@digest]                   │
│                                                                  │
│   Examples:                                                      │
│   • nginx                      (official, latest tag)           │
│   • nginx:1.25                 (specific version)               │
│   • nginx:alpine               (variant)                        │
│   • library/nginx              (official repository)            │
│   • myuser/myapp:v1.0          (user repository)               │
│   • gcr.io/project/app:latest  (Google Container Registry)     │
│   • ghcr.io/owner/app:v1       (GitHub Container Registry)     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Naming Components

| Component | Description | Example |
|-----------|-------------|---------|
| Registry | Image registry host | `docker.io`, `gcr.io` |
| Repository | Namespace/owner | `library`, `myuser` |
| Name | Image name | `nginx`, `myapp` |
| Tag | Version identifier | `latest`, `1.25`, `alpine` |
| Digest | SHA256 hash | `sha256:abc123...` |

## Pulling Images

### docker pull

```bash
# Pull latest tag
docker pull nginx

# Pull specific tag
docker pull nginx:1.25

# Pull specific variant
docker pull nginx:alpine

# Pull from specific registry
docker pull gcr.io/google-containers/nginx

# Pull by digest (immutable)
docker pull nginx@sha256:abc123def456...

# Pull all tags
docker pull -a nginx
```

## Listing Images

### docker images

```bash
# List all images
docker images

# List with specific repository
docker images nginx

# List only image IDs
docker images -q

# List with digests
docker images --digests

# Filter images
docker images -f "dangling=true"
docker images -f "before=nginx:1.24"
docker images -f "reference=nginx:*"

# Format output
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Show all images (including intermediate)
docker images -a
```

### Output Columns

| Column | Description |
|--------|-------------|
| REPOSITORY | Image name |
| TAG | Version tag |
| IMAGE ID | Unique identifier |
| CREATED | When created |
| SIZE | Image size |

## Image Information

### docker inspect

```bash
# Inspect image
docker inspect nginx

# Get specific information
docker inspect --format '{{.Config.Env}}' nginx
docker inspect --format '{{.Config.ExposedPorts}}' nginx
docker inspect --format '{{.Config.Cmd}}' nginx

# Get image layers
docker inspect --format '{{.RootFS.Layers}}' nginx
```

### docker history

```bash
# View image layers/history
docker history nginx

# Show full commands
docker history --no-trunc nginx

# Format output
docker history --format "table {{.CreatedBy}}\t{{.Size}}" nginx
```

## Image Layers

```bash
# View layers
docker history nginx

# Example output:
IMAGE          CREATED       CREATED BY                                      SIZE
a6bd71f48f68   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B
<missing>      2 weeks ago   STOPSIGNAL SIGQUIT                              0B
<missing>      2 weeks ago   EXPOSE map[80/tcp:{}]                           0B
<missing>      2 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B
<missing>      2 weeks ago   COPY file:xxx /docker-entrypoint.d/30...        4.62kB
<missing>      2 weeks ago   COPY file:xxx /docker-entrypoint.d/20...        3.02kB
<missing>      2 weeks ago   RUN /bin/sh -c set -x && addgroup...            61.1MB
```

### Layer Caching

```
┌─────────────────────────────────────────────────────────────────┐
│                     Layer Caching                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Image A              Image B              Disk                 │
│   ┌──────────┐        ┌──────────┐        ┌──────────┐         │
│   │ Layer 4  │        │ Layer 4  │        │ Layer 4  │ Unique  │
│   ├──────────┤        ├──────────┤        ├──────────┤         │
│   │ Layer 3  │ ◄──────┤ Layer 3  │ ──────►│ Layer 3  │ Shared  │
│   ├──────────┤        ├──────────┤        ├──────────┤         │
│   │ Layer 2  │ ◄──────┤ Layer 2  │ ──────►│ Layer 2  │ Shared  │
│   ├──────────┤        ├──────────┤        ├──────────┤         │
│   │ Layer 1  │ ◄──────┤ Layer 1  │ ──────►│ Layer 1  │ Shared  │
│   └──────────┘        └──────────┘        └──────────┘         │
│                                                                  │
│   Shared layers are stored only once on disk                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Tagging Images

### docker tag

```bash
# Tag an image
docker tag nginx:latest myregistry/nginx:v1.0

# Tag with multiple tags
docker tag myapp:latest myapp:v1.0
docker tag myapp:latest myapp:stable

# Tag for different registry
docker tag myapp:latest gcr.io/myproject/myapp:v1.0
docker tag myapp:latest ghcr.io/myuser/myapp:v1.0

# Retag image
docker tag old-name:old-tag new-name:new-tag
```

## Removing Images

### docker rmi

```bash
# Remove image by name
docker rmi nginx

# Remove image by ID
docker rmi abc123

# Force remove (even if used by containers)
docker rmi -f nginx

# Remove multiple images
docker rmi nginx redis mysql

# Remove all unused images
docker image prune

# Remove all unused images (not just dangling)
docker image prune -a

# Remove images matching filter
docker images -q -f "dangling=true" | xargs docker rmi
```

### Dangling Images

```bash
# Dangling images: layers not referenced by any tagged image
# Usually created during builds when new layers replace old ones

# List dangling images
docker images -f "dangling=true"

# Remove dangling images
docker image prune

# Automatically remove with build
docker build --rm -t myapp .
```

## Docker Hub

### Searching Images

```bash
# Search on Docker Hub
docker search nginx

# Search with filters
docker search --filter is-official=true nginx
docker search --filter stars=100 nginx

# Limit results
docker search --limit 5 nginx
```

### Official Images

```
┌─────────────────────────────────────────────────────────────────┐
│                    Official Images                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Official images are curated by Docker, Inc.                   │
│                                                                  │
│   • nginx          Web server                                   │
│   • mysql          Database                                     │
│   • postgres       Database                                     │
│   • redis          Cache/Message broker                         │
│   • node           Node.js runtime                              │
│   • python         Python runtime                               │
│   • alpine         Minimal Linux base                           │
│   • ubuntu         Ubuntu base                                  │
│   • debian         Debian base                                  │
│                                                                  │
│   Pull: docker pull <image-name>                                │
│   Full name: library/<image-name>                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Image Variants

### Common Tags

| Tag Pattern | Description |
|-------------|-------------|
| `latest` | Most recent stable |
| `<version>` | Specific version (e.g., `1.25`) |
| `<version>-alpine` | Alpine-based (smaller) |
| `<version>-slim` | Minimal dependencies |
| `<version>-bullseye` | Debian Bullseye based |
| `<version>-bookworm` | Debian Bookworm based |

### Example: Node.js Tags

```bash
# Full Debian-based (largest)
docker pull node:20

# Slim variant (smaller)
docker pull node:20-slim

# Alpine variant (smallest)
docker pull node:20-alpine

# Size comparison
# node:20         ~1GB
# node:20-slim    ~250MB
# node:20-alpine  ~180MB
```

## Saving and Loading Images

### docker save

```bash
# Save image to tar file
docker save nginx > nginx.tar
docker save -o nginx.tar nginx

# Save multiple images
docker save -o images.tar nginx redis mysql

# Save with gzip compression
docker save nginx | gzip > nginx.tar.gz
```

### docker load

```bash
# Load image from tar file
docker load < nginx.tar
docker load -i nginx.tar

# Load from compressed file
gunzip -c nginx.tar.gz | docker load
```

## Exporting and Importing

### docker export (container → tar)

```bash
# Export container filesystem
docker export container_name > container.tar
docker export -o container.tar container_name
```

### docker import (tar → image)

```bash
# Import tar as image
docker import container.tar myimage:v1

# Import with commit message
docker import -m "Imported from container" container.tar myimage:v1
```

### save/load vs export/import

| Feature | save/load | export/import |
|---------|-----------|---------------|
| Source | Image | Container |
| Preserves | Layers, history | Flattened filesystem |
| Metadata | Yes | No |
| Use case | Transfer images | Create base images |

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker pull` | Download image |
| `docker images` | List images |
| `docker inspect` | Image details |
| `docker history` | Layer history |
| `docker tag` | Tag image |
| `docker rmi` | Remove image |
| `docker image prune` | Remove unused |
| `docker save` | Save to tar |
| `docker load` | Load from tar |
| `docker search` | Search Docker Hub |

---

**Previous:** [04-basic-commands.md](04-basic-commands.md) | **Next:** [06-dockerfile.md](06-dockerfile.md)
