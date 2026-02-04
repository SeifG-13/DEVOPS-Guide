# Containerized Builds

## Why Containerized Builds?

```
┌─────────────────────────────────────────────────────────────────┐
│              Containerized Build Benefits                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Traditional Builds          Containerized Builds              │
│   ┌──────────────────┐       ┌──────────────────┐              │
│   │  Build Server    │       │  Build Container │              │
│   │  • JDK 8,11,17  │       │  • Isolated env  │              │
│   │  • Node 16,18,20│       │  • Reproducible  │              │
│   │  • Python 3.x   │       │  • Consistent    │              │
│   │  • Conflicts!   │       │  • Portable      │              │
│   └──────────────────┘       └──────────────────┘              │
│                                                                  │
│   Issues:                    Benefits:                          │
│   • Version conflicts        • No conflicts                     │
│   • "Works on my machine"    • Same everywhere                  │
│   • Hard to maintain         • Easy to update                   │
│   • State pollution          • Clean every time                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Docker-Based Builds

### Build Inside Container

```yaml
# GitHub Actions
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: node:20-alpine
      options: --user root
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
```

### Jenkins Docker Agent

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

### Custom Build Image

```dockerfile
# Dockerfile.build
FROM node:20-alpine

# Install additional tools
RUN apk add --no-cache git python3 make g++

# Install global dependencies
RUN npm install -g typescript eslint

WORKDIR /app

# Cache dependencies
COPY package*.json ./
RUN npm ci

# Copy source
COPY . .

CMD ["npm", "run", "build"]
```

```yaml
# GitHub Actions using custom image
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: myregistry/build-image:latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

## Multi-Stage Builds

### Optimized Docker Build

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS production

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -S -u 1001 -G appgroup appuser

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Java Multi-Stage Build

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -g 1001 app && \
    adduser -S -u 1001 -G app app

COPY --from=build /app/target/*.jar app.jar

USER app

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Python Multi-Stage Build

```dockerfile
# Build stage
FROM python:3.11-slim AS builder

WORKDIR /app

RUN pip install poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt -o requirements.txt --without-hashes

COPY . .
RUN poetry build

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --from=builder /app/dist/*.whl .
RUN pip install *.whl && rm *.whl

USER nobody

CMD ["python", "-m", "myapp"]
```

## Docker Compose for Builds

### Build with Dependencies

```yaml
# docker-compose.yml
services:
  build:
    build:
      context: .
      dockerfile: Dockerfile.build
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    depends_on:
      - redis
      - postgres

  redis:
    image: redis:7-alpine

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test

volumes:
  node_modules:
```

### Integration Testing

```yaml
# docker-compose.test.yml
services:
  test:
    build:
      context: .
      target: test
    environment:
      DATABASE_URL: postgres://test:test@db:5432/testdb
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: npm test

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
```

```yaml
# CI Pipeline
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker-compose -f docker-compose.test.yml up --exit-code-from test
      - run: docker-compose -f docker-compose.test.yml down -v
```

## BuildKit Features

### Enable BuildKit

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Or in Docker daemon.json
{
  "features": {
    "buildkit": true
  }
}
```

### Cache Mounts

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-alpine

WORKDIR /app

# Cache npm packages
RUN --mount=type=cache,target=/root/.npm \
    npm install -g typescript

COPY package*.json ./

# Cache node_modules
RUN --mount=type=cache,target=/app/node_modules \
    npm ci

COPY . .
RUN npm run build
```

### Secret Mounts

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-alpine

WORKDIR /app

# Use secret during build (not stored in image)
RUN --mount=type=secret,id=npm_token \
    npm config set //registry.npmjs.org/:_authToken=$(cat /run/secrets/npm_token) && \
    npm ci
```

```bash
# Build with secret
docker build --secret id=npm_token,src=.npmrc .
```

### SSH Mount for Private Repos

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-alpine

RUN apk add --no-cache git openssh

# Clone private repo using SSH
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git
```

```bash
# Build with SSH
docker build --ssh default .
```

## CI/CD Integration

### GitHub Actions with Docker

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          load: true
          tags: myapp:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Test
        run: |
          docker run --rm myapp:test npm test

      - uses: docker/login-action@v3
        if: github.ref == 'refs/heads/main'
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        if: github.ref == 'refs/heads/main'
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

### Jenkins Docker Pipeline

```groovy
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    docker.image("${IMAGE_NAME}:${BUILD_NUMBER}").inside {
                        sh 'npm test'
                    }
                }
            }
        }

        stage('Push') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-creds') {
                        docker.image("${IMAGE_NAME}:${BUILD_NUMBER}").push()
                        docker.image("${IMAGE_NAME}:${BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }
    }
}
```

## Build Caching Strategies

### Layer Caching

```dockerfile
# Order matters - least changing first
FROM node:20-alpine

WORKDIR /app

# 1. Copy package files first (changes rarely)
COPY package*.json ./

# 2. Install dependencies (cached if package.json unchanged)
RUN npm ci

# 3. Copy source code (changes frequently)
COPY . .

# 4. Build
RUN npm run build
```

### Registry Cache

```yaml
# GitHub Actions
- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=registry,ref=myregistry/myapp:buildcache
    cache-to: type=registry,ref=myregistry/myapp:buildcache,mode=max
```

### GitHub Actions Cache

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Local Cache

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=local,src=/tmp/.buildx-cache
    cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

# Rotate cache
- run: |
    rm -rf /tmp/.buildx-cache
    mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

## Security Best Practices

### Non-Root User

```dockerfile
FROM node:20-alpine

# Create app user
RUN addgroup -g 1001 appgroup && \
    adduser -S -u 1001 -G appgroup appuser

WORKDIR /app
COPY --chown=appuser:appgroup . .

USER appuser

CMD ["node", "index.js"]
```

### Read-Only Filesystem

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

### Image Scanning

```yaml
# GitHub Actions
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:latest'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

## Complete Containerized CI Pipeline

```yaml
name: Containerized Build

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      # Build test image
      - uses: docker/build-push-action@v5
        with:
          context: .
          target: test
          load: true
          tags: ${{ env.IMAGE_NAME }}:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Run tests in container
      - name: Run Tests
        run: |
          docker run --rm ${{ env.IMAGE_NAME }}:test npm test

      # Security scan
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:test
          format: 'sarif'
          output: 'trivy.sarif'

      - uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy.sarif'

      # Build production image
      - uses: docker/build-push-action@v5
        id: build
        with:
          context: .
          target: production
          load: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}

      # Push to registry
      - uses: docker/login-action@v3
        if: github.event_name != 'pull_request'
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        if: github.event_name != 'pull_request'
        with:
          context: .
          target: production
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
```

## Quick Reference

### Build Commands

| Task | Command |
|------|---------|
| Build image | `docker build -t myapp .` |
| Build with BuildKit | `DOCKER_BUILDKIT=1 docker build .` |
| Build specific target | `docker build --target test .` |
| Build with cache | `docker build --cache-from myapp:cache .` |

### Dockerfile Best Practices

| Practice | Reason |
|----------|--------|
| Multi-stage builds | Smaller images |
| Non-root user | Security |
| Layer ordering | Better caching |
| .dockerignore | Faster builds |
| Specific base tags | Reproducibility |

### Cache Types

| Type | Use Case |
|------|----------|
| `gha` | GitHub Actions |
| `registry` | Shared across machines |
| `local` | Single machine |
| `inline` | Embed in image |

---

**Previous:** [18-artifact-management.md](18-artifact-management.md) | **Next:** [20-deployment-strategies.md](20-deployment-strategies.md)
