# Environment Variables in Docker

## Overview

Environment variables configure container behavior without modifying the image.

```
┌─────────────────────────────────────────────────────────────────┐
│                 Environment Variables Flow                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Build Time                      Run Time                       │
│   ┌──────────────┐               ┌──────────────┐               │
│   │     ARG      │               │     ENV      │               │
│   │ (Dockerfile) │               │   -e flag    │               │
│   └──────┬───────┘               │  .env file   │               │
│          │                       │   --env-file │               │
│          ▼                       └──────┬───────┘               │
│   ┌──────────────┐                      │                       │
│   │    Image     │                      │                       │
│   │  (ENV baked) │                      ▼                       │
│   └──────┬───────┘               ┌──────────────┐               │
│          │                       │  Container   │               │
│          └──────────────────────►│ (final env)  │               │
│                                  └──────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## ENV Instruction (Dockerfile)

### Basic Syntax

```dockerfile
# Single variable
ENV MY_VAR=value

# Multiple variables (new syntax)
ENV MY_VAR1=value1 \
    MY_VAR2=value2 \
    MY_VAR3=value3

# Old syntax (creates multiple layers)
ENV MY_VAR1 value1
ENV MY_VAR2 value2
```

### Using ENV in Dockerfile

```dockerfile
FROM node:20-alpine

# Set environment variables
ENV NODE_ENV=production \
    PORT=3000 \
    APP_HOME=/app

# Use in WORKDIR
WORKDIR $APP_HOME

# Use in RUN
RUN echo "Building for $NODE_ENV environment"

# Use in other instructions
COPY . $APP_HOME

EXPOSE $PORT

CMD ["node", "server.js"]
```

### ENV Persistence

```dockerfile
# ENV values persist in the final image
FROM alpine
ENV MY_VAR=value

# Container inherits MY_VAR
# docker run myimage env | grep MY_VAR
# MY_VAR=value
```

## ARG Instruction (Build-time)

### Basic Syntax

```dockerfile
# Define argument with default
ARG VERSION=latest

# Define argument without default
ARG BUILD_DATE

# Use argument
FROM node:${VERSION}
LABEL build_date=${BUILD_DATE}
```

### Passing Build Arguments

```bash
# Pass argument during build
docker build --build-arg VERSION=20 .
docker build --build-arg BUILD_DATE=$(date +%Y-%m-%d) .

# Multiple arguments
docker build \
    --build-arg VERSION=20 \
    --build-arg BUILD_DATE=$(date +%Y-%m-%d) \
    .
```

## ARG vs ENV

| Feature | ARG | ENV |
|---------|-----|-----|
| Available | Build time only | Build + Run time |
| Persists in image | No | Yes |
| Override | `--build-arg` | `-e` flag |
| In final container | No | Yes |

### Converting ARG to ENV

```dockerfile
# ARG only available during build
ARG VERSION=1.0

# Convert to ENV for runtime
ENV APP_VERSION=$VERSION

# Now APP_VERSION available in container
```

### Practical Example

```dockerfile
FROM node:20-alpine

# Build arguments
ARG NODE_ENV=production
ARG APP_VERSION=1.0.0

# Convert to environment variables
ENV NODE_ENV=$NODE_ENV \
    APP_VERSION=$APP_VERSION

WORKDIR /app
COPY . .
RUN npm ci --only=production

CMD ["node", "server.js"]
```

```bash
# Build with custom values
docker build \
    --build-arg NODE_ENV=development \
    --build-arg APP_VERSION=2.0.0 \
    -t myapp .
```

## Runtime Environment Variables

### Using -e Flag

```bash
# Single variable
docker run -e MY_VAR=value nginx

# Multiple variables
docker run -e VAR1=value1 -e VAR2=value2 nginx

# Pass host environment variable
docker run -e MY_VAR nginx
# Uses $MY_VAR from host

# Override Dockerfile ENV
docker run -e NODE_ENV=development myapp
```

### Using --env-file

```bash
# Create .env file
cat > app.env << EOF
DATABASE_URL=postgres://localhost:5432/db
REDIS_URL=redis://localhost:6379
SECRET_KEY=mysecretkey
DEBUG=false
EOF

# Use env file
docker run --env-file app.env myapp

# Multiple env files
docker run --env-file db.env --env-file app.env myapp
```

### .env File Format

```bash
# Variable assignment
KEY=value

# No spaces around =
DATABASE_HOST=localhost

# Quotes for values with spaces
MESSAGE="Hello World"

# Comments
# This is a comment

# Empty values
EMPTY_VAR=

# Multi-line (use quotes)
MULTI="line1\nline2"
```

## Environment Variable Precedence

```
┌─────────────────────────────────────────────────────────────────┐
│              Environment Variable Precedence                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Priority (highest to lowest):                                 │
│                                                                  │
│   1. docker run -e VAR=value    (highest)                       │
│   2. --env-file                                                 │
│   3. ENV in Dockerfile                                          │
│   4. Base image ENV             (lowest)                        │
│                                                                  │
│   Higher priority overrides lower                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Dockerfile: ENV MY_VAR=dockerfile_value

# Override with -e
docker run -e MY_VAR=override_value myapp
# MY_VAR = override_value
```

## Common Patterns

### Application Configuration

```dockerfile
FROM node:20-alpine

ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info \
    DATABASE_HOST=localhost \
    DATABASE_PORT=5432

WORKDIR /app
COPY . .
RUN npm ci --only=production

EXPOSE $PORT
CMD ["node", "server.js"]
```

```bash
# Run with production database
docker run -d \
    -e DATABASE_HOST=prod-db.example.com \
    -e DATABASE_PORT=5432 \
    -e LOG_LEVEL=warn \
    -p 3000:3000 \
    myapp
```

### Database Containers

```bash
# MySQL
docker run -d \
    -e MYSQL_ROOT_PASSWORD=rootpass \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_USER=user \
    -e MYSQL_PASSWORD=userpass \
    mysql:8

# PostgreSQL
docker run -d \
    -e POSTGRES_PASSWORD=password \
    -e POSTGRES_USER=user \
    -e POSTGRES_DB=mydb \
    postgres:15

# MongoDB
docker run -d \
    -e MONGO_INITDB_ROOT_USERNAME=admin \
    -e MONGO_INITDB_ROOT_PASSWORD=password \
    mongo:6
```

### Secrets (Sensitive Data)

```bash
# DON'T put secrets in Dockerfile
# Bad practice:
# ENV DATABASE_PASSWORD=secret123

# Better: Pass at runtime
docker run -e DATABASE_PASSWORD=secret123 myapp

# Best: Use Docker secrets (Swarm) or external secret manager
docker secret create db_password password.txt
docker service create --secret db_password myapp
```

## Docker Compose Environment Variables

### In docker-compose.yml

```yaml
version: '3.8'
services:
  app:
    image: myapp
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DATABASE_URL=postgres://db:5432/mydb

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
```

### Using .env File

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    image: myapp
    env_file:
      - .env
      - .env.local
    environment:
      # Override specific variables
      - NODE_ENV=production
```

### Variable Substitution

```bash
# .env file
TAG=latest
PORT=3000
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    image: myapp:${TAG}
    ports:
      - "${PORT}:3000"
    environment:
      - APP_PORT=${PORT}
```

```bash
# Run with defaults from .env
docker-compose up

# Override from shell
TAG=v2.0 PORT=8080 docker-compose up
```

## Viewing Environment Variables

```bash
# View all environment variables in container
docker exec container_name env
docker exec container_name printenv

# View specific variable
docker exec container_name printenv MY_VAR

# Inspect container config
docker inspect --format '{{.Config.Env}}' container_name

# Pretty print
docker inspect container_name | jq '.[0].Config.Env'
```

## Best Practices

### 1. Use Meaningful Defaults

```dockerfile
ENV NODE_ENV=production \
    LOG_LEVEL=info \
    PORT=3000
```

### 2. Document Expected Variables

```dockerfile
# Required environment variables:
# - DATABASE_URL: PostgreSQL connection string
# - SECRET_KEY: Application secret key
# Optional:
# - LOG_LEVEL: Logging level (default: info)
# - PORT: Server port (default: 3000)
```

### 3. Validate at Startup

```javascript
// In application code
const required = ['DATABASE_URL', 'SECRET_KEY'];
required.forEach(key => {
    if (!process.env[key]) {
        console.error(`Missing required env var: ${key}`);
        process.exit(1);
    }
});
```

### 4. Never Hardcode Secrets

```dockerfile
# Bad
ENV DATABASE_PASSWORD=secret123

# Good - set at runtime
# docker run -e DATABASE_PASSWORD=secret myapp
```

### 5. Use .env for Development

```bash
# .env (git ignored)
DATABASE_URL=postgres://localhost:5432/dev
SECRET_KEY=dev-secret-key
DEBUG=true

# .env.example (committed)
DATABASE_URL=
SECRET_KEY=
DEBUG=false
```

## Quick Reference

| Method | Scope | Example |
|--------|-------|---------|
| `ENV` | Build + Runtime | `ENV KEY=value` |
| `ARG` | Build only | `ARG KEY=value` |
| `-e` | Runtime | `docker run -e KEY=value` |
| `--env-file` | Runtime | `docker run --env-file .env` |
| Compose `environment` | Runtime | `environment: KEY=value` |
| Compose `env_file` | Runtime | `env_file: .env` |

---

**Previous:** [07-cmd-vs-entrypoint.md](07-cmd-vs-entrypoint.md) | **Next:** [09-building-images.md](09-building-images.md)
