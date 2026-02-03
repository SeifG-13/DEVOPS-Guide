# Docker Compose Advanced

## Health Checks

### Service Health Check

```yaml
version: '3.8'

services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  api:
    image: myapi
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### depends_on with Health Check

```yaml
version: '3.8'

services:
  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:alpine
```

## Environment Configuration

### Multiple env_file

```yaml
services:
  app:
    image: myapp
    env_file:
      - .env                 # Base config
      - .env.${ENV:-dev}     # Environment-specific
      - .env.local           # Local overrides (gitignored)
```

### Variable Priority

```
┌─────────────────────────────────────────────────────────────────┐
│              Environment Variable Priority                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Highest Priority                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 1. Compose file (environment:)                          │   │
│   │ 2. Shell environment variables                          │   │
│   │ 3. env_file                                             │   │
│   │ 4. Dockerfile (ENV)                                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│   Lowest Priority                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Profiles

Run different sets of services for different scenarios.

```yaml
version: '3.8'

services:
  app:
    image: myapp
    # Always runs (no profile)

  db:
    image: postgres
    # Always runs

  debug:
    image: debug-tools
    profiles:
      - debug

  test-db:
    image: postgres
    profiles:
      - test

  monitoring:
    image: prometheus
    profiles:
      - monitoring
      - production
```

```bash
# Run default services only
docker compose up

# Run with debug profile
docker compose --profile debug up

# Run with multiple profiles
docker compose --profile debug --profile monitoring up

# Set via environment
COMPOSE_PROFILES=debug,monitoring docker compose up
```

## Extends and Override

### extends (Legacy)

```yaml
# common.yml
services:
  app:
    image: myapp
    environment:
      - NODE_ENV=production

# docker-compose.yml
services:
  web:
    extends:
      file: common.yml
      service: app
    ports:
      - "80:80"
```

### Override Files

```yaml
# docker-compose.yml (base)
version: '3.8'
services:
  app:
    image: myapp:latest
    ports:
      - "3000:3000"

# docker-compose.override.yml (auto-loaded)
version: '3.8'
services:
  app:
    build: .
    volumes:
      - .:/app
    environment:
      - DEBUG=true

# docker-compose.prod.yml
version: '3.8'
services:
  app:
    image: myapp:${TAG}
    deploy:
      replicas: 3
```

```bash
# Development (auto-loads override)
docker compose up

# Production (explicit files)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

## Resource Limits

```yaml
version: '3.8'

services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  db:
    image: postgres
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
```

## Restart Policies

```yaml
version: '3.8'

services:
  web:
    image: nginx
    restart: always

  api:
    image: myapi
    restart: unless-stopped

  worker:
    image: worker
    restart: on-failure

  batch:
    image: batch-job
    restart: "no"

  # With max retries
  task:
    image: task-runner
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```

## Networking Advanced

### Custom Networks

```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      frontend:
        aliases:
          - web-service
        ipv4_address: 172.20.0.10

  api:
    image: myapi
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      backend:
        aliases:
          - database
          - db-service

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

  backend:
    driver: bridge
    internal: true  # No external access
```

### External Networks

```yaml
version: '3.8'

services:
  app:
    image: myapp
    networks:
      - existing_network

networks:
  existing_network:
    external: true
    name: my_existing_network
```

## Volumes Advanced

### Named Volume with Options

```yaml
version: '3.8'

services:
  db:
    image: postgres
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=10.40.0.199,nolock,soft,rw
      device: ":/docker/data"
```

### External Volumes

```yaml
version: '3.8'

services:
  app:
    image: myapp
    volumes:
      - external_data:/data

volumes:
  external_data:
    external: true
    name: my_existing_volume
```

## Secrets and Configs

### Docker Secrets (Swarm Mode)

```yaml
version: '3.8'

services:
  app:
    image: myapp
    secrets:
      - db_password
      - source: api_key
        target: /run/secrets/api_key
        mode: 0400

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
```

### Configs

```yaml
version: '3.8'

services:
  nginx:
    image: nginx
    configs:
      - source: nginx_conf
        target: /etc/nginx/nginx.conf

configs:
  nginx_conf:
    file: ./nginx.conf
```

## Logging

```yaml
version: '3.8'

services:
  app:
    image: myapp
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  api:
    image: myapi
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://192.168.0.42:123"

  worker:
    image: worker
    logging:
      driver: none  # Disable logging
```

## Build Configuration

### Advanced Build Options

```yaml
version: '3.8'

services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      args:
        - NODE_ENV=production
        - BUILD_DATE=${BUILD_DATE}
      target: production
      cache_from:
        - myapp:cache
      labels:
        - "com.example.version=1.0"
      shm_size: '2gb'
      extra_hosts:
        - "host.docker.internal:host-gateway"
```

### Build with SSH

```yaml
services:
  app:
    build:
      context: .
      ssh:
        - default
```

## Multiple Compose Files Pattern

### File Structure

```
project/
├── docker-compose.yml          # Base configuration
├── docker-compose.override.yml # Development (auto-loaded)
├── docker-compose.prod.yml     # Production
├── docker-compose.test.yml     # Testing
└── .env                        # Environment variables
```

### Usage

```bash
# Development (loads yml + override automatically)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Testing
docker compose -f docker-compose.yml -f docker-compose.test.yml run --rm test

# Or use COMPOSE_FILE
export COMPOSE_FILE=docker-compose.yml:docker-compose.prod.yml
docker compose up -d
```

## Anchor and Aliases (YAML)

```yaml
version: '3.8'

x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"

x-env: &default-env
  NODE_ENV: production
  LOG_LEVEL: info

services:
  web:
    <<: *common
    image: nginx
    environment:
      <<: *default-env
      PORT: 80

  api:
    <<: *common
    image: myapi
    environment:
      <<: *default-env
      PORT: 3000
```

## Useful Commands

```bash
# Config validation
docker compose config

# View merged config
docker compose config --services
docker compose config --volumes

# Convert to different format
docker compose convert

# Top processes
docker compose top

# Port mapping
docker compose port web 80

# Events
docker compose events

# Images used
docker compose images

# Pause/Unpause
docker compose pause
docker compose unpause
```

## Quick Reference

| Feature | Syntax |
|---------|--------|
| Health check | `healthcheck:` |
| Profiles | `profiles: [dev]` |
| Resource limits | `deploy.resources.limits` |
| Restart policy | `restart: always` |
| Networks | `networks:` |
| Secrets | `secrets:` |
| Configs | `configs:` |
| Logging | `logging:` |
| Override file | `-f compose.override.yml` |
| YAML anchor | `&name` / `<<: *name` |

---

**Previous:** [14-docker-compose-basics.md](14-docker-compose-basics.md) | **Next:** [16-registry.md](16-registry.md)
