# Containers Deep Dive

## Container Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    Container Lifecycle                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│         docker create              docker start                  │
│              │                          │                        │
│              ▼                          ▼                        │
│        ┌──────────┐              ┌───────────┐                  │
│        │ Created  │─────────────►│  Running  │◄─────┐           │
│        └──────────┘              └─────┬─────┘      │           │
│                                        │            │           │
│              docker run ───────────────┘            │           │
│                                                     │           │
│                    ┌──────────────┬─────────────────┘           │
│                    │              │                              │
│           docker stop      docker pause                         │
│                    │              │                              │
│                    ▼              ▼                              │
│              ┌──────────┐  ┌──────────┐                         │
│              │  Exited  │  │  Paused  │                         │
│              └────┬─────┘  └────┬─────┘                         │
│                   │             │                                │
│           docker rm      docker unpause                         │
│                   │             │                                │
│                   ▼             │                                │
│              ┌──────────┐       │                                │
│              │ Removed  │◄──────┘ (via stop then rm)            │
│              └──────────┘                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Container States

| State | Description |
|-------|-------------|
| `created` | Container created but not started |
| `running` | Container is running |
| `paused` | Container processes are paused |
| `restarting` | Container is restarting |
| `exited` | Container has stopped |
| `dead` | Container failed to stop properly |

```bash
# Check container state
docker inspect --format '{{.State.Status}}' container_name

# List containers by state
docker ps -a -f status=running
docker ps -a -f status=exited
docker ps -a -f status=paused
```

## Attach vs Exec

### docker attach

Connects to the container's main process (PID 1).

```bash
# Attach to running container
docker attach container_name

# Detach without stopping: Ctrl+P, Ctrl+Q

# Attach without STDIN
docker attach --no-stdin container_name

# Attach with signal proxy disabled
docker attach --sig-proxy=false container_name
```

### docker exec

Runs a new command in a running container.

```bash
# Run command
docker exec container_name ls -la

# Interactive shell
docker exec -it container_name /bin/bash
docker exec -it container_name /bin/sh

# Run as specific user
docker exec -u root container_name whoami

# Set environment variable
docker exec -e MY_VAR=value container_name env

# Set working directory
docker exec -w /app container_name pwd

# Detached mode
docker exec -d container_name touch /tmp/file
```

### Attach vs Exec Comparison

| Feature | attach | exec |
|---------|--------|------|
| Connects to | Main process (PID 1) | New process |
| Sees existing output | Yes | No |
| Multiple connections | Limited | Yes |
| Exit behavior | May stop container | Process exits only |
| Use case | View main process | Run additional commands |

## Resource Limits

### Memory Limits

```bash
# Limit memory
docker run -m 512m nginx
docker run --memory=512m nginx

# Memory + Swap limit
docker run -m 512m --memory-swap=1g nginx

# Disable swap
docker run -m 512m --memory-swap=512m nginx

# Soft limit (memory reservation)
docker run -m 1g --memory-reservation=512m nginx

# OOM killer settings
docker run --oom-kill-disable nginx
docker run --oom-score-adj=-500 nginx
```

### CPU Limits

```bash
# Limit CPU shares (relative weight, default 1024)
docker run --cpu-shares=512 nginx

# Limit CPU cores
docker run --cpus=1.5 nginx

# Limit to specific CPUs
docker run --cpuset-cpus="0,1" nginx
docker run --cpuset-cpus="0-3" nginx

# CPU period and quota
docker run --cpu-period=100000 --cpu-quota=50000 nginx
# 50% of one CPU
```

### Combined Limits

```bash
docker run -d \
    --name myapp \
    --memory=512m \
    --memory-swap=1g \
    --cpus=1.0 \
    --cpu-shares=512 \
    nginx
```

### View Resource Usage

```bash
# Real-time stats
docker stats

# Stats for specific containers
docker stats container1 container2

# One-time snapshot
docker stats --no-stream

# Formatted output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

## Restart Policies

### Available Policies

| Policy | Description |
|--------|-------------|
| `no` | Never restart (default) |
| `always` | Always restart |
| `unless-stopped` | Restart unless manually stopped |
| `on-failure[:max]` | Restart on failure, optional max retries |

```bash
# No restart (default)
docker run --restart=no nginx

# Always restart
docker run --restart=always nginx

# Restart unless stopped
docker run --restart=unless-stopped nginx

# Restart on failure (max 3 times)
docker run --restart=on-failure:3 nginx

# Update restart policy
docker update --restart=always container_name
```

### Restart Policy Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    Restart Policy Behavior                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Scenario              no    always  unless-stopped  on-failure│
│   ─────────────────────────────────────────────────────────────│
│   Container exits       No     Yes        Yes           If err  │
│   Docker daemon restart No     Yes        Yes*          No      │
│   Manual docker stop    No     No         No            No      │
│   Exit code 0           No     Yes        Yes           No      │
│   Exit code non-zero    No     Yes        Yes           Yes     │
│                                                                  │
│   * unless-stopped: Not if container was stopped before daemon  │
│     restart                                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Container Isolation

### PID Namespace

```bash
# Default: isolated PID namespace
docker run nginx
# PID 1 inside container is nginx

# Share host PID namespace
docker run --pid=host nginx
# Container sees host processes

# Share with another container
docker run --pid=container:other_container nginx
```

### Network Namespace

```bash
# Default: bridge network
docker run nginx

# Host network
docker run --network=host nginx

# No network
docker run --network=none nginx

# Share with another container
docker run --network=container:other_container nginx
```

### IPC Namespace

```bash
# Default: isolated
docker run nginx

# Share host IPC
docker run --ipc=host nginx

# Share with another container
docker run --ipc=container:other_container nginx
```

### User Namespace

```bash
# Default: root in container
docker run nginx

# Run as specific user
docker run --user 1000:1000 nginx
docker run --user nobody nginx

# User namespace remapping (daemon config)
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

## Container Security

### Read-Only Root Filesystem

```bash
# Read-only container
docker run --read-only nginx

# Read-only with writable tmpfs
docker run --read-only --tmpfs /tmp nginx

# Read-only with volume for data
docker run --read-only -v data:/var/data nginx
```

### Capabilities

```bash
# Drop all capabilities
docker run --cap-drop=ALL nginx

# Add specific capability
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Common capabilities
# NET_BIND_SERVICE - Bind to ports < 1024
# SYS_ADMIN - Various admin operations
# NET_ADMIN - Network administration
# SYS_TIME - Set system clock
```

### Security Options

```bash
# No new privileges
docker run --security-opt=no-new-privileges nginx

# AppArmor profile
docker run --security-opt apparmor=docker-default nginx

# Seccomp profile
docker run --security-opt seccomp=profile.json nginx

# Disable seccomp
docker run --security-opt seccomp=unconfined nginx
```

## Container Runtime Options

### Privileged Mode

```bash
# Full host access (dangerous!)
docker run --privileged nginx

# Access specific devices instead
docker run --device=/dev/sda:/dev/sda nginx
```

### Hostname and DNS

```bash
# Set hostname
docker run --hostname=myhost nginx

# Set DNS servers
docker run --dns=8.8.8.8 --dns=8.8.4.4 nginx

# Set DNS search domains
docker run --dns-search=example.com nginx

# Add host entry
docker run --add-host=myhost:192.168.1.100 nginx
```

### Working Directory and User

```bash
# Set working directory
docker run -w /app nginx

# Set user
docker run -u 1000 nginx
docker run -u 1000:1000 nginx
docker run -u www-data nginx
```

## Container Updates

### Update Running Container

```bash
# Update memory limit
docker update --memory=1g container_name

# Update CPU limit
docker update --cpus=2 container_name

# Update restart policy
docker update --restart=always container_name

# Update multiple containers
docker update --memory=512m container1 container2

# Cannot update:
# - Port mappings
# - Volume mounts
# - Network settings
```

## Inspecting Containers

```bash
# Full inspection
docker inspect container_name

# Get specific fields
docker inspect --format '{{.State.Status}}' container_name
docker inspect --format '{{.NetworkSettings.IPAddress}}' container_name
docker inspect --format '{{.HostConfig.Memory}}' container_name

# Get multiple fields
docker inspect --format 'IP: {{.NetworkSettings.IPAddress}} State: {{.State.Status}}' container_name

# JSON output with jq
docker inspect container_name | jq '.[0].State'
docker inspect container_name | jq '.[0].NetworkSettings.Ports'
```

## Container Events

```bash
# Watch container events
docker events

# Filter by container
docker events --filter container=container_name

# Filter by event type
docker events --filter event=start
docker events --filter event=stop
docker events --filter event=die

# Filter by time
docker events --since "2024-01-01"
docker events --until "2024-01-02"

# Format output
docker events --format '{{.Time}} {{.Actor.Attributes.name}} {{.Action}}'
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker create` | Create container |
| `docker start` | Start container |
| `docker stop` | Stop container |
| `docker restart` | Restart container |
| `docker pause` | Pause container |
| `docker unpause` | Unpause container |
| `docker attach` | Attach to main process |
| `docker exec` | Run new command |
| `docker update` | Update container config |
| `docker stats` | Resource usage |
| `docker events` | Container events |

---

**Previous:** [09-building-images.md](09-building-images.md) | **Next:** [11-networking.md](11-networking.md)
