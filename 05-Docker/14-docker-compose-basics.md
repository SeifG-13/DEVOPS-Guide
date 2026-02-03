# Docker Compose Basics

## What is Docker Compose?

Docker Compose is a tool for defining and running multi-container applications using a YAML file.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Compose                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   docker-compose.yml                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ services:                                                │   │
│   │   web:                        ┌─────────┐               │   │
│   │     image: nginx        ─────►│   web   │               │   │
│   │   api:                        └─────────┘               │   │
│   │     build: ./api        ─────►┌─────────┐               │   │
│   │   db:                         │   api   │               │   │
│   │     image: postgres     ─────►└─────────┘               │   │
│   │                               ┌─────────┐               │   │
│   │ networks:               ─────►│   db    │               │   │
│   │   mynet:                      └─────────┘               │   │
│   │                                    │                    │   │
│   │ volumes:                           │                    │   │
│   │   data:                 ──────────►│ data volume        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   docker compose up   →   Creates network, volumes, containers   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Installation

```bash
# Docker Compose V2 (plugin) - Recommended
# Included with Docker Desktop
# Or install separately:
sudo apt-get install docker-compose-plugin

# Verify installation
docker compose version

# Docker Compose V1 (standalone) - Legacy
# Command: docker-compose (with hyphen)
```

## Basic docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"

  api:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - ./api:/app
    command: npm start
    ports:
      - "3000:3000"

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

## Compose File Structure

```yaml
version: '3.8'          # Compose file version

services:               # Container definitions
  service_name:
    image: ...
    build: ...
    ports: ...
    volumes: ...
    environment: ...
    depends_on: ...
    networks: ...

networks:               # Network definitions
  network_name:
    driver: ...

volumes:                # Volume definitions
  volume_name:
    driver: ...

configs:                # Config definitions (Swarm)
secrets:                # Secret definitions (Swarm)
```

## Service Configuration

### Image vs Build

```yaml
services:
  # Use existing image
  web:
    image: nginx:alpine

  # Build from Dockerfile
  api:
    build: ./api

  # Build with options
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production
      target: production
```

### Ports

```yaml
services:
  web:
    image: nginx
    ports:
      # HOST:CONTAINER
      - "80:80"
      - "443:443"

      # Container port only (random host port)
      - "3000"

      # Specific interface
      - "127.0.0.1:8080:80"

      # Port range
      - "8000-8010:8000-8010"

      # UDP
      - "53:53/udp"
```

### Volumes

```yaml
services:
  app:
    image: myapp
    volumes:
      # Named volume
      - data:/app/data

      # Bind mount
      - ./src:/app/src

      # Read-only
      - ./config:/app/config:ro

      # Anonymous volume
      - /app/node_modules

volumes:
  data:
```

### Environment Variables

```yaml
services:
  app:
    image: myapp
    environment:
      # Key=value
      NODE_ENV: production
      PORT: 3000

      # Or as list
      - NODE_ENV=production
      - PORT=3000

    # From file
    env_file:
      - .env
      - .env.local
```

### Networks

```yaml
services:
  web:
    image: nginx
    networks:
      - frontend

  api:
    image: myapi
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
```

### Dependencies

```yaml
services:
  web:
    image: nginx
    depends_on:
      - api

  api:
    image: myapi
    depends_on:
      - db
      - redis

  db:
    image: postgres

  redis:
    image: redis

# Start order: db, redis → api → web
# Note: doesn't wait for services to be "ready"
```

## Basic Commands

### Starting Services

```bash
# Start all services
docker compose up

# Start in detached mode
docker compose up -d

# Start specific services
docker compose up web api

# Build before starting
docker compose up --build

# Force recreate containers
docker compose up --force-recreate

# Remove orphan containers
docker compose up --remove-orphans
```

### Stopping Services

```bash
# Stop services
docker compose stop

# Stop and remove containers, networks
docker compose down

# Also remove volumes
docker compose down -v

# Also remove images
docker compose down --rmi all

# Stop specific service
docker compose stop web
```

### Managing Services

```bash
# List services
docker compose ps

# View logs
docker compose logs

# Follow logs
docker compose logs -f

# Logs for specific service
docker compose logs api

# Tail logs
docker compose logs --tail 100

# Execute command
docker compose exec api sh
docker compose exec db psql -U postgres

# Run one-off command
docker compose run api npm test
docker compose run --rm api npm test
```

### Building

```bash
# Build all services
docker compose build

# Build specific service
docker compose build api

# Build without cache
docker compose build --no-cache

# Pull images
docker compose pull
```

### Scaling

```bash
# Scale service
docker compose up -d --scale api=3

# View scaled services
docker compose ps
```

## Practical Examples

### Web Application Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app

  app:
    build: ./app
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Development Environment

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    command: npm run dev

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: devpass
    ports:
      - "5432:5432"
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```

### WordPress Stack

```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      - db

  db:
    image: mysql:8
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - db_data:/var/lib/mysql

volumes:
  wordpress_data:
  db_data:
```

## Variable Substitution

### Using Variables

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: myapp:${TAG:-latest}
    ports:
      - "${PORT:-3000}:3000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
```

```bash
# .env file (auto-loaded)
TAG=v1.0
PORT=8080
DATABASE_URL=postgres://localhost/mydb

# Or set in shell
TAG=v2.0 docker compose up
```

### .env File

```bash
# .env
COMPOSE_PROJECT_NAME=myproject
TAG=latest
PORT=3000
DB_PASSWORD=secret
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker compose up` | Start services |
| `docker compose up -d` | Start detached |
| `docker compose down` | Stop and remove |
| `docker compose ps` | List services |
| `docker compose logs` | View logs |
| `docker compose exec` | Execute command |
| `docker compose build` | Build services |
| `docker compose pull` | Pull images |
| `docker compose stop` | Stop services |
| `docker compose restart` | Restart services |

---

**Previous:** [13-volumes.md](13-volumes.md) | **Next:** [15-docker-compose-advanced.md](15-docker-compose-advanced.md)
