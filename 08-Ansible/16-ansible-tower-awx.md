# Ansible Tower / AWX

## What is Ansible Tower / AWX?

**Ansible Tower** is Red Hat's commercial web-based UI, REST API, and task engine for Ansible.
**AWX** is the open-source upstream project from which Tower is derived.

```
┌─────────────────────────────────────────────────────────────┐
│                    AWX / TOWER                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Web UI                            │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │Dashboard│ │Templates│ │Inventory│ │ Jobs    │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           │ REST API                        │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Task Engine                        │   │
│  │                                                      │   │
│  │   Scheduling │ RBAC │ Logging │ Notifications       │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Ansible Execution                       │   │
│  │                                                      │   │
│  │   Playbooks ──► Inventory ──► Managed Nodes         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Tower vs AWX

| Feature | AWX | Ansible Tower |
|---------|-----|---------------|
| **License** | Open Source (Apache 2.0) | Commercial (Subscription) |
| **Support** | Community | Red Hat Support |
| **Updates** | Frequent (can be unstable) | Stable releases |
| **Use Case** | Development, Learning | Production, Enterprise |
| **Cost** | Free | Paid subscription |

---

## AWX Installation

### Prerequisites

- Docker and Docker Compose
- 4GB+ RAM
- 20GB+ disk space
- Linux host (RHEL, CentOS, Ubuntu)

### Docker Compose Installation

```bash
# Clone AWX repository
git clone https://github.com/ansible/awx.git
cd awx

# Install dependencies
pip install docker-compose

# Run the installer
cd tools/docker-compose
make docker-compose-build
docker-compose up -d

# Default credentials:
# URL: http://localhost:8080
# Username: admin
# Password: password (change immediately!)
```

### Kubernetes Installation (AWX Operator)

```bash
# Install AWX Operator
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/main/deploy/awx-operator.yaml

# Create AWX instance
cat <<EOF | kubectl apply -f -
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
EOF

# Get admin password
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
```

---

## Key Concepts

### Organizations

Top-level container for users, teams, projects, and inventories.

```
Organization: "ACME Corp"
├── Teams
│   ├── DevOps
│   └── Developers
├── Projects
│   ├── Infrastructure
│   └── Applications
├── Inventories
│   ├── Production
│   └── Staging
└── Users
    ├── admin
    └── operators
```

### Projects

SCM-backed collections of playbooks.

```yaml
# Project configuration
Name: "Web Infrastructure"
Organization: "ACME Corp"
SCM Type: Git
SCM URL: https://github.com/company/ansible-playbooks.git
SCM Branch: main
SCM Credential: GitHub Token
Update on Launch: Yes
```

### Inventories

Collections of hosts and groups.

```yaml
# Inventory structure
Name: "Production"
Organization: "ACME Corp"
Variables:
  environment: production

Groups:
  - webservers:
      hosts:
        - web1.example.com
        - web2.example.com
      variables:
        http_port: 80

  - databases:
      hosts:
        - db1.example.com
      variables:
        db_port: 5432
```

### Credentials

Securely stored authentication information.

| Credential Type | Use Case |
|-----------------|----------|
| Machine | SSH access to managed nodes |
| Source Control | Git/SVN authentication |
| Vault | Ansible Vault password |
| Cloud | AWS, Azure, GCP authentication |
| Network | Network device authentication |
| Container Registry | Docker/Podman registry |

```yaml
# Machine credential
Name: "Production SSH"
Credential Type: Machine
Username: ansible
SSH Private Key: (stored encrypted)
Privilege Escalation Method: sudo
```

### Job Templates

Definition of how to run a playbook.

```yaml
# Job Template
Name: "Deploy Web Application"
Job Type: Run
Inventory: Production
Project: Web Infrastructure
Playbook: deploy.yml
Credentials:
  - Production SSH
  - Vault Password
Variables:
  version: "1.2.3"
Limit: webservers
Verbosity: 0 (Normal)
Job Tags: deploy,restart
```

### Workflows

Chain multiple job templates together.

```
┌─────────────────────────────────────────────────────────────┐
│                      WORKFLOW                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────┐     ┌─────────┐     ┌─────────┐             │
│   │ Deploy  │────►│  Test   │────►│ Notify  │             │
│   │   App   │     │   App   │     │  Team   │             │
│   └─────────┘     └────┬────┘     └─────────┘             │
│                        │                                    │
│                   On Failure                                │
│                        │                                    │
│                        ▼                                    │
│                   ┌─────────┐                              │
│                   │Rollback │                              │
│                   └─────────┘                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Web Interface

### Dashboard

- Job status overview
- Recent job activity
- Host status
- Inventory sync status
- Project sync status

### Templates

Create and manage job templates:

1. **Name**: Descriptive name
2. **Job Type**: Run, Check, or Scan
3. **Inventory**: Target hosts
4. **Project**: Source of playbooks
5. **Playbook**: Specific playbook to run
6. **Credentials**: Authentication
7. **Variables**: Extra variables (YAML/JSON)
8. **Limit**: Subset of inventory
9. **Tags**: Job tags to run

### Inventory Management

#### Static Inventory

Add hosts and groups manually through UI.

#### Dynamic Inventory (Smart Inventory)

```yaml
# Smart Inventory Filter
kind: smart
host_filter: ansible_facts.os_family:Debian
```

#### Inventory Sources

Sync from external sources:
- Amazon EC2
- Google Compute Engine
- Microsoft Azure
- VMware vCenter
- Red Hat Satellite
- Custom scripts

```yaml
# AWS EC2 Inventory Source
Name: "AWS Production"
Source: Amazon EC2
Credential: AWS Production
Regions: us-east-1, us-west-2
Instance Filters: tag:Environment=production
Update on Launch: Yes
Overwrite: Yes
```

---

## RBAC (Role-Based Access Control)

### Built-in Roles

| Role | Permissions |
|------|-------------|
| **Admin** | Full access |
| **Auditor** | Read-only access to all |
| **Execute** | Run jobs |
| **Read** | View only |
| **Use** | Use in job templates |
| **Update** | Modify resources |

### Team Permissions

```yaml
# Team: DevOps
Permissions:
  - Organization: ACME Corp (Admin)
  - Project: Infrastructure (Admin)
  - Project: Applications (Use)
  - Inventory: Production (Use)
  - Inventory: Staging (Admin)
  - Job Template: Deploy App (Execute)
```

### User Roles

```yaml
# User: john.doe
Organization Roles:
  - ACME Corp: Member

Team Memberships:
  - DevOps: Member

Direct Permissions:
  - Job Template "Emergency Restart": Execute
```

---

## API Usage

### Authentication

```bash
# Get auth token
curl -X POST https://awx.example.com/api/v2/tokens/ \
  -H "Content-Type: application/json" \
  -u admin:password

# Use token in requests
curl https://awx.example.com/api/v2/me/ \
  -H "Authorization: Bearer <token>"
```

### Common API Operations

```bash
# List job templates
curl https://awx.example.com/api/v2/job_templates/ \
  -H "Authorization: Bearer <token>"

# Launch a job
curl -X POST https://awx.example.com/api/v2/job_templates/1/launch/ \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": {"version": "1.2.3"}}'

# Check job status
curl https://awx.example.com/api/v2/jobs/123/ \
  -H "Authorization: Bearer <token>"

# List inventories
curl https://awx.example.com/api/v2/inventories/ \
  -H "Authorization: Bearer <token>"

# Add host to inventory
curl -X POST https://awx.example.com/api/v2/hosts/ \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "web3.example.com",
    "inventory": 1,
    "variables": "ansible_host: 192.168.1.13"
  }'
```

### Using awx CLI

```bash
# Install awx CLI
pip install awxkit

# Configure
export TOWER_HOST=https://awx.example.com
export TOWER_USERNAME=admin
export TOWER_PASSWORD=password

# Or use config file
# ~/.tower_cli.cfg
# [general]
# host = https://awx.example.com
# username = admin
# password = password
# verify_ssl = true

# List job templates
awx job_templates list

# Launch job
awx job_templates launch 1 --extra_vars '{"version": "1.2.3"}'

# Monitor job
awx jobs monitor 123

# Get job output
awx jobs stdout 123
```

---

## Schedules

### Creating Schedules

```yaml
# Schedule configuration
Name: "Nightly Backup"
Job Template: "Backup Infrastructure"
Frequency: Daily
Start Time: 02:00 AM UTC
Timezone: UTC
Repeat: Every 1 day

# Cron-style schedule
Name: "Weekly Maintenance"
RRule: DTSTART:20240101T060000Z RRULE:FREQ=WEEKLY;BYDAY=SU
```

### Schedule Types

- **Run Once**: Single execution at specified time
- **Minute**: Every X minutes
- **Hour**: Every X hours
- **Day**: Daily at specified time
- **Week**: Weekly on specified days
- **Month**: Monthly on specified day

---

## Notifications

### Notification Types

| Type | Description |
|------|-------------|
| Email | SMTP email notifications |
| Slack | Slack channel messages |
| Microsoft Teams | Teams channel messages |
| PagerDuty | Incident alerts |
| Webhook | Custom HTTP callbacks |
| IRC | IRC channel messages |

### Notification Template

```yaml
# Slack Notification
Name: "Slack DevOps"
Type: Slack
Destination Channel: #devops-alerts
Token: xoxb-xxx-xxx

# Message Template (Jinja2)
{{ job_friendly_name }} #{{ job.id }} '{{ job.name }}'
{{ job.status }}: {{ url }}
```

### Notification Triggers

- **Start**: Job started
- **Success**: Job completed successfully
- **Failure**: Job failed
- **Approval**: Workflow approval needed

---

## Workflows

### Creating Workflows

```yaml
# Workflow: Deploy Application
Nodes:
  1. Pre-flight Checks
     └── On Success → 2. Deploy to Staging

  2. Deploy to Staging
     ├── On Success → 3. Test Staging
     └── On Failure → 5. Notify Failure

  3. Test Staging
     ├── On Success → 4. Deploy to Production
     └── On Failure → 5. Notify Failure

  4. Deploy to Production
     ├── On Success → 6. Notify Success
     └── On Failure → 5. Notify Failure

  5. Notify Failure (Notification)

  6. Notify Success (Notification)
```

### Workflow Features

- **Convergence**: Wait for multiple nodes to complete
- **Approval Nodes**: Require manual approval
- **Inventory/Credential Override**: Dynamic resource selection
- **Survey Variables**: Prompt for input at launch

### Approval Nodes

```yaml
# Workflow with approval
1. Deploy to Staging → Success
2. Approval Node (requires admin approval) → Approved
3. Deploy to Production
```

---

## Best Practices

### Organization Structure

```
Organization: Company
├── Teams
│   ├── Platform (Admin: Infrastructure projects)
│   ├── Development (Execute: App deployments)
│   └── Security (Audit: All resources)
├── Projects
│   ├── Infrastructure (Platform team)
│   ├── Applications (Development team)
│   └── Security-Scanning (Security team)
└── Inventories
    ├── Development (Development team - Admin)
    ├── Staging (Development team - Use)
    └── Production (Platform team - Admin)
```

### Credential Management

1. Use separate credentials per environment
2. Rotate credentials regularly
3. Use external credential lookups (HashiCorp Vault, CyberArk)
4. Never store credentials in projects

### Project Management

1. Use SCM (Git) for all playbooks
2. Enable "Update on Launch" for dynamic content
3. Use branches for environments (main, staging, develop)
4. Tag releases for production deployments

### Job Template Standards

1. Use descriptive names
2. Set appropriate verbosity
3. Enable job isolation
4. Configure proper timeouts
5. Add relevant job tags

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Organization** | Top-level container |
| **Project** | SCM-backed playbooks |
| **Inventory** | Target hosts |
| **Credential** | Secure authentication |
| **Job Template** | Playbook execution config |
| **Workflow** | Chain job templates |
| **Schedule** | Automated execution |
| **Notification** | Alerting and messaging |
| **RBAC** | Access control |
| **API** | Programmatic access |

### AWX vs Command Line

| Feature | AWX/Tower | CLI |
|---------|-----------|-----|
| Web UI | ✓ | ✗ |
| RBAC | ✓ | Limited |
| Scheduling | ✓ | Cron |
| Audit Logs | ✓ | Manual |
| API | ✓ | ✗ |
| Workflows | ✓ | Scripts |
| Learning Curve | Higher | Lower |
