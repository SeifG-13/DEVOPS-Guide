# Jenkins Installation

## What is Jenkins?

Jenkins is an open-source automation server for building, testing, and deploying software.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Jenkins Architecture                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Jenkins Controller                     │   │
│   │                    (Master Node)                         │   │
│   │  ┌───────────┐ ┌───────────┐ ┌───────────────────────┐ │   │
│   │  │    Web    │ │   REST    │ │     Job Scheduler     │ │   │
│   │  │    UI     │ │    API    │ │                       │ │   │
│   │  └───────────┘ └───────────┘ └───────────────────────┘ │   │
│   │  ┌───────────┐ ┌───────────┐ ┌───────────────────────┐ │   │
│   │  │  Plugins  │ │Credentials│ │    Build Queue        │ │   │
│   │  │           │ │  Store    │ │                       │ │   │
│   │  └───────────┘ └───────────┘ └───────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│              ┌─────────────┼─────────────┐                      │
│              ▼             ▼             ▼                      │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│   │   Agent 1    │ │   Agent 2    │ │   Agent 3    │           │
│   │   (Linux)    │ │  (Windows)   │ │   (Docker)   │           │
│   └──────────────┘ └──────────────┘ └──────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| RAM | 256 MB | 4 GB+ |
| Disk Space | 1 GB | 50 GB+ |
| Java | JDK 11 or 17 | JDK 17 |

## Installation on Ubuntu/Debian

### Step 1: Install Java

```bash
# Update system
sudo apt update

# Install Java 17
sudo apt install -y openjdk-17-jdk

# Verify Java installation
java -version

# Set JAVA_HOME
echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> ~/.bashrc
source ~/.bashrc
```

### Step 2: Add Jenkins Repository

```bash
# Add Jenkins GPG key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### Step 3: Install Jenkins

```bash
# Update package list
sudo apt update

# Install Jenkins
sudo apt install -y jenkins

# Verify installation
jenkins --version
```

### Step 4: Start Jenkins Service

```bash
# Start Jenkins
sudo systemctl start jenkins

# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Check Jenkins status
sudo systemctl status jenkins

# View Jenkins logs
sudo journalctl -u jenkins -f
```

### Step 5: Open Firewall Port

```bash
# Ubuntu/Debian with UFW
sudo ufw allow 8080
sudo ufw status

# Or with iptables
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

## Installation on CentOS/RHEL

### Step 1: Install Java

```bash
# Install Java 17
sudo yum install -y java-17-openjdk java-17-openjdk-devel

# Verify
java -version
```

### Step 2: Add Jenkins Repository

```bash
# Add Jenkins repo
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

# Import key
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

### Step 3: Install and Start

```bash
# Install Jenkins
sudo yum install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Open firewall port
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

## Installation on Windows

### Step 1: Download and Install

```powershell
# Download Jenkins MSI from https://www.jenkins.io/download/
# Or use Chocolatey
choco install jenkins

# Jenkins installs as a Windows service
# Default location: C:\Program Files\Jenkins
```

### Step 2: Configure Service

```powershell
# Check service status
Get-Service jenkins

# Start service
Start-Service jenkins

# Open firewall port
New-NetFirewallRule -DisplayName "Jenkins" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow
```

## Installation with Docker

### Run Jenkins Container

```bash
# Create volume for Jenkins data
docker volume create jenkins_home

# Run Jenkins
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# View logs
docker logs -f jenkins

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false

volumes:
  jenkins_home:
```

```bash
# Start with Docker Compose
docker-compose up -d
```

## Initial Setup

### Step 1: Get Initial Admin Password

```bash
# Linux
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Docker
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Windows
type "C:\Program Files\Jenkins\secrets\initialAdminPassword"
```

### Step 2: Access Jenkins Web UI

```
1. Open browser: http://localhost:8080
2. Enter initial admin password
3. Select plugin installation option:
   - Install suggested plugins (recommended for beginners)
   - Select plugins to install (for advanced users)
4. Create first admin user
5. Configure Jenkins URL
6. Start using Jenkins!
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   Initial Setup Flow                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Unlock Jenkins                                             │
│      └── Enter initial admin password                          │
│                                                                  │
│   2. Customize Jenkins                                          │
│      ├── Install suggested plugins                              │
│      └── Select plugins to install                              │
│                                                                  │
│   3. Create First Admin User                                    │
│      ├── Username                                               │
│      ├── Password                                               │
│      ├── Full name                                              │
│      └── Email                                                  │
│                                                                  │
│   4. Instance Configuration                                     │
│      └── Jenkins URL (e.g., http://jenkins.example.com:8080)   │
│                                                                  │
│   5. Jenkins is ready!                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Jenkins Directory Structure

```
/var/lib/jenkins/           # JENKINS_HOME (Linux)
├── config.xml              # Global configuration
├── credentials.xml         # Credentials
├── jobs/                   # Job configurations
│   └── my-job/
│       ├── config.xml
│       └── builds/
├── nodes/                  # Agent configurations
├── plugins/                # Installed plugins
├── secrets/                # Encrypted secrets
│   └── initialAdminPassword
├── users/                  # User configurations
├── workspace/              # Build workspaces
└── logs/                   # Jenkins logs
```

## Service Management

### Linux (systemd)

```bash
# Start Jenkins
sudo systemctl start jenkins

# Stop Jenkins
sudo systemctl stop jenkins

# Restart Jenkins
sudo systemctl restart jenkins

# Check status
sudo systemctl status jenkins

# View logs
sudo journalctl -u jenkins -f

# Reload configuration
sudo systemctl daemon-reload
```

### Configure Service Options

```bash
# Edit service configuration
sudo systemctl edit jenkins

# Or edit directly
sudo nano /etc/default/jenkins

# Common options:
JENKINS_PORT=8080
JENKINS_ARGS="--httpPort=$JENKINS_PORT"
JAVA_ARGS="-Xmx2048m -Djava.awt.headless=true"
```

### Custom Port Configuration

```bash
# Edit /etc/default/jenkins
HTTP_PORT=9090

# Or /lib/systemd/system/jenkins.service
Environment="JENKINS_PORT=9090"

# Restart Jenkins
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

## Verify Installation

```bash
# Check Jenkins is running
curl -I http://localhost:8080

# Check Jenkins version via CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ version

# Check service
systemctl is-active jenkins
```

## Troubleshooting Installation

### Common Issues

```bash
# Port already in use
sudo netstat -tlnp | grep 8080
# Change port in /etc/default/jenkins

# Java not found
which java
update-alternatives --config java

# Permission issues
sudo chown -R jenkins:jenkins /var/lib/jenkins

# View detailed logs
sudo tail -f /var/log/jenkins/jenkins.log
```

### Memory Issues

```bash
# Increase heap size in /etc/default/jenkins
JAVA_ARGS="-Xmx4096m -Xms1024m"

# Restart Jenkins
sudo systemctl restart jenkins
```

## Quick Reference

### Installation Commands

| OS | Install Command |
|----|-----------------|
| Ubuntu/Debian | `sudo apt install jenkins` |
| CentOS/RHEL | `sudo yum install jenkins` |
| Docker | `docker run jenkins/jenkins:lts` |
| Windows | Download MSI or `choco install jenkins` |

### Important Paths

| Path | Description |
|------|-------------|
| `/var/lib/jenkins` | JENKINS_HOME (Linux) |
| `/var/log/jenkins` | Log files |
| `/etc/default/jenkins` | Service configuration |
| `C:\Program Files\Jenkins` | JENKINS_HOME (Windows) |

### Default Ports

| Port | Purpose |
|------|---------|
| 8080 | Web UI |
| 50000 | Agent communication |

---

**Previous:** [02-version-control-integration.md](02-version-control-integration.md) | **Next:** [04-jenkins-configuration.md](04-jenkins-configuration.md)
