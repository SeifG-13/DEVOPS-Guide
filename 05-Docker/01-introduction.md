# Introduction to Docker

## What is Docker?

Docker is a platform for developing, shipping, and running applications in containers - lightweight, portable, and self-sufficient units that include everything needed to run software.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Docker Overview                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Developer          Docker           Production                 │
│   ┌─────────┐       ┌─────────┐      ┌─────────┐               │
│   │  Code   │──────►│ Image   │─────►│Container│               │
│   │  +      │       │ Build   │      │  Run    │               │
│   │Dockerfile│       │         │      │         │               │
│   └─────────┘       └─────────┘      └─────────┘               │
│                                                                  │
│   "Build Once, Run Anywhere"                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Why Docker?

| Benefit | Description |
|---------|-------------|
| Consistency | Same environment from dev to production |
| Isolation | Applications run independently |
| Portability | Run anywhere Docker is installed |
| Efficiency | Share OS kernel, lightweight |
| Scalability | Easy to scale up/down |
| Speed | Start containers in seconds |

## Containers vs Virtual Machines

```
┌─────────────────────────────────────────────────────────────────┐
│         Virtual Machines              Containers                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────┐ ┌─────┐ ┌─────┐      ┌─────┐ ┌─────┐ ┌─────┐        │
│   │App A│ │App B│ │App C│      │App A│ │App B│ │App C│        │
│   ├─────┤ ├─────┤ ├─────┤      ├─────┤ ├─────┤ ├─────┤        │
│   │Bins │ │Bins │ │Bins │      │Bins │ │Bins │ │Bins │        │
│   │Libs │ │Libs │ │Libs │      │Libs │ │Libs │ │Libs │        │
│   ├─────┤ ├─────┤ ├─────┤      └──┬──┘ └──┬──┘ └──┬──┘        │
│   │Guest│ │Guest│ │Guest│         │       │       │            │
│   │ OS  │ │ OS  │ │ OS  │      ┌──┴───────┴───────┴──┐        │
│   └──┬──┘ └──┬──┘ └──┬──┘      │   Container Engine  │        │
│      │       │       │          │      (Docker)       │        │
│   ┌──┴───────┴───────┴──┐      └──────────┬──────────┘        │
│   │     Hypervisor      │                 │                    │
│   └──────────┬──────────┘      ┌──────────┴──────────┐        │
│   ┌──────────┴──────────┐      │      Host OS        │        │
│   │      Host OS        │      └──────────┬──────────┘        │
│   └──────────┬──────────┘      ┌──────────┴──────────┐        │
│   ┌──────────┴──────────┐      │    Infrastructure   │        │
│   │    Infrastructure   │      └─────────────────────┘        │
│   └─────────────────────┘                                      │
│                                                                  │
│   Heavy, Minutes to start      Light, Seconds to start         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Comparison Table

| Feature | Containers | Virtual Machines |
|---------|------------|------------------|
| Size | MBs | GBs |
| Startup | Seconds | Minutes |
| OS | Shares host kernel | Full OS per VM |
| Isolation | Process level | Hardware level |
| Resource usage | Lightweight | Heavy |
| Portability | High | Medium |
| Performance | Near native | Overhead |

## Docker Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Architecture                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client                    Docker Host                          │
│   ┌──────────┐             ┌────────────────────────────────┐   │
│   │ docker   │             │         Docker Daemon          │   │
│   │  CLI     │────────────►│          (dockerd)             │   │
│   │          │   REST API  │                                │   │
│   │ build    │             │  ┌──────────┐  ┌──────────┐   │   │
│   │ pull     │             │  │ Container│  │ Container│   │   │
│   │ run      │             │  └──────────┘  └──────────┘   │   │
│   └──────────┘             │                                │   │
│                            │  ┌──────────┐  ┌──────────┐   │   │
│                            │  │  Image   │  │  Image   │   │   │
│   Registry                 │  └──────────┘  └──────────┘   │   │
│   ┌──────────┐             │                                │   │
│   │Docker Hub│◄───────────►│  ┌─────────────────────────┐  │   │
│   │          │             │  │      Volumes/Networks    │  │   │
│   │ Images   │             │  └─────────────────────────┘  │   │
│   └──────────┘             └────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Description |
|-----------|-------------|
| Docker Client | CLI tool to interact with Docker |
| Docker Daemon | Background service managing containers |
| Docker Images | Read-only templates for containers |
| Docker Containers | Running instances of images |
| Docker Registry | Storage for Docker images |

## Core Concepts

### Images

An image is a read-only template with instructions for creating a container.

```bash
# List images
docker images

# Pull an image
docker pull nginx

# Image structure
┌─────────────────────┐
│    Application      │  Layer 4
├─────────────────────┤
│    Dependencies     │  Layer 3
├─────────────────────┤
│    Runtime          │  Layer 2
├─────────────────────┤
│    Base OS (Alpine) │  Layer 1
└─────────────────────┘
```

### Containers

A container is a runnable instance of an image.

```bash
# Run a container
docker run nginx

# Container lifecycle
Created → Running → Paused → Stopped → Removed
```

### Volumes

Volumes persist data beyond container lifecycle.

```bash
# Create volume
docker volume create mydata

# Use volume
docker run -v mydata:/app/data nginx
```

### Networks

Networks enable container communication.

```bash
# Create network
docker network create mynet

# Connect container
docker run --network mynet nginx
```

## Docker Use Cases

### Development

```
┌─────────────────────────────────────────┐
│  Consistent Development Environments    │
├─────────────────────────────────────────┤
│                                         │
│  Developer A    Developer B    CI/CD    │
│  ┌─────────┐   ┌─────────┐   ┌───────┐ │
│  │ Docker  │   │ Docker  │   │Docker │ │
│  │Container│   │Container│   │Build  │ │
│  │  Same   │   │  Same   │   │ Same  │ │
│  │  Image  │   │  Image  │   │ Image │ │
│  └─────────┘   └─────────┘   └───────┘ │
│                                         │
│  "Works on my machine" → Works everywhere│
└─────────────────────────────────────────┘
```

### Microservices

```
┌─────────────────────────────────────────┐
│        Microservices Architecture       │
├─────────────────────────────────────────┤
│                                         │
│  ┌────────┐ ┌────────┐ ┌────────┐      │
│  │  Web   │ │  API   │ │  Auth  │      │
│  │Service │ │Service │ │Service │      │
│  └────┬───┘ └────┬───┘ └────┬───┘      │
│       │          │          │           │
│  ┌────┴──────────┴──────────┴────┐     │
│  │         Docker Network         │     │
│  └────┬──────────┬──────────┬────┘     │
│       │          │          │           │
│  ┌────┴───┐ ┌────┴───┐ ┌────┴───┐      │
│  │  DB    │ │ Cache  │ │ Queue  │      │
│  │Service │ │Service │ │Service │      │
│  └────────┘ └────────┘ └────────┘      │
│                                         │
└─────────────────────────────────────────┘
```

### CI/CD Pipeline

```
┌─────────────────────────────────────────┐
│            CI/CD with Docker            │
├─────────────────────────────────────────┤
│                                         │
│  Code → Build → Test → Push → Deploy    │
│   │      │       │      │       │       │
│   ▼      ▼       ▼      ▼       ▼       │
│  Git   Docker  Docker  Docker  Docker   │
│  Push  Build   Run     Push    Deploy   │
│         │       │       │       │       │
│         ▼       ▼       ▼       ▼       │
│       Image   Test   Registry  Prod     │
│               Container        Cluster  │
│                                         │
└─────────────────────────────────────────┘
```

## Docker Workflow

### Basic Workflow

```bash
# 1. Write a Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
EOF

# 2. Build an image
docker build -t myapp:1.0 .

# 3. Run a container
docker run -d -p 3000:3000 myapp:1.0

# 4. Push to registry
docker push myregistry/myapp:1.0

# 5. Deploy anywhere
docker pull myregistry/myapp:1.0
docker run -d -p 3000:3000 myregistry/myapp:1.0
```

## Quick Reference

| Concept | Description |
|---------|-------------|
| Image | Blueprint for containers |
| Container | Running instance of image |
| Dockerfile | Instructions to build image |
| Registry | Storage for images |
| Volume | Persistent data storage |
| Network | Container communication |
| Compose | Multi-container orchestration |

---

**Next:** [02-docker-engine.md](02-docker-engine.md)
