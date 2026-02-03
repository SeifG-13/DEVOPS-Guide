# Docker Engine

## What is Docker Engine?

Docker Engine is the core component of Docker - an open-source containerization technology for building and containerizing applications.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Docker Engine                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Docker CLI                            │   │
│   │                  (docker command)                        │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │ REST API                           │
│   ┌─────────────────────────▼───────────────────────────────┐   │
│   │                   Docker Daemon                          │   │
│   │                    (dockerd)                             │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │ gRPC                               │
│   ┌─────────────────────────▼───────────────────────────────┐   │
│   │                    containerd                            │   │
│   │              (container runtime)                         │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│   ┌─────────────────────────▼───────────────────────────────┐   │
│   │                       runc                               │   │
│   │              (OCI runtime spec)                          │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│   ┌─────────────────────────▼───────────────────────────────┐   │
│   │                   Linux Kernel                           │   │
│   │           (namespaces, cgroups, etc.)                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Docker Engine Components

### 1. Docker CLI

The command-line interface for interacting with Docker.

```bash
# The CLI sends commands to the Docker daemon
docker run nginx
docker build -t myapp .
docker ps

# CLI communicates via REST API
# Default socket: /var/run/docker.sock
```

### 2. Docker Daemon (dockerd)

The background service that manages Docker objects.

```bash
# Start Docker daemon
sudo systemctl start docker

# Check daemon status
sudo systemctl status docker

# Daemon configuration file
cat /etc/docker/daemon.json
```

**Daemon Responsibilities:**
- Managing images
- Managing containers
- Managing networks
- Managing volumes
- Handling API requests

### 3. containerd

High-level container runtime that manages container lifecycle.

```
┌─────────────────────────────────────────────────────────────────┐
│                       containerd                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│   │  Image    │  │ Container │  │  Storage  │  │  Network  │   │
│   │  Service  │  │  Service  │  │  Service  │  │  Service  │   │
│   └───────────┘  └───────────┘  └───────────┘  └───────────┘   │
│                                                                  │
│   Features:                                                      │
│   • Pull/push images                                            │
│   • Manage container lifecycle                                  │
│   • Manage storage (snapshots)                                  │
│   • Manage networking                                           │
│   • Plugin architecture                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Check containerd status
sudo systemctl status containerd

# containerd CLI tool
sudo ctr images list
sudo ctr containers list
```

### 4. runc

Low-level container runtime implementing OCI specification.

```
┌─────────────────────────────────────────────────────────────────┐
│                          runc                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   OCI (Open Container Initiative) Runtime Spec                  │
│                                                                  │
│   • Creates and runs containers                                 │
│   • Spawns container processes                                  │
│   • Sets up namespaces                                          │
│   • Configures cgroups                                          │
│   • Manages container lifecycle                                 │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  config.json                             │   │
│   │   {                                                      │   │
│   │     "ociVersion": "1.0.0",                              │   │
│   │     "process": { ... },                                 │   │
│   │     "root": { "path": "rootfs" },                       │   │
│   │     "linux": { "namespaces": [...], "cgroups": {...} }  │   │
│   │   }                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Linux Kernel Features

Docker relies on Linux kernel features for containerization.

### Namespaces

Provide isolation for containers.

| Namespace | Isolates |
|-----------|----------|
| PID | Process IDs |
| NET | Network interfaces |
| MNT | Mount points |
| UTS | Hostname and domain |
| IPC | Inter-process communication |
| USER | User and group IDs |
| CGROUP | Cgroup root directory |

```bash
# View namespaces of a container
docker inspect --format '{{.State.Pid}}' container_id
ls -la /proc/<PID>/ns/

# Example output
lrwxrwxrwx 1 root root 0 Jan 1 00:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jan 1 00:00 ipc -> 'ipc:[4026532389]'
lrwxrwxrwx 1 root root 0 Jan 1 00:00 mnt -> 'mnt:[4026532387]'
lrwxrwxrwx 1 root root 0 Jan 1 00:00 net -> 'net:[4026532392]'
lrwxrwxrwx 1 root root 0 Jan 1 00:00 pid -> 'pid:[4026532390]'
lrwxrwxrwx 1 root root 0 Jan 1 00:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jan 1 00:00 uts -> 'uts:[4026532388]'
```

### Control Groups (cgroups)

Limit and monitor resource usage.

```bash
# View cgroup limits
docker inspect --format '{{.HostConfig.Memory}}' container_id
docker inspect --format '{{.HostConfig.CpuShares}}' container_id

# Cgroup resources controlled
- CPU
- Memory
- Disk I/O
- Network
- Device access
```

### Union File System

Layers multiple file systems into a single view.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Union File System                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Container Layer (R/W)                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Container changes (writable)                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                      │
│   Image Layers (R/O)      ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Layer 4: Application code                              │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  Layer 3: Dependencies                                  │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  Layer 2: Runtime                                       │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  Layer 1: Base OS                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Docker Daemon Configuration

### Configuration File

```bash
# Location
/etc/docker/daemon.json

# Example configuration
{
  "data-root": "/var/lib/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {"base": "172.17.0.0/16", "size": 24}
  ],
  "dns": ["8.8.8.8", "8.8.4.4"],
  "registry-mirrors": ["https://mirror.example.com"],
  "insecure-registries": ["registry.local:5000"],
  "live-restore": true,
  "userland-proxy": false,
  "experimental": false,
  "debug": false
}
```

### Common Configuration Options

| Option | Description |
|--------|-------------|
| `data-root` | Docker data directory |
| `storage-driver` | Storage driver to use |
| `log-driver` | Default logging driver |
| `dns` | DNS servers for containers |
| `registry-mirrors` | Registry mirror URLs |
| `insecure-registries` | Registries without TLS |
| `live-restore` | Keep containers running during daemon downtime |
| `debug` | Enable debug mode |

### Apply Configuration Changes

```bash
# Edit configuration
sudo vim /etc/docker/daemon.json

# Reload daemon
sudo systemctl reload docker

# Or restart daemon
sudo systemctl restart docker

# Verify configuration
docker info
```

## Docker API

### REST API

```bash
# API via curl (using socket)
curl --unix-socket /var/run/docker.sock http://localhost/version

# List containers
curl --unix-socket /var/run/docker.sock http://localhost/containers/json

# Create container
curl --unix-socket /var/run/docker.sock \
  -H "Content-Type: application/json" \
  -d '{"Image": "nginx"}' \
  http://localhost/containers/create

# Enable TCP API (daemon.json)
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"]
}
```

### API Endpoints

| Endpoint | Description |
|----------|-------------|
| `/containers` | Container operations |
| `/images` | Image operations |
| `/networks` | Network operations |
| `/volumes` | Volume operations |
| `/info` | System information |
| `/version` | Version information |

## Docker Context

Manage multiple Docker environments.

```bash
# List contexts
docker context ls

# Create context for remote Docker
docker context create remote \
  --docker "host=tcp://remote-host:2375"

# Use context
docker context use remote

# Run command with specific context
docker --context remote ps
```

## Checking Docker Engine

```bash
# Docker version
docker version

# System information
docker info

# Docker root directory
docker info | grep "Docker Root Dir"

# Storage driver
docker info | grep "Storage Driver"

# Check daemon logs
sudo journalctl -u docker.service

# Real-time logs
sudo journalctl -u docker.service -f
```

## Engine Modes

### Standalone Mode

```
┌─────────────────────────────────────────┐
│           Single Host                   │
├─────────────────────────────────────────┤
│                                         │
│   ┌─────────────────────────────────┐   │
│   │        Docker Engine            │   │
│   │   ┌─────┐ ┌─────┐ ┌─────┐     │   │
│   │   │ C1  │ │ C2  │ │ C3  │     │   │
│   │   └─────┘ └─────┘ └─────┘     │   │
│   └─────────────────────────────────┘   │
│                                         │
└─────────────────────────────────────────┘
```

### Swarm Mode

```
┌─────────────────────────────────────────────────────────────────┐
│                      Docker Swarm Cluster                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Manager Nodes                    Worker Nodes                  │
│   ┌───────────────┐               ┌───────────────┐             │
│   │   Manager 1   │               │   Worker 1    │             │
│   │   (Leader)    │               │ ┌───┐ ┌───┐  │             │
│   └───────┬───────┘               │ │C1 │ │C2 │  │             │
│           │                        │ └───┘ └───┘  │             │
│   ┌───────┴───────┐               └───────────────┘             │
│   │   Manager 2   │               ┌───────────────┐             │
│   │   (Reachable) │               │   Worker 2    │             │
│   └───────────────┘               │ ┌───┐ ┌───┐  │             │
│                                    │ │C3 │ │C4 │  │             │
│                                    │ └───┘ └───┘  │             │
│                                    └───────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Initialize swarm
docker swarm init

# Join as worker
docker swarm join --token <token> <manager-ip>:2377

# List nodes
docker node ls
```

## Quick Reference

| Component | Role |
|-----------|------|
| Docker CLI | User interface |
| dockerd | Docker daemon |
| containerd | Container runtime |
| runc | OCI runtime |
| Namespaces | Process isolation |
| cgroups | Resource limits |

---

**Previous:** [01-introduction.md](01-introduction.md) | **Next:** [03-installation.md](03-installation.md)
