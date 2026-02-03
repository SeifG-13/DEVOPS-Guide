# Docker Installation

## Installation Methods

| Method | Description | Best For |
|--------|-------------|----------|
| Docker Engine | CLI-only installation | Linux servers |
| Docker Desktop | GUI + Engine | Windows, macOS, Linux desktop |
| Docker in Docker | Container-based | CI/CD pipelines |

## Linux Installation

### Ubuntu/Debian

```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify installation
sudo docker run hello-world
```

### CentOS/RHEL/Fedora

```bash
# Remove old versions
sudo yum remove docker docker-client docker-client-latest \
    docker-common docker-latest docker-latest-logrotate \
    docker-logrotate docker-engine

# Install prerequisites
sudo yum install -y yum-utils

# Add Docker repository
sudo yum-config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker Engine
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
sudo docker run hello-world
```

### Fedora

```bash
# Install prerequisites
sudo dnf -y install dnf-plugins-core

# Add Docker repository
sudo dnf config-manager --add-repo \
    https://download.docker.com/linux/fedora/docker-ce.repo

# Install Docker
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker
```

### Quick Install Script

```bash
# Convenience script (for testing/development)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# For specific version
# VERSION=20.10 sh get-docker.sh
```

## Windows Installation

### Docker Desktop for Windows

**Requirements:**
- Windows 10/11 64-bit: Pro, Enterprise, or Education
- WSL 2 backend or Hyper-V
- 4GB RAM minimum

```powershell
# Enable WSL 2 (PowerShell as Admin)
wsl --install

# Download Docker Desktop
# https://desktop.docker.com/win/stable/Docker%20Desktop%20Installer.exe

# Or using winget
winget install Docker.DockerDesktop

# After installation, restart computer
# Launch Docker Desktop from Start Menu
```

### WSL 2 Backend Setup

```powershell
# Enable WSL
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Enable Virtual Machine Platform
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Restart computer

# Set WSL 2 as default
wsl --set-default-version 2

# Install Linux distribution
wsl --install -d Ubuntu
```

## macOS Installation

### Docker Desktop for Mac

**Requirements:**
- macOS 11 (Big Sur) or newer
- Apple Silicon (M1/M2) or Intel processor
- 4GB RAM minimum

```bash
# Using Homebrew
brew install --cask docker

# Or download from Docker website
# https://desktop.docker.com/mac/stable/Docker.dmg

# After installation
# Launch Docker from Applications
# Wait for Docker to start (whale icon in menu bar)

# Verify installation
docker run hello-world
```

### Colima (Alternative for macOS)

```bash
# Install Colima (lightweight alternative)
brew install colima docker

# Start Colima
colima start

# Verify
docker ps
```

## Post-Installation Steps

### Run Docker Without sudo (Linux)

```bash
# Create docker group
sudo groupadd docker

# Add user to docker group
sudo usermod -aG docker $USER

# Activate changes
newgrp docker

# Verify
docker run hello-world
```

### Configure Docker to Start on Boot

```bash
# Enable Docker service
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Disable auto-start
sudo systemctl disable docker.service
sudo systemctl disable containerd.service
```

### Configure Default Logging

```bash
# Edit daemon configuration
sudo vim /etc/docker/daemon.json

# Add logging configuration
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# Restart Docker
sudo systemctl restart docker
```

### Configure Storage Location

```bash
# Default location: /var/lib/docker

# Change storage location
sudo vim /etc/docker/daemon.json

{
  "data-root": "/mnt/docker-data"
}

# Stop Docker
sudo systemctl stop docker

# Move existing data
sudo mv /var/lib/docker /mnt/docker-data

# Restart Docker
sudo systemctl start docker

# Verify
docker info | grep "Docker Root Dir"
```

## Verify Installation

```bash
# Check Docker version
docker version

# Output:
# Client: Docker Engine - Community
#  Version:           24.0.x
#  API version:       1.43
#  ...
# Server: Docker Engine - Community
#  Engine:
#   Version:          24.0.x
#   ...

# Check system info
docker info

# Run test container
docker run hello-world

# Run interactive container
docker run -it ubuntu bash
```

## Docker Desktop Configuration

### Settings (Windows/macOS)

| Setting | Description |
|---------|-------------|
| Resources → CPU | CPU cores allocated |
| Resources → Memory | RAM allocated |
| Resources → Disk | Virtual disk size |
| Docker Engine | daemon.json configuration |
| Kubernetes | Enable/disable K8s |

### Command Line Configuration

```bash
# Docker Desktop uses a VM, access via settings or:
# macOS: ~/Library/Group Containers/group.com.docker/settings.json
# Windows: %APPDATA%\Docker\settings.json
```

## Uninstallation

### Linux

```bash
# Ubuntu/Debian
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd

# CentOS/RHEL
sudo yum remove docker-ce docker-ce-cli containerd.io
sudo rm -rf /var/lib/docker
```

### Windows

```powershell
# Uninstall via Settings → Apps
# Or via PowerShell
winget uninstall Docker.DockerDesktop

# Clean up WSL (optional)
wsl --unregister docker-desktop
wsl --unregister docker-desktop-data
```

### macOS

```bash
# Remove Docker Desktop
# Drag from Applications to Trash

# Or via Homebrew
brew uninstall --cask docker

# Clean up data
rm -rf ~/Library/Group\ Containers/group.com.docker
rm -rf ~/Library/Containers/com.docker.docker
rm -rf ~/.docker
```

## Troubleshooting

### Common Issues

```bash
# Permission denied
sudo usermod -aG docker $USER
# Log out and back in

# Cannot connect to Docker daemon
sudo systemctl status docker
sudo systemctl start docker

# WSL 2 issues (Windows)
wsl --update
wsl --shutdown

# Reset Docker Desktop
# Settings → Troubleshoot → Reset to factory defaults
```

### Check Logs

```bash
# Linux
sudo journalctl -u docker.service

# Docker Desktop
# View logs in Docker Desktop → Troubleshoot → Get support

# Container logs
docker logs <container_id>
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker version` | Show version info |
| `docker info` | System-wide information |
| `docker run hello-world` | Test installation |
| `systemctl start docker` | Start Docker daemon |
| `systemctl enable docker` | Enable auto-start |

---

**Previous:** [02-docker-engine.md](02-docker-engine.md) | **Next:** [04-basic-commands.md](04-basic-commands.md)
