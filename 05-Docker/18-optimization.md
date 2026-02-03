# Docker Optimization

## Image Size Optimization

### Size Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    Base Image Sizes                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ubuntu:22.04        ████████████████████████████  77MB        │
│   debian:bookworm     ████████████████████████      124MB       │
│   node:20             █████████████████████████████████████ 1GB │
│   node:20-slim        ████████████████  200MB                   │
│   node:20-alpine      █████  140MB                              │
│   alpine:3.18         █  7MB                                    │
│   gcr.io/distroless   ██  20MB                                  │
│   scratch             0MB                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Use Minimal Base Images

```dockerfile
# Before: 1GB+
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]

# After: ~150MB
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["node", "server.js"]
```

### Multi-Stage Builds

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

# Result: Only production files, no build tools
```

### Reduce Layers

```dockerfile
# Bad: Many layers
FROM alpine
RUN apk update
RUN apk add curl
RUN apk add nginx
RUN rm -rf /var/cache/apk/*

# Good: Single layer
FROM alpine
RUN apk update && \
    apk add --no-cache curl nginx
```

### Clean Up in Same Layer

```dockerfile
# Bad: Files remain in previous layer
RUN apt-get update
RUN apt-get install -y build-essential
RUN make && make install
RUN apt-get remove -y build-essential  # Files still in earlier layer!

# Good: Clean up in same layer
RUN apt-get update && \
    apt-get install -y build-essential && \
    make && make install && \
    apt-get remove -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
```

### Use .dockerignore

```dockerignore
# Reduce build context
.git
.gitignore
node_modules
npm-debug.log
Dockerfile*
docker-compose*
.dockerignore
README.md
.env*
tests/
coverage/
*.md
```

## Build Cache Optimization

### Layer Ordering

```dockerfile
# Dependencies change less frequently than code
# Copy dependency files first

# Good order:
FROM node:20-alpine
WORKDIR /app

# 1. System dependencies (rarely change)
RUN apk add --no-cache python3 make g++

# 2. Package files (change sometimes)
COPY package*.json ./
RUN npm ci --only=production

# 3. Application code (changes frequently)
COPY . .

CMD ["node", "server.js"]
```

### Cache External Dependencies

```dockerfile
# Use BuildKit cache mount
FROM python:3.11-slim

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Node.js example
FROM node:20-alpine

RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production
```

### Parallel Builds

```dockerfile
# syntax=docker/dockerfile:1.4

# Build multiple stages in parallel
FROM node:20 AS frontend
WORKDIR /frontend
COPY frontend/ .
RUN npm ci && npm run build

FROM golang:1.21 AS backend
WORKDIR /backend
COPY backend/ .
RUN go build -o server .

FROM alpine:3.18
COPY --from=frontend /frontend/dist /app/static
COPY --from=backend /backend/server /app/server
CMD ["/app/server"]
```

## Runtime Optimization

### Resource Limits

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1
```

### Proper Signal Handling

```dockerfile
# Use exec form for proper signal handling
CMD ["node", "server.js"]

# Or use tini as init system
FROM node:20-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

## Language-Specific Optimizations

### Node.js

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Install production dependencies only
COPY package*.json ./
RUN npm ci --only=production

# Don't copy unnecessary files
COPY src/ ./src/

# Use node directly, not npm
CMD ["node", "src/server.js"]
```

### Python

```dockerfile
FROM python:3.11-slim

# Prevent Python from writing bytecode
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

### Go

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

# Production stage - scratch for minimal size
FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
ENTRYPOINT ["/server"]
```

### Java

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Production stage
FROM eclipse-temurin:17-jre-alpine
COPY --from=builder /app/target/*.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

## Network Optimization

### Use Host Network for Performance

```bash
# When network performance is critical
docker run --network host myapp

# Note: Reduces isolation
```

### DNS Caching

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    dns:
      - 8.8.8.8
      - 8.8.4.4
```

## Storage Optimization

### Use Volumes for Heavy I/O

```bash
# Better performance than container layer
docker run -v data:/var/lib/mysql mysql
```

### Tmpfs for Temporary Data

```bash
# Memory-based storage for temp files
docker run --tmpfs /tmp:rw,noexec,nosuid,size=100m myapp
```

## Monitoring and Profiling

### Check Image Size

```bash
# Image size
docker images myapp

# Layer sizes
docker history myapp

# Detailed analysis
docker inspect myapp | jq '.[0].Size'

# Use dive tool
dive myapp:latest
```

### Runtime Stats

```bash
# Real-time stats
docker stats

# Container resource usage
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### Build Time Analysis

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Build with timing
docker build --progress=plain -t myapp .

# Profile build
time docker build -t myapp .
```

## Optimization Checklist

### Image Size

- [ ] Use minimal base image (Alpine, slim, distroless)
- [ ] Multi-stage builds
- [ ] Combine RUN commands
- [ ] Clean up in same layer
- [ ] Use .dockerignore
- [ ] Remove unnecessary files

### Build Time

- [ ] Order layers by change frequency
- [ ] Use cache mounts
- [ ] Copy dependency files first
- [ ] Parallel builds where possible

### Runtime

- [ ] Set resource limits
- [ ] Use health checks
- [ ] Proper signal handling
- [ ] Use volumes for data

### General

- [ ] Scan for vulnerabilities
- [ ] Use specific version tags
- [ ] Document Dockerfile

## Quick Reference

| Optimization | Technique |
|-------------|-----------|
| Smaller base | Alpine, slim, distroless |
| Reduce layers | Combine RUN commands |
| Build cache | Order by change frequency |
| Multi-stage | Separate build/runtime |
| Clean up | Same layer as install |
| Dependencies | Cache mount, copy first |
| Static binary | FROM scratch |

---

**Previous:** [17-docker-security.md](17-docker-security.md) | **Next:** [19-logging-monitoring.md](19-logging-monitoring.md)
