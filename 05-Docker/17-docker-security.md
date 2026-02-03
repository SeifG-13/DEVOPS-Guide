# Docker Security

## Security Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Security Layers                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Application                           │   │
│   │              (secure code, dependencies)                │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │                    Container                             │   │
│   │        (non-root, read-only, capabilities)              │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │                      Image                               │   │
│   │          (minimal base, no secrets, scanning)           │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │                  Docker Daemon                           │   │
│   │            (rootless, TLS, authorization)               │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │                    Host OS                               │   │
│   │         (updated, hardened, limited access)             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Image Security

### Use Minimal Base Images

```dockerfile
# Bad: Full OS
FROM ubuntu:22.04

# Better: Slim variant
FROM python:3.11-slim

# Best: Minimal base
FROM python:3.11-alpine

# Ultimate: Distroless
FROM gcr.io/distroless/python3

# Or scratch for static binaries
FROM scratch
COPY myapp /myapp
CMD ["/myapp"]
```

### Don't Store Secrets in Images

```dockerfile
# Bad: Secret in image
ENV API_KEY=secret123
COPY credentials.json /app/

# Good: Pass at runtime
# docker run -e API_KEY=secret myapp

# Good: Use Docker secrets (Swarm)
# docker secret create api_key key.txt

# Good: Use build secrets (BuildKit)
RUN --mount=type=secret,id=api_key \
    cat /run/secrets/api_key > /tmp/key && \
    ./setup.sh && \
    rm /tmp/key
```

### Scan Images for Vulnerabilities

```bash
# Docker Scout (built-in)
docker scout cves myimage:latest
docker scout recommendations myimage:latest

# Trivy
trivy image myimage:latest

# Snyk
snyk container test myimage:latest

# Grype
grype myimage:latest
```

### Verify Image Integrity

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Pull verified images only
docker pull nginx  # Fails if not signed

# Sign images when pushing
docker push myregistry/myapp:latest  # Signs automatically

# Check image digest
docker images --digests
docker pull nginx@sha256:abc123...
```

## Container Security

### Run as Non-Root

```dockerfile
# Create non-root user
FROM node:20-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

USER appuser

CMD ["node", "server.js"]
```

```bash
# Run as specific user
docker run --user 1000:1000 myapp
docker run --user nobody myapp
```

### Read-Only Root Filesystem

```bash
# Read-only container
docker run --read-only nginx

# With writable temp directories
docker run --read-only \
    --tmpfs /tmp \
    --tmpfs /var/cache/nginx \
    nginx

# In Compose
services:
  app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
```

### Drop Capabilities

```bash
# Drop all capabilities
docker run --cap-drop=ALL nginx

# Add only needed capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Common capabilities to drop
# - SYS_ADMIN (very powerful)
# - NET_ADMIN (network admin)
# - SYS_PTRACE (process tracing)
```

### Security Options

```bash
# No new privileges
docker run --security-opt=no-new-privileges nginx

# AppArmor profile
docker run --security-opt apparmor=docker-default nginx

# Seccomp profile
docker run --security-opt seccomp=profile.json nginx

# SELinux labels
docker run --security-opt label=type:container_t nginx
```

### Resource Limits

```bash
# Limit memory
docker run -m 512m nginx

# Limit CPU
docker run --cpus=1 nginx

# Limit processes
docker run --pids-limit=100 nginx

# Prevent fork bombs
docker run --ulimit nproc=100 nginx
```

## Network Security

### Use User-Defined Networks

```bash
# Create isolated network
docker network create --internal backend

# Containers on internal network can't reach internet
docker run --network backend myapp
```

### Limit Port Exposure

```bash
# Bind to localhost only
docker run -p 127.0.0.1:8080:80 nginx

# Don't expose unnecessary ports
# Bad: -p 3306:3306 (database exposed)
# Good: Use internal network only
```

### Network Policies

```yaml
# Docker Compose with network isolation
services:
  frontend:
    networks:
      - frontend

  api:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend  # Only accessible from api

networks:
  frontend:
  backend:
    internal: true
```

## Secrets Management

### Docker Secrets (Swarm Mode)

```bash
# Create secret
echo "mypassword" | docker secret create db_password -

# Create from file
docker secret create api_key ./api_key.txt

# Use in service
docker service create \
    --secret db_password \
    --name myapp \
    myimage
```

### In Containers

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
```

```bash
# Access in container: /run/secrets/db_password
cat /run/secrets/db_password
```

### External Secret Managers

```bash
# HashiCorp Vault
docker run -e VAULT_TOKEN=$(vault token) myapp

# AWS Secrets Manager
docker run -e AWS_SECRET=$(aws secretsmanager get-secret-value ...) myapp

# Environment injection
docker run --env-file <(vault kv get -format=json secret/myapp | jq -r '.data | to_entries | .[] | "\(.key)=\(.value)"') myapp
```

## Docker Daemon Security

### Rootless Mode

```bash
# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Use rootless Docker
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
docker run hello-world
```

### TLS Authentication

```bash
# Generate certificates
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem

# Configure daemon
# /etc/docker/daemon.json
{
  "tls": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"]
}

# Connect with TLS
docker --tlsverify \
    --tlscacert=ca.pem \
    --tlscert=cert.pem \
    --tlskey=key.pem \
    -H=tcp://host:2376 ps
```

### Authorization Plugins

```bash
# Enable authorization plugin
{
  "authorization-plugins": ["plugin-name"]
}
```

## Dockerfile Security Best Practices

```dockerfile
# 1. Use specific versions
FROM node:20.10-alpine3.18

# 2. Update packages and clean up
RUN apk update && \
    apk upgrade && \
    apk add --no-cache package && \
    rm -rf /var/cache/apk/*

# 3. Copy only needed files
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/

# 4. Don't run as root
RUN adduser -D appuser
USER appuser

# 5. Use COPY instead of ADD
COPY config.json /app/

# 6. Set health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

# 7. Use multi-stage builds
FROM node:20 AS builder
RUN npm run build

FROM node:20-alpine AS production
COPY --from=builder /app/dist ./dist
```

## Security Scanning in CI/CD

```yaml
# GitHub Actions
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    severity: HIGH,CRITICAL
    exit-code: 1

- name: Scan with Snyk
  uses: snyk/actions/docker@master
  with:
    image: myapp:${{ github.sha }}
```

## Security Checklist

### Image

- [ ] Use minimal base image
- [ ] No secrets in image
- [ ] Scan for vulnerabilities
- [ ] Use specific version tags
- [ ] Sign images
- [ ] Multi-stage builds

### Container

- [ ] Run as non-root
- [ ] Read-only filesystem
- [ ] Drop capabilities
- [ ] Resource limits
- [ ] No privileged mode
- [ ] Security options enabled

### Network

- [ ] Use user-defined networks
- [ ] Limit port exposure
- [ ] Internal networks for databases
- [ ] TLS for external communication

### Host

- [ ] Keep Docker updated
- [ ] Use rootless mode if possible
- [ ] Enable TLS for remote API
- [ ] Limit Docker group membership

## Quick Reference

| Security Feature | Command/Option |
|-----------------|----------------|
| Non-root user | `USER appuser` |
| Read-only | `--read-only` |
| Drop capabilities | `--cap-drop=ALL` |
| No new privileges | `--security-opt=no-new-privileges` |
| Memory limit | `-m 512m` |
| CPU limit | `--cpus=1` |
| Seccomp | `--security-opt seccomp=profile.json` |
| Content trust | `DOCKER_CONTENT_TRUST=1` |

---

**Previous:** [16-registry.md](16-registry.md) | **Next:** [18-optimization.md](18-optimization.md)
