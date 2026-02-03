# Building Docker Images

## docker build Command

### Basic Build

```bash
# Build from current directory
docker build .

# Build with tag
docker build -t myapp:1.0 .

# Build with multiple tags
docker build -t myapp:1.0 -t myapp:latest .

# Build from specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build from URL
docker build https://github.com/user/repo.git
docker build https://github.com/user/repo.git#branch
docker build https://github.com/user/repo.git#:directory
```

### Build Options

| Option | Description | Example |
|--------|-------------|---------|
| `-t, --tag` | Name and tag | `-t myapp:1.0` |
| `-f, --file` | Dockerfile path | `-f Dockerfile.prod` |
| `--build-arg` | Build argument | `--build-arg VERSION=1.0` |
| `--no-cache` | Don't use cache | `--no-cache` |
| `--pull` | Always pull base image | `--pull` |
| `--target` | Build stage target | `--target builder` |
| `--platform` | Target platform | `--platform linux/amd64` |
| `-q, --quiet` | Suppress output | `-q` |
| `--progress` | Progress output | `--progress=plain` |

## Build Context

The build context is the set of files sent to the Docker daemon for building.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Build Context                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Project Directory (Context)                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Dockerfile                                              │   │
│   │  .dockerignore                                           │   │
│   │  src/                                                    │   │
│   │  package.json          ──────────►  Docker Daemon       │   │
│   │  node_modules/  ✗ (excluded)        (builds image)      │   │
│   │  .git/          ✗ (excluded)                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   docker build -t myapp .                                       │
│                       └── Context path                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Context Size Matters

```bash
# Check context size
docker build . 2>&1 | head -5
# Sending build context to Docker daemon  125MB

# Large context = slow builds
# Use .dockerignore to exclude files
```

## .dockerignore

Excludes files from the build context.

### Basic .dockerignore

```dockerignore
# Dependencies
node_modules/
vendor/
__pycache__/
*.pyc

# Build outputs
dist/
build/
*.o
*.a

# Version control
.git/
.gitignore
.svn/

# IDE/Editor
.idea/
.vscode/
*.swp
*.swo

# Docker files
Dockerfile*
docker-compose*
.docker/

# Environment files
.env
.env.*
*.local

# Documentation
README.md
docs/
*.md

# Tests
tests/
test/
*.test.js
coverage/

# Logs
*.log
logs/

# OS files
.DS_Store
Thumbs.db
```

### Pattern Syntax

| Pattern | Description |
|---------|-------------|
| `#` | Comment |
| `*` | Any sequence (except /) |
| `?` | Single character |
| `**` | Any directory depth |
| `!` | Negate exclusion |

```dockerignore
# Ignore all .md files
*.md

# But include README.md
!README.md

# Ignore all in temp directories
**/temp/*

# Ignore specific directory
mydir/

# Ignore by pattern
*.log
*.tmp
```

## Multi-Stage Builds

Build efficient images by separating build and runtime stages.

### Why Multi-Stage?

```
┌─────────────────────────────────────────────────────────────────┐
│                    Multi-Stage Benefits                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Single Stage:                Multi-Stage:                      │
│   ┌──────────────┐            ┌──────────────┐                  │
│   │ Build Tools  │            │ Build Tools  │ Stage 1          │
│   │ Source Code  │            │ Source Code  │ (builder)        │
│   │ Dependencies │            │ Dependencies │                  │
│   │ Test Files   │            └──────┬───────┘                  │
│   │ Final App    │                   │ COPY artifacts           │
│   └──────────────┘            ┌──────▼───────┐                  │
│   Size: 1.2 GB                │ Final App    │ Stage 2          │
│                               │ Runtime Only │ (production)     │
│                               └──────────────┘                  │
│                               Size: 150 MB                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Stage Example

```dockerfile
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Building Specific Stage

```bash
# Build only builder stage
docker build --target builder -t myapp:builder .

# Build production stage (default - last stage)
docker build -t myapp:prod .
```

### Advanced Multi-Stage

```dockerfile
# Base stage
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./

# Dependencies stage
FROM base AS deps
RUN npm ci --only=production
RUN cp -R node_modules /prod_modules
RUN npm ci

# Build stage
FROM deps AS build
COPY . .
RUN npm run build
RUN npm run test

# Production stage
FROM base AS production
COPY --from=deps /prod_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]

# Development stage
FROM deps AS development
COPY . .
CMD ["npm", "run", "dev"]
```

## Build Arguments

### Using ARG

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG APP_VERSION=1.0.0
ARG BUILD_DATE

LABEL version=${APP_VERSION}
LABEL build_date=${BUILD_DATE}

ENV APP_VERSION=${APP_VERSION}
```

### Passing Arguments

```bash
# Single argument
docker build --build-arg NODE_VERSION=18 -t myapp .

# Multiple arguments
docker build \
    --build-arg NODE_VERSION=18 \
    --build-arg APP_VERSION=2.0.0 \
    --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
    -t myapp .
```

## Build Cache

### How Cache Works

```dockerfile
FROM node:20-alpine          # Cache layer 1
WORKDIR /app                  # Cache layer 2
COPY package*.json ./         # Cache layer 3 (invalidated if files change)
RUN npm ci                    # Cache layer 4 (invalidated if layer 3 changes)
COPY . .                      # Cache layer 5 (usually invalidated)
RUN npm run build             # Cache layer 6
```

### Cache Optimization

```dockerfile
# Good: Copy dependency files first
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./         # Changes rarely
RUN npm ci                    # Cached if package.json unchanged
COPY . .                      # Changes often
RUN npm run build

# Bad: Copy everything first
FROM node:20-alpine
WORKDIR /app
COPY . .                      # Any change invalidates cache
RUN npm ci                    # Always rebuilds
RUN npm run build
```

### Cache Control

```bash
# Disable cache
docker build --no-cache -t myapp .

# Pull latest base image
docker build --pull -t myapp .

# Use cache from specific image
docker build --cache-from myapp:latest -t myapp:new .
```

## BuildKit

BuildKit is the next-generation builder with advanced features.

### Enable BuildKit

```bash
# Environment variable
export DOCKER_BUILDKIT=1
docker build -t myapp .

# Or in daemon.json
{
  "features": {
    "buildkit": true
  }
}
```

### BuildKit Features

```dockerfile
# syntax=docker/dockerfile:1

# Mount cache for package managers
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Mount secrets (not stored in image)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci

# Mount SSH for private repos
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git
```

### Build with Secrets

```bash
# Build with secret file
docker build --secret id=npmrc,src=.npmrc -t myapp .

# Build with SSH
docker build --ssh default -t myapp .
```

## Cross-Platform Builds

### Building for Different Platforms

```bash
# Build for specific platform
docker build --platform linux/amd64 -t myapp:amd64 .
docker build --platform linux/arm64 -t myapp:arm64 .

# Multi-platform build (requires buildx)
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t myapp:latest \
    --push .
```

### Using buildx

```bash
# Create builder
docker buildx create --name mybuilder --use

# Build multi-platform
docker buildx build \
    --platform linux/amd64,linux/arm64,linux/arm/v7 \
    -t myregistry/myapp:latest \
    --push .

# Inspect builder
docker buildx inspect

# List builders
docker buildx ls
```

## Best Practices

### 1. Order Instructions by Change Frequency

```dockerfile
# Least frequent changes first
FROM node:20-alpine
WORKDIR /app

# Dependencies (change occasionally)
COPY package*.json ./
RUN npm ci --only=production

# Application code (changes frequently)
COPY . .

CMD ["node", "server.js"]
```

### 2. Combine RUN Commands

```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y nginx
RUN apt-get clean

# Good: Single layer
RUN apt-get update && \
    apt-get install -y \
        curl \
        nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 3. Use Specific Base Image Tags

```dockerfile
# Bad: Unpredictable
FROM node:latest

# Good: Predictable
FROM node:20.10-alpine3.18
```

### 4. Clean Up in Same Layer

```dockerfile
# Clean up in same RUN to reduce layer size
RUN apt-get update && \
    apt-get install -y build-essential && \
    make && \
    make install && \
    apt-get remove -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker build .` | Build from current dir |
| `docker build -t name:tag .` | Build with tag |
| `docker build -f Dockerfile.dev .` | Use specific Dockerfile |
| `docker build --no-cache .` | Build without cache |
| `docker build --target stage .` | Build specific stage |
| `docker build --build-arg K=V .` | Pass build argument |
| `docker build --platform P .` | Build for platform |

---

**Previous:** [08-environment-variables.md](08-environment-variables.md) | **Next:** [10-containers-deep-dive.md](10-containers-deep-dive.md)
