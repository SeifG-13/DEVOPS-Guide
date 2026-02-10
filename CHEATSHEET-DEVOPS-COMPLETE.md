# Complete DevOps Engineer Cheat Sheet

> A comprehensive guide for DevOps interview preparation and daily reference

---

## Table of Contents
1. [Linux Essentials](#1-linux-essentials)
2. [Networking](#2-networking)
3. [Git & Version Control](#3-git--version-control)
4. [Shell Scripting](#4-shell-scripting)
5. [Docker](#5-docker)
6. [Kubernetes](#6-kubernetes)
7. [CI/CD](#7-cicd)
8. [Ansible](#8-ansible)
9. [Terraform](#9-terraform)
10. [Cloud (Azure)](#10-cloud-azure)
11. [Monitoring (Prometheus/Grafana)](#11-monitoring-prometheusgrafana)
12. [Logging (ELK Stack)](#12-logging-elk-stack)
13. [GitOps (ArgoCD)](#13-gitops-argocd)
14. [Security (DevSecOps)](#14-security-devsecops)
15. [Helm](#15-helm)
16. [Interview Quick Tips](#16-interview-quick-tips)

---

## 1. Linux Essentials

### Must-Know Commands
```bash
# File Operations
ls -la | head | tail | cat | grep | find | chmod | chown

# Process Management
ps aux | top | htop | kill -9 PID | systemctl status/start/stop

# Networking
ip addr | ss -tulnp | netstat | curl | wget | ping | traceroute

# Disk & Storage
df -h | du -sh | lsblk | mount | fdisk

# Logs
journalctl -u service | tail -f /var/log/syslog
```

### Key Interview Topics
| Topic | Key Points |
|-------|------------|
| File Permissions | 755 (rwxr-xr-x), 644 (rw-r--r--), chmod, chown |
| Process vs Thread | Process: own memory, Thread: shared memory |
| Boot Process | BIOS → GRUB → Kernel → Init/Systemd |
| Hard vs Soft Link | Hard: inode, Soft: pointer to filename |
| Runlevels | 0=halt, 1=single, 3=multi-user, 5=GUI, 6=reboot |

---

## 2. Networking

### OSI Model (7 Layers)
```
7. Application  - HTTP, DNS, FTP, SSH
6. Presentation - SSL/TLS, Encryption
5. Session      - Sessions, Sockets
4. Transport    - TCP, UDP (ports)
3. Network      - IP, ICMP, Routing
2. Data Link    - MAC, ARP, Switches
1. Physical     - Cables, Signals
```

### Key Protocols & Ports
| Port | Service | Port | Service |
|------|---------|------|---------|
| 22 | SSH | 443 | HTTPS |
| 80 | HTTP | 3306 | MySQL |
| 53 | DNS | 5432 | PostgreSQL |

### TCP vs UDP
- **TCP**: Connection-oriented, reliable, ordered (HTTP, SSH)
- **UDP**: Connectionless, fast, unreliable (DNS, streaming)

### TCP 3-Way Handshake
```
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
```

---

## 3. Git & Version Control

### Essential Commands
```bash
git clone/init/status/add/commit/push/pull
git branch/checkout/merge/rebase
git stash/log/diff/reset/revert
git tag -a v1.0.0 -m "Release"
```

### Merge vs Rebase
- **Merge**: Preserves history, creates merge commit
- **Rebase**: Linear history, rewrites commits (don't rebase shared branches)

### Git Workflow Strategies
- **GitFlow**: main/develop/feature/release/hotfix
- **GitHub Flow**: main + feature branches + PRs
- **Trunk-Based**: Short-lived branches, frequent merges

---

## 4. Shell Scripting

### Script Template
```bash
#!/bin/bash
set -euo pipefail  # Exit on error, undefined vars, pipe failures

# Variables
VAR="value"
readonly CONST="immutable"

# Conditionals
if [[ condition ]]; then
    commands
fi

# Loops
for item in list; do
    echo "$item"
done

# Functions
my_function() {
    local var="$1"
    echo "$var"
}
```

### Key Concepts
- `$0, $1, $#, $@, $?, $$` - Special variables
- `[[ ]]` vs `[ ]` - Modern vs POSIX test
- `set -e` - Exit on error
- Always quote variables: `"$var"`

---

## 5. Docker

### Essential Commands
```bash
docker run -d -p 8080:80 --name app nginx
docker ps | logs | exec -it container bash
docker build -t image:tag .
docker-compose up -d | down | logs
```

### Dockerfile Best Practices
```dockerfile
FROM node:18-alpine          # Specific version, minimal base
WORKDIR /app
COPY package*.json ./        # Layer caching
RUN npm ci --only=production
COPY . .
USER node                    # Non-root user
EXPOSE 3000
CMD ["node", "server.js"]
```

### Key Concepts
| Concept | Description |
|---------|-------------|
| Image vs Container | Blueprint vs Running instance |
| Volume vs Bind Mount | Managed storage vs Host filesystem |
| CMD vs ENTRYPOINT | Default args vs Main executable |
| Bridge vs Host Network | Isolated vs Shared networking |

---

## 6. Kubernetes

### Essential Commands
```bash
kubectl get pods/deployments/services -A
kubectl describe pod <name>
kubectl logs -f <pod>
kubectl exec -it <pod> -- bash
kubectl apply -f manifest.yaml
kubectl rollout status/history/undo deployment/<name>
```

### Core Resources
| Resource | Purpose |
|----------|---------|
| Pod | Smallest unit, one or more containers |
| Deployment | Manages ReplicaSets, rolling updates |
| Service | Network access (ClusterIP, NodePort, LoadBalancer) |
| ConfigMap | Non-sensitive configuration |
| Secret | Sensitive data (base64) |
| Ingress | L7 routing, TLS termination |

### Probes
- **Liveness**: Is container alive? (restart if fails)
- **Readiness**: Is container ready for traffic?
- **Startup**: For slow-starting applications

---

## 7. CI/CD

### Pipeline Stages
```
Source → Build → Test → Security Scan → Package → Deploy → Monitor
```

### Deployment Strategies
| Strategy | Description | Rollback |
|----------|-------------|----------|
| Rolling | Gradual replacement | Medium |
| Blue-Green | Two identical environments | Fast |
| Canary | Small percentage first | Fast |
| Recreate | All at once | Slow |

### Key Metrics (DORA)
- **Lead Time**: Code commit to production
- **Deployment Frequency**: How often you deploy
- **Change Failure Rate**: Failed deployments
- **MTTR**: Mean time to recovery

---

## 8. Ansible

### Essential Commands
```bash
ansible all -m ping
ansible-playbook playbook.yml -i inventory
ansible-playbook playbook.yml --check  # Dry run
ansible-vault encrypt/decrypt secrets.yml
```

### Playbook Structure
```yaml
- name: Configure servers
  hosts: webservers
  become: yes
  vars:
    app_port: 8080
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
      notify: Restart nginx
  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### Key Concepts
- **Idempotency**: Safe to run multiple times
- **Roles**: Reusable automation units
- **Handlers**: Run once when notified
- **Vault**: Encrypted secrets

---

## 9. Terraform

### Essential Commands
```bash
terraform init      # Initialize
terraform plan      # Preview changes
terraform apply     # Apply changes
terraform destroy   # Destroy infrastructure
terraform state list/show/mv/rm
```

### Configuration Structure
```hcl
provider "aws" {
  region = "us-east-1"
}

variable "instance_type" {
  default = "t2.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = var.instance_type
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

### Key Concepts
- **State**: Maps real resources to config
- **Modules**: Reusable components
- **Remote Backend**: Shared state (S3, Azure Storage)
- **Workspaces**: Multiple environments

---

## 10. Cloud (Azure)

### Essential CLI Commands
```bash
az login
az group create -n myRG -l eastus
az vm create -g myRG -n myVM --image Ubuntu2204
az aks create -g myRG -n myAKS --node-count 3
az acr create -g myRG -n myacr --sku Basic
az webapp create -g myRG -p myPlan -n myApp
```

### Core Services
| Category | Services |
|----------|----------|
| Compute | VMs, App Service, AKS, Functions |
| Storage | Blob, Files, Disks |
| Database | SQL, Cosmos DB, PostgreSQL |
| Networking | VNet, Load Balancer, App Gateway |
| Security | Key Vault, Azure AD |
| Monitoring | Azure Monitor, App Insights |

---

## 11. Monitoring (Prometheus/Grafana)

### PromQL Essentials
```promql
# Rate (per-second)
rate(http_requests_total[5m])

# Sum by label
sum(rate(http_requests_total[5m])) by (status)

# Histogram percentile
histogram_quantile(0.95, rate(http_request_duration_bucket[5m]))

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

### Four Golden Signals
1. **Latency**: Response time
2. **Traffic**: Requests per second
3. **Errors**: Error rate
4. **Saturation**: Resource usage

---

## 12. Logging (ELK Stack)

### Stack Components
```
Application → Filebeat → Logstash → Elasticsearch → Kibana
              (Ship)     (Process)   (Store)        (Visualize)
```

### KQL (Kibana Query)
```
level: "ERROR" AND service: "api"
status: 500
@timestamp >= "2024-01-01"
message: "connection*"
```

### Best Practices
- Use structured logging (JSON)
- Include correlation IDs
- Set appropriate log levels
- Implement log rotation

---

## 13. GitOps (ArgoCD)

### Core Principles
1. **Declarative**: Everything in Git
2. **Versioned**: Git is source of truth
3. **Automated**: Auto-sync to cluster
4. **Reconciled**: Self-healing

### Essential Commands
```bash
argocd app create myapp --repo url --path k8s --dest-server https://kubernetes.default.svc
argocd app sync myapp
argocd app list
argocd app rollback myapp <revision>
```

### Sync Policies
```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources not in Git
    selfHeal: true   # Revert manual changes
```

---

## 14. Security (DevSecOps)

### Security Pipeline
```
SAST → SCA → Build → Container Scan → DAST → Deploy
(Code) (Deps) (Image)  (Trivy)      (Runtime)
```

### OWASP Top 10
1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable Components
7. Authentication Failures
8. Data Integrity Failures
9. Logging Failures
10. SSRF

### Container Security
```dockerfile
FROM alpine:3.18           # Minimal base
RUN adduser -D appuser     # Non-root
USER appuser
# Never store secrets in image
```

---

## 15. Helm

### Essential Commands
```bash
helm repo add/update/list
helm install myrelease chart -f values.yaml
helm upgrade myrelease chart --set key=value
helm rollback myrelease <revision>
helm uninstall myrelease
helm template myrelease chart  # Render locally
```

### Chart Structure
```
mychart/
├── Chart.yaml          # Metadata
├── values.yaml         # Default values
├── templates/          # K8s manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl
└── charts/             # Dependencies
```

---

## 16. Interview Quick Tips

### Behavioral Questions
- Use STAR method: Situation, Task, Action, Result
- Prepare examples of troubleshooting scenarios
- Know your projects deeply

### Technical Questions Pattern
1. Explain the concept
2. Describe how you've used it
3. Mention best practices
4. Discuss trade-offs

### Common Scenario Questions
| Question | Key Points |
|----------|------------|
| "How do you troubleshoot a slow server?" | top, htop, iostat, netstat, logs |
| "Describe your CI/CD pipeline" | Stages, tools, testing, deployment strategy |
| "How do you handle secrets?" | Vault, K8s secrets, Key Vault, never in code |
| "Explain your monitoring setup" | Metrics, logs, alerts, dashboards |
| "How do you ensure zero-downtime?" | Rolling updates, blue-green, health checks |

### Quick Reference Numbers
| Metric | Typical Value |
|--------|---------------|
| SLA target | 99.9% (8.76h downtime/year) |
| API latency p99 | < 500ms |
| Deployment frequency | Daily to weekly |
| MTTR | < 1 hour |
| Container startup | < 30 seconds |

---

## Quick Command Reference Card

### Daily Commands
```bash
# Kubernetes
kubectl get pods -A
kubectl logs -f deployment/app
kubectl describe pod <pod>

# Docker
docker ps -a
docker logs -f container
docker-compose up -d

# Git
git status && git pull
git checkout -b feature/name
git push -u origin branch

# Linux
systemctl status service
journalctl -u service -f
df -h && free -h
```

### Troubleshooting Flow
```
1. Check service status (systemctl, kubectl)
2. Check logs (journalctl, kubectl logs)
3. Check resources (top, df, kubectl top)
4. Check network (ss, curl, telnet)
5. Check configuration (env vars, configmaps)
```

---

> **Pro Tip**: Focus on understanding concepts, not memorizing commands. Interviewers want to see problem-solving skills and practical experience.
