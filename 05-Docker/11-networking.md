# Docker Networking

## Network Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Networking                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Host Machine                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   ┌─────────┐    ┌─────────┐    ┌─────────┐            │   │
│   │   │Container│    │Container│    │Container│            │   │
│   │   │   A     │    │   B     │    │   C     │            │   │
│   │   └────┬────┘    └────┬────┘    └────┬────┘            │   │
│   │        │              │              │                  │   │
│   │   ┌────┴──────────────┴──────────────┴────┐            │   │
│   │   │           Docker Network              │            │   │
│   │   │         (bridge: docker0)             │            │   │
│   │   └─────────────────┬─────────────────────┘            │   │
│   │                     │                                   │   │
│   └─────────────────────┼───────────────────────────────────┘   │
│                         │                                       │
│                    ┌────┴────┐                                  │
│                    │  eth0   │ ◄── Physical Network            │
│                    └─────────┘                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Network Drivers

| Driver | Description | Use Case |
|--------|-------------|----------|
| `bridge` | Default network driver | Container-to-container on same host |
| `host` | No network isolation | Performance-critical apps |
| `none` | No networking | Security isolation |
| `overlay` | Multi-host networking | Docker Swarm |
| `macvlan` | Assign MAC address | Legacy applications |
| `ipvlan` | Layer 2/3 networking | Advanced networking |

## Bridge Network (Default)

### Default Bridge (docker0)

```bash
# Containers on default bridge
docker run -d --name web1 nginx
docker run -d --name web2 nginx

# Containers can communicate via IP
docker exec web1 ping <web2-ip>

# But NOT via container name on default bridge
docker exec web1 ping web2  # Fails
```

### User-Defined Bridge

```bash
# Create network
docker network create mynetwork

# Run containers on network
docker run -d --name web1 --network mynetwork nginx
docker run -d --name web2 --network mynetwork nginx

# Containers can communicate via name
docker exec web1 ping web2  # Works!
```

### Bridge Network Benefits

```
┌─────────────────────────────────────────────────────────────────┐
│           User-Defined Bridge vs Default Bridge                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Default Bridge              User-Defined Bridge               │
│   ┌──────────────────┐       ┌──────────────────┐              │
│   │ ✗ No DNS         │       │ ✓ Automatic DNS  │              │
│   │ ✗ All connected  │       │ ✓ Isolation      │              │
│   │ ✗ Link required  │       │ ✓ Hot connect    │              │
│   │ ✗ Env vars only  │       │ ✓ Service discovery │           │
│   └──────────────────┘       └──────────────────┘              │
│                                                                  │
│   Use user-defined bridges for production!                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Host Network

Container shares host's network namespace.

```bash
# Use host network
docker run -d --network host nginx

# Container uses host's IP directly
# No port mapping needed (-p flag ignored)
# Best performance, least isolation
```

### When to Use Host Network

- Performance-critical applications
- Container needs to handle lots of ports
- Container needs to see all host traffic

## None Network

Complete network isolation.

```bash
# No network at all
docker run -d --network none nginx

# Container only has loopback interface
docker exec container_name ip addr
# Only shows lo interface
```

## Overlay Network

For multi-host container communication (Docker Swarm).

```bash
# Initialize swarm
docker swarm init

# Create overlay network
docker network create -d overlay myoverlay

# Create service on overlay
docker service create --network myoverlay --name web nginx

# Overlay features:
# - Multi-host communication
# - Encrypted traffic (optional)
# - Service discovery
# - Load balancing
```

## Network Commands

### List Networks

```bash
# List all networks
docker network ls

# Detailed output
docker network ls --no-trunc

# Filter networks
docker network ls -f driver=bridge
docker network ls -f name=my
```

### Create Networks

```bash
# Create bridge network
docker network create mynetwork

# Create with specific driver
docker network create -d bridge mybridge

# Create with subnet
docker network create --subnet=172.20.0.0/16 mynetwork

# Create with gateway
docker network create \
    --subnet=172.20.0.0/16 \
    --gateway=172.20.0.1 \
    mynetwork

# Create with IP range
docker network create \
    --subnet=172.20.0.0/16 \
    --ip-range=172.20.10.0/24 \
    --gateway=172.20.0.1 \
    mynetwork

# Create with options
docker network create \
    --driver bridge \
    --opt com.docker.network.bridge.name=my_bridge \
    mynetwork
```

### Inspect Networks

```bash
# Inspect network
docker network inspect mynetwork

# Get specific info
docker network inspect --format '{{.IPAM.Config}}' mynetwork

# List containers on network
docker network inspect --format '{{json .Containers}}' mynetwork | jq
```

### Connect/Disconnect Containers

```bash
# Connect running container to network
docker network connect mynetwork container_name

# Connect with specific IP
docker network connect --ip 172.20.10.100 mynetwork container_name

# Connect with alias
docker network connect --alias db mynetwork container_name

# Disconnect from network
docker network disconnect mynetwork container_name
```

### Remove Networks

```bash
# Remove network
docker network rm mynetwork

# Remove unused networks
docker network prune

# Force remove
docker network rm -f mynetwork
```

## Port Mapping

### Publish Ports

```bash
# Map host port to container port
docker run -p 8080:80 nginx
# hostPort:containerPort

# Map to specific interface
docker run -p 127.0.0.1:8080:80 nginx

# Map random host port
docker run -p 80 nginx
docker port container_name 80  # See assigned port

# Map all exposed ports
docker run -P nginx

# Multiple port mappings
docker run -p 80:80 -p 443:443 nginx

# UDP port
docker run -p 53:53/udp dns-server

# TCP and UDP
docker run -p 53:53/tcp -p 53:53/udp dns-server
```

### View Port Mappings

```bash
# List port mappings
docker port container_name

# Specific port
docker port container_name 80
```

## Container DNS

### Built-in DNS Server

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker DNS Resolution                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Container A                   Container B                      │
│   ┌─────────────┐              ┌─────────────┐                  │
│   │   web       │              │   api       │                  │
│   │             │──── ping api ──►           │                  │
│   └──────┬──────┘              └──────┬──────┘                  │
│          │                            │                          │
│          └───────────┬────────────────┘                          │
│                      │                                           │
│              ┌───────▼───────┐                                   │
│              │  Docker DNS   │                                   │
│              │   (127.0.0.11)│                                   │
│              │               │                                   │
│              │ api → 172.18.0.3                                  │
│              │ web → 172.18.0.2                                  │
│              └───────────────┘                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### DNS Configuration

```bash
# Custom DNS servers
docker run --dns=8.8.8.8 --dns=8.8.4.4 nginx

# DNS search domains
docker run --dns-search=example.com nginx

# Add host entry
docker run --add-host=myhost:192.168.1.100 nginx

# Multiple hosts
docker run \
    --add-host=host1:192.168.1.1 \
    --add-host=host2:192.168.1.2 \
    nginx
```

### Network Aliases

```bash
# Add alias when connecting
docker network connect --alias db --alias database mynetwork container_name

# Run with alias
docker run -d --network mynetwork --network-alias api nginx

# Container accessible via multiple names:
# - container name
# - aliases
```

## Practical Examples

### Multi-Container Application

```bash
# Create network
docker network create webapp

# Run database
docker run -d \
    --name db \
    --network webapp \
    -e POSTGRES_PASSWORD=secret \
    postgres:15

# Run backend
docker run -d \
    --name api \
    --network webapp \
    -e DATABASE_URL=postgres://postgres:secret@db:5432/postgres \
    myapi:latest

# Run frontend
docker run -d \
    --name web \
    --network webapp \
    -p 80:80 \
    myweb:latest

# All containers can communicate via names
```

### Isolated Networks

```bash
# Frontend network
docker network create frontend

# Backend network
docker network create backend

# API connects to both
docker run -d --name api --network backend myapi
docker network connect frontend api

# Web only on frontend
docker run -d --name web --network frontend -p 80:80 nginx

# DB only on backend
docker run -d --name db --network backend postgres

# web → api (frontend) ✓
# api → db (backend) ✓
# web → db (no connection) ✗
```

## Network Troubleshooting

```bash
# Check container IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Check network connectivity
docker exec container1 ping container2

# Check DNS resolution
docker exec container_name nslookup other_container

# Check routes
docker exec container_name ip route

# Check network interfaces
docker exec container_name ip addr

# Check listening ports
docker exec container_name netstat -tlnp
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker network ls` | List networks |
| `docker network create` | Create network |
| `docker network rm` | Remove network |
| `docker network inspect` | Inspect network |
| `docker network connect` | Connect container |
| `docker network disconnect` | Disconnect container |
| `docker network prune` | Remove unused |
| `-p 8080:80` | Map port |
| `--network name` | Use network |
| `--dns` | Set DNS server |

---

**Previous:** [10-containers-deep-dive.md](10-containers-deep-dive.md) | **Next:** [12-storage.md](12-storage.md)
