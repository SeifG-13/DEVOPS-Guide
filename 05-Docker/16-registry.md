# Docker Registry

## What is a Registry?

A registry stores and distributes Docker images.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Registry Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Developer                Registry                 Server       │
│   ┌─────────┐             ┌─────────┐            ┌─────────┐   │
│   │ docker  │   push      │         │    pull    │ docker  │   │
│   │ build   │────────────►│  Image  │───────────►│ run     │   │
│   │         │             │ Storage │            │         │   │
│   └─────────┘             └─────────┘            └─────────┘   │
│                                                                  │
│   Image naming: [registry/][namespace/]name[:tag]               │
│                                                                  │
│   Examples:                                                      │
│   • nginx                    (Docker Hub official)              │
│   • myuser/myapp:v1          (Docker Hub user repo)             │
│   • gcr.io/project/app       (Google Container Registry)        │
│   • ghcr.io/owner/app        (GitHub Container Registry)        │
│   • registry.local:5000/app  (Private registry)                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Docker Hub

### Login and Logout

```bash
# Login to Docker Hub
docker login

# Login with credentials
docker login -u username -p password

# Login to specific registry
docker login registry.example.com

# Logout
docker logout
docker logout registry.example.com
```

### Push to Docker Hub

```bash
# Tag image for Docker Hub
docker tag myapp:latest username/myapp:latest
docker tag myapp:latest username/myapp:v1.0

# Push to Docker Hub
docker push username/myapp:latest
docker push username/myapp:v1.0

# Push all tags
docker push username/myapp --all-tags
```

### Pull from Docker Hub

```bash
# Pull latest
docker pull nginx

# Pull specific tag
docker pull nginx:1.25

# Pull from user repository
docker pull username/myapp:v1.0
```

### Search Docker Hub

```bash
# Search for images
docker search nginx

# Filter official images
docker search --filter is-official=true nginx

# Filter by stars
docker search --filter stars=100 nginx
```

## Private Registry

### Run Local Registry

```bash
# Simple registry
docker run -d -p 5000:5000 --name registry registry:2

# With persistent storage
docker run -d \
    -p 5000:5000 \
    --name registry \
    -v registry_data:/var/lib/registry \
    registry:2

# With TLS
docker run -d \
    -p 5000:5000 \
    --name registry \
    -v /certs:/certs \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    registry:2
```

### Use Private Registry

```bash
# Tag for private registry
docker tag myapp:latest localhost:5000/myapp:latest

# Push to private registry
docker push localhost:5000/myapp:latest

# Pull from private registry
docker pull localhost:5000/myapp:latest
```

### Insecure Registry (HTTP)

```bash
# Configure daemon for insecure registry
# /etc/docker/daemon.json
{
  "insecure-registries": ["registry.local:5000"]
}

# Restart Docker
sudo systemctl restart docker
```

## Popular Registries

### Google Container Registry (GCR)

```bash
# Authenticate
gcloud auth configure-docker

# Tag and push
docker tag myapp gcr.io/project-id/myapp:v1
docker push gcr.io/project-id/myapp:v1

# Pull
docker pull gcr.io/project-id/myapp:v1
```

### Amazon ECR

```bash
# Get login command
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp

# Tag and push
docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### GitHub Container Registry (GHCR)

```bash
# Login with PAT
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag and push
docker tag myapp:latest ghcr.io/username/myapp:latest
docker push ghcr.io/username/myapp:latest

# Pull public image
docker pull ghcr.io/username/myapp:latest
```

### Azure Container Registry (ACR)

```bash
# Login
az acr login --name myregistry

# Tag and push
docker tag myapp myregistry.azurecr.io/myapp:v1
docker push myregistry.azurecr.io/myapp:v1
```

## Registry Authentication

### Credential Helpers

```bash
# Install credential helper
# macOS: docker-credential-osxkeychain
# Linux: docker-credential-secretservice
# Windows: docker-credential-wincred

# Configure in ~/.docker/config.json
{
  "credHelpers": {
    "gcr.io": "gcloud",
    "aws_account_id.dkr.ecr.region.amazonaws.com": "ecr-login"
  }
}
```

### Service Account Authentication

```bash
# Login with service account (CI/CD)
echo $REGISTRY_PASSWORD | docker login -u $REGISTRY_USER --password-stdin registry.example.com
```

## Registry API

### List Repositories

```bash
# List repositories (v2 API)
curl -X GET https://registry.example.com/v2/_catalog

# List tags
curl -X GET https://registry.example.com/v2/myapp/tags/list

# Get manifest
curl -X GET https://registry.example.com/v2/myapp/manifests/latest
```

### Delete Image

```bash
# Get digest
digest=$(curl -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
    https://registry.example.com/v2/myapp/manifests/v1 | \
    grep Docker-Content-Digest | awk '{print $2}')

# Delete by digest
curl -X DELETE https://registry.example.com/v2/myapp/manifests/$digest
```

## Harbor Registry

Enterprise-class registry with advanced features.

### Features

- Role-based access control
- Image scanning
- Image signing
- Replication
- Audit logs
- LDAP/AD integration

### Deploy Harbor

```bash
# Download installer
wget https://github.com/goharbor/harbor/releases/download/v2.9.0/harbor-online-installer-v2.9.0.tgz
tar xvf harbor-online-installer-v2.9.0.tgz
cd harbor

# Configure
cp harbor.yml.tmpl harbor.yml
# Edit harbor.yml

# Install
./install.sh
```

### Use Harbor

```bash
# Login
docker login harbor.example.com

# Push
docker tag myapp harbor.example.com/library/myapp:v1
docker push harbor.example.com/library/myapp:v1
```

## Registry Best Practices

### Image Tagging Strategy

```bash
# Semantic versioning
myapp:1.0.0
myapp:1.0
myapp:1

# Git-based tags
myapp:abc1234          # Commit hash
myapp:main-abc1234     # Branch + commit
myapp:v1.0.0-abc1234   # Version + commit

# Environment tags
myapp:production
myapp:staging
myapp:development

# Build metadata
myapp:1.0.0-build.123
myapp:1.0.0-20240101
```

### Security

```bash
# Use digest for production
docker pull nginx@sha256:abc123...

# Scan images
docker scan myapp:latest

# Sign images (Docker Content Trust)
export DOCKER_CONTENT_TRUST=1
docker push myapp:latest  # Signs automatically
```

### CI/CD Integration

```yaml
# GitHub Actions example
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    push: true
    tags: user/app:latest
```

## Registry Mirroring

### Configure Mirror

```bash
# /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://mirror.example.com"
  ]
}
```

### Run Mirror

```bash
docker run -d \
    -p 5000:5000 \
    --name mirror \
    -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
    registry:2
```

## Quick Reference

| Registry | URL Format |
|----------|------------|
| Docker Hub | `docker.io/user/image` |
| GCR | `gcr.io/project/image` |
| ECR | `account.dkr.ecr.region.amazonaws.com/image` |
| GHCR | `ghcr.io/owner/image` |
| ACR | `registry.azurecr.io/image` |
| Self-hosted | `registry.local:5000/image` |

| Command | Description |
|---------|-------------|
| `docker login` | Authenticate |
| `docker logout` | Remove credentials |
| `docker push` | Upload image |
| `docker pull` | Download image |
| `docker search` | Search registry |
| `docker tag` | Tag for registry |

---

**Previous:** [15-docker-compose-advanced.md](15-docker-compose-advanced.md) | **Next:** [17-docker-security.md](17-docker-security.md)
