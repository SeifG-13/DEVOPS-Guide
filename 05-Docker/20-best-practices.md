# Docker Best Practices

## Dockerfile Best Practices

### Use Official Base Images

```dockerfile
# Good: Official images are maintained and secure
FROM node:20-alpine
FROM python:3.11-slim
FROM nginx:alpine

# Avoid: Unknown or unverified images
FROM random-user/random-image
```

### Use Specific Version Tags

```dockerfile
# Bad: Unpredictable builds
FROM node:latest
FROM ubuntu

# Good: Reproducible builds
FROM node:20.10-alpine3.18
FROM ubuntu:22.04
```

### Minimize Layers

```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y nginx
RUN apt-get clean

# Good: Single layer with cleanup
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Order Instructions by Change Frequency

```dockerfile
# Least frequently changed first
FROM node:20-alpine

# System dependencies (rarely change)
RUN apk add --no-cache python3 make g++

# Application dependencies (change sometimes)
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Application code (changes frequently)
COPY . .

CMD ["node", "server.js"]
```

### Use Multi-Stage Builds

```dockerfile
# Build stage
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

### Run as Non-Root User

```dockerfile
FROM node:20-alpine

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

USER appuser

CMD ["node", "server.js"]
```

### Use COPY Instead of ADD

```dockerfile
# Bad: ADD has extra features that are rarely needed
ADD . /app
ADD https://example.com/file.tar.gz /app/

# Good: COPY is explicit
COPY . /app
```

### Don't Store Secrets in Images

```dockerfile
# Bad: Secret in image
ENV API_KEY=secret123

# Good: Pass at runtime
# docker run -e API_KEY=secret myapp
```

### Include Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1
```

## Container Best Practices

### One Process Per Container

```yaml
# Good: Separate containers
services:
  web:
    image: nginx
  api:
    image: myapi
  db:
    image: postgres

# Bad: Multiple processes in one container
```

### Use Read-Only Filesystem

```bash
docker run --read-only --tmpfs /tmp nginx
```

### Set Resource Limits

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
```

### Use Restart Policies

```bash
docker run --restart unless-stopped myapp
```

### Keep Containers Stateless

```bash
# Store data in volumes, not container layer
docker run -v data:/var/lib/mysql mysql
```

## Image Best Practices

### Scan for Vulnerabilities

```bash
# Regular scanning
docker scout cves myimage:latest
trivy image myimage:latest
```

### Sign Images

```bash
export DOCKER_CONTENT_TRUST=1
docker push myregistry/myapp:latest
```

### Use .dockerignore

```dockerignore
.git
.gitignore
node_modules
npm-debug.log
Dockerfile*
docker-compose*
.env*
*.md
tests/
coverage/
```

### Tag Images Properly

```bash
# Use semantic versioning
myapp:1.0.0
myapp:1.0
myapp:1

# Include build info
myapp:1.0.0-abc1234
myapp:1.0.0-20240101
```

## Security Best Practices

### Security Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│                    Security Checklist                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Image Security                                                │
│   □ Use minimal base images                                     │
│   □ Scan for vulnerabilities                                    │
│   □ No secrets in images                                        │
│   □ Use specific version tags                                   │
│   □ Sign images                                                 │
│                                                                  │
│   Container Security                                            │
│   □ Run as non-root                                            │
│   □ Read-only filesystem                                        │
│   □ Drop capabilities                                           │
│   □ Set resource limits                                         │
│   □ No privileged mode                                          │
│                                                                  │
│   Network Security                                              │
│   □ Use user-defined networks                                   │
│   □ Limit port exposure                                         │
│   □ Internal networks for databases                             │
│                                                                  │
│   Host Security                                                 │
│   □ Keep Docker updated                                         │
│   □ Use rootless mode                                           │
│   □ Enable TLS for remote API                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Drop Capabilities

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

### Use Security Options

```bash
docker run --security-opt=no-new-privileges myapp
```

## Networking Best Practices

### Use User-Defined Networks

```bash
# Create network for service isolation
docker network create myapp

# Connect containers
docker run --network myapp --name api myapi
docker run --network myapp --name db postgres
```

### Don't Expose Database Ports

```yaml
services:
  db:
    image: postgres
    # No ports exposed externally
    networks:
      - backend

  api:
    image: myapi
    networks:
      - backend
      - frontend
    ports:
      - "3000:3000"  # Only API exposed
```

## Volume Best Practices

### Use Named Volumes for Persistence

```bash
# Named volume (managed by Docker)
docker run -v mydata:/var/lib/mysql mysql

# Not anonymous volumes
docker run -v /var/lib/mysql mysql
```

### Backup Volumes Regularly

```bash
# Backup
docker run --rm -v mydata:/source:ro -v $(pwd):/backup alpine \
    tar czf /backup/mydata.tar.gz -C /source .

# Restore
docker run --rm -v mydata:/target -v $(pwd):/backup alpine \
    tar xzf /backup/mydata.tar.gz -C /target
```

## Logging Best Practices

### Configure Log Rotation

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### Log to stdout/stderr

```dockerfile
# Application should log to stdout/stderr
# Don't log to files inside container
CMD ["myapp", "--log-to-stdout"]
```

## CI/CD Best Practices

### Build Once, Deploy Everywhere

```yaml
# Build image once
- docker build -t myapp:$CI_COMMIT_SHA .

# Deploy same image to all environments
- docker tag myapp:$CI_COMMIT_SHA myapp:staging
- docker tag myapp:$CI_COMMIT_SHA myapp:production
```

### Use Build Cache

```yaml
# Cache dependencies in CI
- docker build --cache-from myapp:latest -t myapp:$SHA .
```

### Scan in Pipeline

```yaml
- docker build -t myapp:$SHA .
- trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:$SHA
- docker push myapp:$SHA
```

## Production Checklist

### Pre-Deployment

```
□ Image scanned for vulnerabilities
□ Image signed (if using content trust)
□ Resource limits defined
□ Health checks configured
□ Logging configured
□ Secrets management in place
□ Backup strategy defined
□ Monitoring configured
```

### Runtime

```
□ Non-root user
□ Read-only filesystem (if possible)
□ Minimal capabilities
□ Network isolation
□ Resource limits enforced
□ Restart policy set
□ Log rotation enabled
```

## Quick Reference

### Dockerfile

| Practice | Example |
|----------|---------|
| Specific tags | `FROM node:20-alpine` |
| Multi-stage | `FROM builder AS production` |
| Non-root | `USER appuser` |
| Health check | `HEALTHCHECK CMD curl...` |
| Copy over Add | `COPY . /app` |
| Combine RUN | `RUN cmd1 && cmd2` |

### Container

| Practice | Flag/Option |
|----------|-------------|
| Read-only | `--read-only` |
| Memory limit | `-m 512m` |
| CPU limit | `--cpus=1` |
| No root | `--user 1000` |
| Drop caps | `--cap-drop=ALL` |
| Restart | `--restart unless-stopped` |

### Security

| Practice | Implementation |
|----------|----------------|
| Scan images | `trivy image myapp` |
| Sign images | `DOCKER_CONTENT_TRUST=1` |
| Secrets | Docker secrets, external managers |
| Network | User-defined, internal networks |
| Privileges | `--security-opt=no-new-privileges` |

---

**Previous:** [19-logging-monitoring.md](19-logging-monitoring.md)

---

## Congratulations!

You've completed the Docker section of the DevOps roadmap. You now understand:

- Docker architecture and core concepts
- Building and managing images
- Container lifecycle and management
- Networking and storage
- Docker Compose for multi-container applications
- Security best practices
- Optimization techniques
- Logging and monitoring

**Next Topic:** [06-Kubernetes](../06-Kubernetes/)
