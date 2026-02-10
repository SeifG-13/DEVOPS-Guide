# Docker Cheat Sheet for DevOps Engineers

## Quick Reference Commands

### Container Lifecycle
```bash
# Run containers
docker run nginx                    # Run container
docker run -d nginx                 # Detached mode
docker run -it ubuntu bash          # Interactive terminal
docker run --name myapp nginx       # Named container
docker run -p 8080:80 nginx         # Port mapping
docker run -v /host:/container nginx # Volume mount
docker run --rm nginx               # Auto-remove on exit
docker run -e VAR=value nginx       # Environment variable
docker run --network mynet nginx    # Custom network

# Container management
docker ps                           # List running containers
docker ps -a                        # List all containers
docker stop container               # Stop container
docker start container              # Start container
docker restart container            # Restart container
docker rm container                 # Remove container
docker rm -f container              # Force remove
docker kill container               # Kill container
```

### Container Inspection
```bash
docker logs container               # View logs
docker logs -f container            # Follow logs
docker logs --tail 100 container    # Last 100 lines
docker exec -it container bash      # Execute command
docker inspect container            # Full details (JSON)
docker top container                # Running processes
docker stats                        # Resource usage
docker diff container               # Filesystem changes
```

### Images
```bash
docker images                       # List images
docker pull nginx:latest            # Pull image
docker push myrepo/myimage:tag      # Push image
docker build -t myimage:v1 .        # Build image
docker tag image:v1 image:latest    # Tag image
docker rmi image                    # Remove image
docker image prune                  # Remove unused images
docker history image                # Image layers
docker save -o img.tar image        # Export image
docker load -i img.tar              # Import image
```

### Networking
```bash
docker network ls                   # List networks
docker network create mynet         # Create network
docker network inspect mynet        # Network details
docker network connect mynet cont   # Connect container
docker network disconnect mynet cont # Disconnect container
docker run --network=host nginx     # Host network mode

# Network types
# - bridge: Default, isolated network
# - host: Share host's network
# - none: No networking
# - overlay: Multi-host (Swarm)
```

### Volumes
```bash
docker volume ls                    # List volumes
docker volume create myvol          # Create volume
docker volume inspect myvol         # Volume details
docker volume rm myvol              # Remove volume
docker volume prune                 # Remove unused

# Mount types
docker run -v myvol:/data nginx     # Named volume
docker run -v /host:/data nginx     # Bind mount
docker run --tmpfs /tmp nginx       # tmpfs mount
```

### Cleanup
```bash
docker system prune                 # Remove unused data
docker system prune -a              # Remove all unused
docker container prune              # Remove stopped containers
docker image prune -a               # Remove all unused images
docker volume prune                 # Remove unused volumes
docker system df                    # Disk usage
```

---

## Dockerfile Reference

### Basic Dockerfile
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

USER node

CMD ["node", "server.js"]
```

### Multi-Stage Build
```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Common Instructions
```dockerfile
FROM image:tag              # Base image
WORKDIR /app                # Set working directory
COPY src dest               # Copy files
ADD src dest                # Copy + extract archives
RUN command                 # Execute command
ENV KEY=value               # Environment variable
ARG VAR=default             # Build-time variable
EXPOSE 8080                 # Document port
USER username               # Switch user
VOLUME ["/data"]            # Define volume
ENTRYPOINT ["executable"]   # Main command
CMD ["param1", "param2"]    # Default arguments
HEALTHCHECK CMD curl -f http://localhost/ || exit 1
LABEL key="value"           # Metadata
```

---

## Docker Compose

### Basic docker-compose.yml
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    depends_on:
      - db
    networks:
      - app-network
    volumes:
      - ./data:/app/data
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  db-data:

networks:
  app-network:
    driver: bridge
```

### Compose Commands
```bash
docker-compose up                   # Start services
docker-compose up -d                # Detached mode
docker-compose up --build           # Rebuild images
docker-compose down                 # Stop and remove
docker-compose down -v              # Also remove volumes
docker-compose ps                   # List services
docker-compose logs                 # View logs
docker-compose logs -f service      # Follow service logs
docker-compose exec service bash    # Execute in service
docker-compose build                # Build images
docker-compose pull                 # Pull images
docker-compose restart              # Restart services
docker-compose config               # Validate compose file
```

---

## Interview Q&A

### Q1: What is the difference between CMD and ENTRYPOINT?
**A:**
- **ENTRYPOINT**: Main executable, not easily overridden
- **CMD**: Default arguments to ENTRYPOINT, easily overridden

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myimage          -> python app.py
# docker run myimage test.py  -> python test.py
```

### Q2: What is the difference between COPY and ADD?
**A:**
- **COPY**: Simple file copy (preferred)
- **ADD**: Copy + auto-extract tarballs + fetch URLs
Use COPY unless you need ADD's special features.

### Q3: How do you reduce Docker image size?
**A:**
1. Use multi-stage builds
2. Use Alpine-based images
3. Combine RUN commands (reduce layers)
4. Remove unnecessary files in same layer
5. Use .dockerignore
6. Don't install dev dependencies

### Q4: What is the difference between volumes and bind mounts?
**A:**
- **Volumes**: Managed by Docker, persist data, best for production
- **Bind mounts**: Direct host filesystem access, for development

### Q5: Explain Docker networking modes
**A:**
- **bridge**: Default, containers on same bridge can communicate
- **host**: Container uses host's network stack directly
- **none**: No networking
- **overlay**: Multi-host networking (Swarm/K8s)

### Q6: How do you handle secrets in Docker?
**A:**
- Docker Secrets (Swarm mode)
- Environment variables (not recommended for sensitive data)
- Mounted secret files
- External secret managers (Vault, AWS Secrets Manager)
```bash
docker secret create my_secret secret.txt
docker service create --secret my_secret myimage
```

### Q7: What is a Docker layer?
**A:** Each instruction in Dockerfile creates a layer. Layers are cached and reused. Order instructions from least to most frequently changing for optimal caching.

### Q8: How do you debug a container that won't start?
**A:**
```bash
docker logs container               # Check logs
docker inspect container            # Check config
docker run -it image sh             # Override entrypoint
docker run --entrypoint sh image    # Override entrypoint
docker events                       # Docker daemon events
```

### Q9: What is a Docker registry?
**A:** Storage and distribution system for Docker images.
- **Docker Hub**: Public default registry
- **Private**: Harbor, AWS ECR, Azure ACR, GCR
```bash
docker login registry.example.com
docker push registry.example.com/image:tag
```

### Q10: How do you implement health checks?
**A:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost/health || exit 1
```
```bash
docker inspect --format='{{.State.Health.Status}}' container
```

---

## Best Practices

### Dockerfile Best Practices
```dockerfile
# 1. Use specific base image tags
FROM node:18.17-alpine3.18

# 2. Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 3. Use multi-stage builds
FROM node:18 AS builder
# ... build steps
FROM node:18-alpine
COPY --from=builder /app/dist ./

# 4. Combine RUN commands
RUN apt-get update && \
    apt-get install -y package && \
    rm -rf /var/lib/apt/lists/*

# 5. Copy package files first (leverage cache)
COPY package*.json ./
RUN npm ci
COPY . .

# 6. Use .dockerignore
# .dockerignore: node_modules, .git, *.log
```

### Security Best Practices
1. **Don't run as root** - Use USER instruction
2. **Scan images** - Trivy, Snyk, Clair
3. **Use trusted base images** - Official images
4. **Keep images updated** - Security patches
5. **Minimize attack surface** - Only install needed packages
6. **Use secrets properly** - Not in images or environment variables
7. **Read-only filesystem** - `--read-only` flag
8. **Limit resources** - `--memory`, `--cpus`

---

## Useful Patterns

### Init Container Pattern
```yaml
services:
  init:
    image: busybox
    command: ["sh", "-c", "until nc -z db 5432; do sleep 1; done"]

  app:
    depends_on:
      init:
        condition: service_completed_successfully
```

### Sidecar Pattern
```yaml
services:
  app:
    image: myapp
    volumes:
      - logs:/var/log/app

  log-shipper:
    image: fluentd
    volumes:
      - logs:/var/log/app:ro
```

### Environment-Specific Compose
```bash
# Base + override
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```
