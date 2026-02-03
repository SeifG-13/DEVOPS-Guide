# Docker Basic Commands

## Container Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    Container Lifecycle                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   docker create                docker start                      │
│        │                            │                            │
│        ▼                            ▼                            │
│   ┌─────────┐     docker run    ┌─────────┐                     │
│   │ Created │ ─────────────────►│ Running │◄──┐                 │
│   └─────────┘                   └────┬────┘   │                 │
│                                      │        │                  │
│                    docker pause      │        │ docker unpause   │
│                         │            ▼        │                  │
│                         │      ┌─────────┐    │                  │
│                         └─────►│ Paused  │────┘                  │
│                                └─────────┘                       │
│                                      │                           │
│                    docker stop       │                           │
│                         │            ▼                           │
│                         │      ┌─────────┐                       │
│                         └─────►│ Stopped │                       │
│                                └────┬────┘                       │
│                                     │                            │
│                    docker rm        │                            │
│                         │           ▼                            │
│                         │      ┌─────────┐                       │
│                         └─────►│ Removed │                       │
│                                └─────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Running Containers

### docker run

```bash
# Basic run
docker run nginx

# Run in background (detached)
docker run -d nginx

# Run with name
docker run -d --name webserver nginx

# Run with port mapping
docker run -d -p 8080:80 nginx
# Host port 8080 → Container port 80

# Run with multiple ports
docker run -d -p 8080:80 -p 8443:443 nginx

# Run interactively
docker run -it ubuntu bash

# Run and remove after exit
docker run --rm ubuntu echo "Hello"

# Run with environment variable
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql

# Run with volume
docker run -d -v /host/path:/container/path nginx

# Run with working directory
docker run -w /app node npm start

# Run with network
docker run -d --network mynetwork nginx

# Run with restart policy
docker run -d --restart always nginx
```

### Common run Options

| Option | Description | Example |
|--------|-------------|---------|
| `-d` | Detached mode (background) | `docker run -d nginx` |
| `-it` | Interactive terminal | `docker run -it ubuntu bash` |
| `-p` | Port mapping | `-p 8080:80` |
| `-v` | Volume mount | `-v /data:/app/data` |
| `-e` | Environment variable | `-e KEY=value` |
| `--name` | Container name | `--name myapp` |
| `--rm` | Remove on exit | `docker run --rm ubuntu` |
| `--network` | Connect to network | `--network bridge` |
| `-w` | Working directory | `-w /app` |
| `--restart` | Restart policy | `--restart always` |

## Listing Containers

### docker ps

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List container IDs only
docker ps -q

# List all container IDs
docker ps -aq

# List with size
docker ps -s

# Format output
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"

# Filter containers
docker ps -f "status=exited"
docker ps -f "name=web"
docker ps -f "ancestor=nginx"

# Show latest created container
docker ps -l

# Show last n containers
docker ps -n 5
```

### Output Columns

| Column | Description |
|--------|-------------|
| CONTAINER ID | Unique identifier |
| IMAGE | Image used |
| COMMAND | Command running |
| CREATED | When created |
| STATUS | Current status |
| PORTS | Port mappings |
| NAMES | Container name |

## Managing Containers

### Start, Stop, Restart

```bash
# Stop a container
docker stop container_name
docker stop container_id

# Stop with timeout (default 10s)
docker stop -t 30 container_name

# Start a stopped container
docker start container_name

# Restart a container
docker restart container_name

# Pause a container (freeze processes)
docker pause container_name

# Unpause a container
docker unpause container_name

# Kill a container (force stop)
docker kill container_name
```

### Remove Containers

```bash
# Remove a stopped container
docker rm container_name

# Force remove running container
docker rm -f container_name

# Remove multiple containers
docker rm container1 container2

# Remove all stopped containers
docker container prune

# Remove all containers (force)
docker rm -f $(docker ps -aq)
```

## Container Information

### docker inspect

```bash
# Full container details
docker inspect container_name

# Get specific field
docker inspect --format '{{.State.Status}}' container_name

# Get IP address
docker inspect --format '{{.NetworkSettings.IPAddress}}' container_name

# Get port bindings
docker inspect --format '{{.NetworkSettings.Ports}}' container_name

# Get environment variables
docker inspect --format '{{.Config.Env}}' container_name

# Get mounts
docker inspect --format '{{.Mounts}}' container_name

# Pretty print JSON
docker inspect container_name | jq '.'
```

### docker logs

```bash
# View container logs
docker logs container_name

# Follow logs (real-time)
docker logs -f container_name

# Show timestamps
docker logs -t container_name

# Tail last n lines
docker logs --tail 100 container_name

# Logs since timestamp
docker logs --since 2024-01-01T00:00:00 container_name

# Logs until timestamp
docker logs --until 2024-01-01T12:00:00 container_name

# Combined options
docker logs -f --tail 50 -t container_name
```

### docker stats

```bash
# Real-time resource usage
docker stats

# Stats for specific containers
docker stats container1 container2

# One-time stats (no stream)
docker stats --no-stream

# Format output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### docker top

```bash
# View processes in container
docker top container_name

# With specific ps options
docker top container_name aux
```

## Executing Commands

### docker exec

```bash
# Run command in running container
docker exec container_name ls -la

# Interactive shell
docker exec -it container_name bash
docker exec -it container_name sh

# Run as specific user
docker exec -u root container_name whoami

# Set environment variable
docker exec -e VAR=value container_name env

# Set working directory
docker exec -w /app container_name pwd

# Run detached command
docker exec -d container_name touch /tmp/newfile
```

### docker attach

```bash
# Attach to running container
docker attach container_name

# Detach without stopping: Ctrl+P, Ctrl+Q

# Attach with signal proxy disabled
docker attach --sig-proxy=false container_name
```

## Copying Files

### docker cp

```bash
# Copy from host to container
docker cp file.txt container_name:/path/in/container/

# Copy from container to host
docker cp container_name:/path/in/container/file.txt ./

# Copy entire directory
docker cp ./mydir container_name:/app/

# Copy from container directory
docker cp container_name:/var/log/ ./logs/
```

## Container Differences

### docker diff

```bash
# Show filesystem changes
docker diff container_name

# Output:
# A = Added
# C = Changed
# D = Deleted
```

## Port Mapping

```bash
# View port mappings
docker port container_name

# Map specific port
docker run -p 8080:80 nginx
# hostPort:containerPort

# Map to specific interface
docker run -p 127.0.0.1:8080:80 nginx

# Map random port
docker run -p 80 nginx
docker port container_name 80

# Map all exposed ports randomly
docker run -P nginx
```

## Practical Examples

### Run Web Server

```bash
# Run Nginx
docker run -d --name web -p 80:80 nginx

# Verify
curl http://localhost

# Check logs
docker logs web

# Stop and remove
docker stop web && docker rm web
```

### Run Database

```bash
# Run MySQL
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=mydb \
  -p 3306:3306 \
  -v mysql_data:/var/lib/mysql \
  mysql:8

# Connect to MySQL
docker exec -it mysql mysql -u root -psecret

# Run PostgreSQL
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15
```

### Run Application Stack

```bash
# Create network
docker network create myapp

# Run database
docker run -d \
  --name db \
  --network myapp \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Run application
docker run -d \
  --name app \
  --network myapp \
  -e DATABASE_URL=postgres://postgres:secret@db:5432/postgres \
  -p 3000:3000 \
  myapp:latest
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker run` | Create and start container |
| `docker ps` | List containers |
| `docker stop` | Stop container |
| `docker start` | Start container |
| `docker restart` | Restart container |
| `docker rm` | Remove container |
| `docker logs` | View logs |
| `docker exec` | Execute command |
| `docker inspect` | Detailed info |
| `docker cp` | Copy files |
| `docker stats` | Resource usage |

---

**Previous:** [03-installation.md](03-installation.md) | **Next:** [05-images.md](05-images.md)
