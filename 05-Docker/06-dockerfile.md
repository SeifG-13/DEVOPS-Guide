# Dockerfile

## What is a Dockerfile?

A Dockerfile is a text document containing instructions to build a Docker image. Each instruction creates a layer in the image.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Dockerfile → Image                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Dockerfile            Build              Image                 │
│   ┌──────────────┐      ┌─────┐       ┌──────────────┐         │
│   │ FROM node    │      │     │       │   Layer 4    │         │
│   │ WORKDIR /app │ ────►│Build│──────►│   Layer 3    │         │
│   │ COPY . .     │      │     │       │   Layer 2    │         │
│   │ RUN npm      │      └─────┘       │   Layer 1    │         │
│   │ CMD ["node"] │                    └──────────────┘         │
│   └──────────────┘                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Basic Structure

```dockerfile
# Comment
INSTRUCTION arguments

# Example
FROM ubuntu:22.04
LABEL maintainer="you@example.com"
RUN apt-get update && apt-get install -y nginx
COPY index.html /var/www/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## Dockerfile Instructions

### FROM

Specifies the base image.

```dockerfile
# Official image
FROM ubuntu:22.04

# Specific version
FROM node:20-alpine

# Minimal base
FROM alpine:3.18

# Empty base (for static binaries)
FROM scratch

# Multiple FROM (multi-stage)
FROM node:20 AS builder
FROM nginx:alpine AS production
```

### LABEL

Adds metadata to the image.

```dockerfile
LABEL maintainer="you@example.com"
LABEL version="1.0"
LABEL description="My application"

# Multiple labels
LABEL maintainer="you@example.com" \
      version="1.0" \
      description="My application"

# OCI standard labels
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.authors="you@example.com"
```

### RUN

Executes commands during build.

```dockerfile
# Shell form
RUN apt-get update

# Exec form
RUN ["apt-get", "update"]

# Multiple commands (single layer)
RUN apt-get update && \
    apt-get install -y \
        curl \
        nginx \
        vim && \
    rm -rf /var/lib/apt/lists/*

# Best practice: chain commands to reduce layers
RUN apt-get update && apt-get install -y nginx && apt-get clean
```

### COPY

Copies files from build context to image.

```dockerfile
# Copy single file
COPY app.js /app/

# Copy multiple files
COPY package.json package-lock.json /app/

# Copy directory
COPY src/ /app/src/

# Copy with wildcard
COPY *.json /app/

# Copy and rename
COPY app.js /app/server.js

# Copy with ownership
COPY --chown=node:node . /app/

# Copy from build stage
COPY --from=builder /app/dist /usr/share/nginx/html
```

### ADD

Similar to COPY but with extra features.

```dockerfile
# Copy file (same as COPY)
ADD app.js /app/

# Auto-extract tar archive
ADD app.tar.gz /app/

# Download from URL (not recommended)
ADD https://example.com/file.tar.gz /app/

# Prefer COPY over ADD unless you need tar extraction
```

### WORKDIR

Sets the working directory.

```dockerfile
WORKDIR /app

# Create directory if it doesn't exist
WORKDIR /app/src

# Multiple WORKDIR (relative paths work)
WORKDIR /app
WORKDIR src      # Results in /app/src
WORKDIR tests    # Results in /app/src/tests
```

### ENV

Sets environment variables.

```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV NODE_ENV=production \
    PORT=3000 \
    DEBUG=false

# Use in subsequent instructions
ENV APP_HOME=/app
WORKDIR $APP_HOME
```

### ARG

Defines build-time variables.

```dockerfile
# Define argument
ARG NODE_VERSION=20

# Use in FROM
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}

# Use in other instructions
ARG APP_VERSION
LABEL version=${APP_VERSION}

# Default value
ARG PORT=3000
ENV PORT=$PORT

# Build with argument
# docker build --build-arg NODE_VERSION=18 .
```

### EXPOSE

Documents exposed ports (does not publish).

```dockerfile
# Single port
EXPOSE 80

# Multiple ports
EXPOSE 80 443

# UDP port
EXPOSE 53/udp

# TCP port (default)
EXPOSE 3000/tcp
```

### USER

Sets the user for subsequent instructions.

```dockerfile
# Create and switch user
RUN useradd -m appuser
USER appuser

# Use existing user
USER node

# Use UID
USER 1000

# Use user:group
USER appuser:appgroup
```

### VOLUME

Creates mount point for volumes.

```dockerfile
# Single volume
VOLUME /data

# Multiple volumes
VOLUME ["/data", "/logs"]

# Volume for database
VOLUME /var/lib/mysql
```

### HEALTHCHECK

Defines container health check.

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# Custom health check
HEALTHCHECK --interval=1m --timeout=10s \
    CMD /app/healthcheck.sh

# Disable health check
HEALTHCHECK NONE
```

| Option | Description | Default |
|--------|-------------|---------|
| `--interval` | Time between checks | 30s |
| `--timeout` | Check timeout | 30s |
| `--start-period` | Startup grace period | 0s |
| `--retries` | Consecutive failures | 3 |

### SHELL

Changes the default shell.

```dockerfile
# Default: ["/bin/sh", "-c"]

# Change to bash
SHELL ["/bin/bash", "-c"]

# PowerShell (Windows)
SHELL ["powershell", "-Command"]
```

### STOPSIGNAL

Sets the signal to stop the container.

```dockerfile
STOPSIGNAL SIGTERM
STOPSIGNAL SIGQUIT
STOPSIGNAL 9
```

## Complete Examples

### Node.js Application

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Production stage
FROM node:20-alpine
WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy dependencies
COPY --from=builder /app/node_modules ./node_modules

# Copy application
COPY --chown=appuser:appgroup . .

# Set user
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

# Start application
CMD ["node", "server.js"]
```

### Python Application

```dockerfile
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Create non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# Copy application
COPY --chown=appuser:appuser . .

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### Nginx Static Site

```dockerfile
FROM nginx:alpine

# Remove default config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom config
COPY nginx.conf /etc/nginx/conf.d/

# Copy static files
COPY --chown=nginx:nginx dist/ /usr/share/nginx/html/

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

### Go Application

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Download dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build application
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/main .

# Production stage
FROM scratch

# Copy binary
COPY --from=builder /app/main /main

# Copy CA certificates (for HTTPS)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080

ENTRYPOINT ["/main"]
```

## Instruction Summary

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image |
| `LABEL` | Metadata |
| `RUN` | Execute commands |
| `COPY` | Copy files |
| `ADD` | Copy with extras |
| `WORKDIR` | Working directory |
| `ENV` | Environment variables |
| `ARG` | Build arguments |
| `EXPOSE` | Document ports |
| `USER` | Set user |
| `VOLUME` | Create mount point |
| `HEALTHCHECK` | Health check |
| `CMD` | Default command |
| `ENTRYPOINT` | Container entry point |

---

**Previous:** [05-images.md](05-images.md) | **Next:** [07-cmd-vs-entrypoint.md](07-cmd-vs-entrypoint.md)
