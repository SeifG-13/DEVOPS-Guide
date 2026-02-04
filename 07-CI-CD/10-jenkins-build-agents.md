# Jenkins Build Agents

## Master-Agent Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 Jenkins Master-Agent Architecture                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │               Jenkins Controller (Master)                │   │
│   │  • Scheduling builds                                     │   │
│   │  • Distributing work to agents                          │   │
│   │  • Monitoring agents                                     │   │
│   │  • Recording results                                     │   │
│   │  • Serving Jenkins UI                                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│            ┌───────────────┼───────────────┐                    │
│            │               │               │                    │
│            ▼               ▼               ▼                    │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│   │   Agent 1    │ │   Agent 2    │ │   Agent 3    │           │
│   │   (Linux)    │ │  (Windows)   │ │   (Docker)   │           │
│   │              │ │              │ │              │           │
│   │  Labels:     │ │  Labels:     │ │  Labels:     │           │
│   │  - linux     │ │  - windows   │ │  - docker    │           │
│   │  - java      │ │  - dotnet    │ │  - linux     │           │
│   └──────────────┘ └──────────────┘ └──────────────┘           │
│                                                                  │
│   Communication: JNLP (Java Web Start) or SSH                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Agent Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Permanent** | Always connected | Dedicated build servers |
| **Cloud** | On-demand provisioning | Dynamic scaling |
| **SSH** | Connect via SSH | Linux/Unix servers |
| **JNLP** | Agent initiates connection | Behind firewall |
| **Docker** | Run in containers | Isolated builds |
| **Kubernetes** | Run as K8s pods | Cloud-native |

## Adding a Permanent Agent

### Via SSH

```
Manage Jenkins → Nodes → New Node

Node name: linux-agent-01
Type: Permanent Agent

Configuration:
  # of executors: 4
  Remote root directory: /home/jenkins/agent
  Labels: linux java maven
  Usage: Only build jobs with label expressions matching this node
  Launch method: Launch agents via SSH
    Host: 192.168.1.100
    Credentials: jenkins-ssh-key
    Host Key Verification Strategy: Known hosts file
  Availability: Keep this agent online as much as possible
```

### Prepare SSH Agent (on agent machine)

```bash
# Create jenkins user
sudo useradd -m -d /home/jenkins jenkins
sudo mkdir -p /home/jenkins/agent
sudo chown jenkins:jenkins /home/jenkins/agent

# Add SSH key
sudo mkdir -p /home/jenkins/.ssh
sudo cat > /home/jenkins/.ssh/authorized_keys << 'EOF'
ssh-rsa AAAAB3NzaC1yc2E... jenkins@master
EOF
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh
sudo chmod 600 /home/jenkins/.ssh/authorized_keys

# Install Java
sudo apt install -y openjdk-17-jdk

# Verify
java -version
```

## JNLP Agent (Inbound Agent)

### Configure Agent Node

```
Manage Jenkins → Nodes → New Node

Node name: windows-agent-01
Type: Permanent Agent

Configuration:
  # of executors: 2
  Remote root directory: C:\Jenkins\agent
  Labels: windows dotnet
  Launch method: Launch agent by connecting it to the controller
  Availability: Keep this agent online as much as possible
```

### Connect from Agent Machine

```bash
# Download agent.jar from Jenkins
curl -O http://jenkins-server:8080/jnlpJars/agent.jar

# Connect (get secret from Jenkins UI)
java -jar agent.jar \
    -jnlpUrl http://jenkins-server:8080/computer/windows-agent-01/jenkins-agent.jnlp \
    -secret <secret-from-jenkins> \
    -workDir "C:\Jenkins\agent"
```

### JNLP Agent as Windows Service

```powershell
# Download Jenkins agent
Invoke-WebRequest -Uri "http://jenkins-server:8080/jnlpJars/agent.jar" -OutFile "C:\Jenkins\agent.jar"

# Create service using NSSM
nssm install JenkinsAgent java
nssm set JenkinsAgent AppDirectory C:\Jenkins
nssm set JenkinsAgent AppParameters -jar agent.jar -jnlpUrl http://jenkins-server:8080/computer/windows-agent-01/jenkins-agent.jnlp -secret <secret>
nssm start JenkinsAgent
```

### JNLP Agent as Linux Service

```bash
# /etc/systemd/system/jenkins-agent.service
[Unit]
Description=Jenkins Agent
After=network.target

[Service]
User=jenkins
WorkingDirectory=/home/jenkins/agent
ExecStart=/usr/bin/java -jar /home/jenkins/agent/agent.jar \
    -jnlpUrl http://jenkins-server:8080/computer/linux-agent-01/jenkins-agent.jnlp \
    -secret <secret> \
    -workDir /home/jenkins/agent
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins-agent
sudo systemctl start jenkins-agent
```

## Docker Agent

### Docker Cloud Configuration

```
Manage Jenkins → Clouds → New cloud → Docker

Docker Cloud details:
  Name: docker-cloud
  Docker Host URI: unix:///var/run/docker.sock
    # Or TCP: tcp://docker-host:2376

Docker Agent templates:
  Labels: docker linux
  Docker Image: jenkins/inbound-agent:latest
  Remote File System Root: /home/jenkins/agent
  Connect method: Attach Docker container
```

### Pipeline with Docker Agent

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '-v $HOME/.m2:/root/.m2'
        }
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

### Custom Docker Agent

```groovy
pipeline {
    agent {
        dockerfile {
            filename 'Dockerfile.build'
            dir 'ci'
            additionalBuildArgs '--build-arg VERSION=1.0'
            args '-v /tmp:/tmp'
        }
    }

    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }
}
```

## Kubernetes Agent

### Kubernetes Cloud Configuration

```
Manage Jenkins → Clouds → New cloud → Kubernetes

Kubernetes Cloud details:
  Name: kubernetes
  Kubernetes URL: https://kubernetes.default
  Kubernetes Namespace: jenkins
  Credentials: kubernetes-token
  Jenkins URL: http://jenkins.jenkins.svc.cluster.local:8080

Pod Templates:
  Name: maven
  Labels: maven
  Containers:
    - Name: maven
      Docker image: maven:3.9-eclipse-temurin-17
      Working directory: /home/jenkins/agent
      Command to run: (leave empty)
```

### Pipeline with Kubernetes Agent

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: maven
                image: maven:3.9-eclipse-temurin-17
                command:
                - cat
                tty: true
              - name: docker
                image: docker:24-dind
                securityContext:
                  privileged: true
            '''
        }
    }

    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp .'
                }
            }
        }
    }
}
```

## Labels and Node Selection

### Using Labels in Pipeline

```groovy
pipeline {
    agent {
        label 'linux && java'  // Must have both labels
    }
    stages {
        stage('Build') {
            steps {
                sh 'java -version'
            }
        }
    }
}

// OR logic
pipeline {
    agent {
        label 'linux || macos'  // Either label
    }
    stages { ... }
}

// Specific node
pipeline {
    agent {
        label 'linux-agent-01'
    }
    stages { ... }
}
```

### Per-Stage Agent

```groovy
pipeline {
    agent none

    stages {
        stage('Build on Linux') {
            agent { label 'linux' }
            steps {
                sh 'make build'
            }
        }
        stage('Build on Windows') {
            agent { label 'windows' }
            steps {
                bat 'msbuild project.sln'
            }
        }
        stage('Deploy') {
            agent { label 'deploy' }
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

## Agent Management

### Managing Agents

```
Manage Jenkins → Nodes

Actions:
• Configure - Edit agent settings
• Delete - Remove agent
• Mark Offline - Temporarily disable
• Disconnect - Disconnect agent
• Bring Online - Enable agent
```

### Monitor Agent Status

```bash
# Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ list-nodes

# Via API
curl http://localhost:8080/computer/api/json
```

### Agent Offline Message

```groovy
// In pipeline, check agent availability
pipeline {
    agent any
    stages {
        stage('Check Agent') {
            steps {
                script {
                    def node = Jenkins.instance.getNode('agent-name')
                    if (node?.toComputer()?.isOffline()) {
                        error "Agent is offline"
                    }
                }
            }
        }
    }
}
```

## Agent Security

### Restrict Agent Access

```
Manage Jenkins → Nodes → agent-name → Configure

Node Properties:
  ✓ Restrict project execution
    Allow job permissions: job1, job2, folder/*
```

### Agent-to-Controller Security

```
Manage Jenkins → Security → Agent → Controller Security

Enable Agent → Controller Access Control: ✓

Rules:
• File path rules
• Command rules
```

## Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                   Agent Best Practices                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Architecture                                                  │
│   ✓ Don't run builds on controller (set executors to 0)        │
│   ✓ Use multiple agents for redundancy                         │
│   ✓ Use labels for job targeting                               │
│   ✓ Size agents appropriately (CPU, memory, disk)              │
│                                                                  │
│   Security                                                      │
│   ✓ Use SSH agents when possible                               │
│   ✓ Enable agent-to-controller access control                  │
│   ✓ Restrict project execution per agent                       │
│   ✓ Use separate credentials per agent                         │
│                                                                  │
│   Maintenance                                                   │
│   ✓ Monitor agent health                                       │
│   ✓ Clean up workspaces regularly                              │
│   ✓ Keep agent software updated                                │
│   ✓ Use cloud agents for dynamic scaling                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Reference

### Launch Methods

| Method | Use Case |
|--------|----------|
| SSH | Linux/Unix with SSH access |
| JNLP | Windows, behind firewall |
| Docker | Container-based builds |
| Kubernetes | Cloud-native, dynamic |

### Label Expressions

| Expression | Meaning |
|------------|---------|
| `linux` | Has linux label |
| `linux && java` | Has both labels |
| `linux \|\| macos` | Has either label |
| `!windows` | Doesn't have windows label |

### Agent Commands

| Command | Description |
|---------|-------------|
| `agent any` | Run on any available agent |
| `agent none` | Don't allocate agent |
| `agent { label 'x' }` | Run on agent with label |
| `agent { docker {...} }` | Run in Docker container |

---

**Previous:** [09-jenkins-shared-libraries.md](09-jenkins-shared-libraries.md) | **Next:** [11-jenkins-backup-restore.md](11-jenkins-backup-restore.md)
