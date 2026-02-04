# Introduction to Ansible

## What is Ansible?

Ansible is an open-source automation tool developed by Red Hat that simplifies IT infrastructure management, configuration management, application deployment, and orchestration.

### Key Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Agentless** | No software required on managed nodes |
| **Push-Based** | Control node pushes configurations to managed nodes |
| **Idempotent** | Running the same task multiple times produces the same result |
| **Human-Readable** | Uses YAML for playbooks |
| **SSH-Based** | Communicates over SSH (WinRM for Windows) |

### How Ansible Works

```
┌─────────────────┐         SSH/WinRM        ┌─────────────────┐
│  Control Node   │ ──────────────────────►  │  Managed Node   │
│  (Your Machine) │                          │  (Target Server)│
│                 │  ◄──────────────────────  │                 │
│  - Ansible      │        Results           │  - Python       │
│  - Playbooks    │                          │  - SSH Server   │
│  - Inventory    │                          │                 │
└─────────────────┘                          └─────────────────┘
```

---

## Why Ansible?

### Benefits

1. **Simple to Learn**
   - YAML-based syntax
   - No programming knowledge required
   - Extensive documentation

2. **Agentless Architecture**
   - No agent installation on managed nodes
   - Reduced maintenance overhead
   - Lower security footprint

3. **Powerful & Flexible**
   - 3000+ built-in modules
   - Extensible with custom modules
   - Supports multi-tier deployments

4. **Secure**
   - Uses SSH (no extra ports)
   - No root daemon required
   - Vault for secrets management

5. **Large Community**
   - Active development
   - Ansible Galaxy for shared roles
   - Extensive community support

---

## Use Cases

### 1. Configuration Management

```yaml
# Ensure packages are installed
- name: Install web server packages
  apt:
    name:
      - nginx
      - php-fpm
    state: present
```

**Examples:**
- Installing and configuring software
- Managing system configurations
- Ensuring consistent environments

### 2. Application Deployment

```yaml
# Deploy application
- name: Deploy web application
  git:
    repo: https://github.com/company/app.git
    dest: /var/www/app
    version: "{{ app_version }}"
```

**Examples:**
- Deploying code to servers
- Managing application updates
- Rolling deployments

### 3. Orchestration

```yaml
# Multi-tier deployment
- hosts: load_balancers
  tasks:
    - name: Disable server in pool

- hosts: web_servers
  serial: 1
  tasks:
    - name: Update application

- hosts: load_balancers
  tasks:
    - name: Enable server in pool
```

**Examples:**
- Coordinating multi-tier deployments
- Managing service dependencies
- Zero-downtime updates

### 4. Provisioning

```yaml
# Create cloud resources
- name: Create EC2 instance
  amazon.aws.ec2_instance:
    name: web-server
    instance_type: t3.micro
    image_id: ami-12345678
```

**Examples:**
- Creating cloud resources
- Setting up new environments
- Infrastructure as Code

### 5. Security & Compliance

```yaml
# Ensure compliance
- name: Set file permissions
  file:
    path: /etc/ssh/sshd_config
    owner: root
    mode: '0600'
```

**Examples:**
- Security hardening
- Compliance auditing
- Patch management

---

## Key Concepts

### Control Node

The machine where Ansible is installed and from which automation is run.

**Requirements:**
- Python 3.8+
- Linux, macOS, or WSL (not native Windows)
- SSH client

### Managed Nodes

The target machines that Ansible manages.

**Requirements:**
- Python 2.7+ or Python 3.5+
- SSH server running
- User account with appropriate permissions

### Inventory

A list of managed nodes organized in groups.

```ini
[webservers]
web1.example.com
web2.example.com

[databases]
db1.example.com
```

### Modules

Units of code that Ansible executes on managed nodes.

```yaml
# Using the 'apt' module
- apt:
    name: nginx
    state: present
```

### Playbooks

YAML files containing plays and tasks.

```yaml
---
- name: Configure web servers
  hosts: webservers
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
```

### Idempotency

Running the same playbook multiple times produces the same result without side effects.

```yaml
# This task is idempotent
- name: Create user
  user:
    name: deploy
    state: present
# Running twice won't create duplicate users
```

---

## Ansible vs Other Tools

| Feature | Ansible | Chef | Puppet | SaltStack |
|---------|---------|------|--------|-----------|
| **Architecture** | Agentless | Agent-based | Agent-based | Agent/Agentless |
| **Language** | YAML | Ruby DSL | Puppet DSL | YAML |
| **Communication** | SSH | HTTPS | HTTPS | ZeroMQ/SSH |
| **Learning Curve** | Low | High | Medium | Medium |
| **Configuration** | Push | Pull | Pull | Push/Pull |
| **Setup Complexity** | Simple | Complex | Complex | Medium |
| **Scalability** | Good | Excellent | Excellent | Excellent |
| **Windows Support** | Good | Good | Good | Good |

### When to Choose Ansible

- **Quick setup needed** - No agents to install
- **Team has limited experience** - Easy to learn YAML
- **Mixed environments** - Works with any SSH-enabled system
- **Ad-hoc tasks** - Great for one-off commands
- **Small to medium scale** - Simple architecture

### When to Consider Alternatives

- **Very large scale (10,000+ nodes)** - Consider SaltStack or Puppet
- **Need constant state enforcement** - Agent-based tools may be better
- **Complex dependencies** - Chef's Ruby DSL offers more flexibility

---

## Ansible Ecosystem

```
┌─────────────────────────────────────────────────────────────┐
│                    Ansible Ecosystem                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Ansible   │  │   Ansible   │  │  Ansible Automation │ │
│  │    Core     │  │   Galaxy    │  │      Platform       │ │
│  │             │  │             │  │    (Tower/AWX)      │ │
│  │  - CLI      │  │  - Roles    │  │                     │ │
│  │  - Modules  │  │  - Content  │  │  - Web UI           │ │
│  │  - Plugins  │  │  - Sharing  │  │  - RBAC             │ │
│  └─────────────┘  └─────────────┘  │  - Scheduling       │ │
│                                    │  - API              │ │
│  ┌─────────────┐  ┌─────────────┐  └─────────────────────┘ │
│  │  Ansible    │  │  Ansible    │                          │
│  │  Molecule   │  │    Lint     │                          │
│  │             │  │             │                          │
│  │  - Testing  │  │  - Linting  │                          │
│  │  - CI/CD    │  │  - Standards│                          │
│  └─────────────┘  └─────────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Getting Started Checklist

- [ ] Install Ansible on control node
- [ ] Set up SSH key authentication
- [ ] Create inventory file
- [ ] Test connectivity with `ansible all -m ping`
- [ ] Write your first playbook
- [ ] Run playbook with `ansible-playbook`

---

## Summary

| Concept | Description |
|---------|-------------|
| **Ansible** | Agentless automation tool |
| **Control Node** | Machine running Ansible |
| **Managed Node** | Target machine being configured |
| **Inventory** | List of managed nodes |
| **Module** | Unit of work (apt, copy, service) |
| **Playbook** | YAML file with automation tasks |
| **Idempotency** | Same result regardless of runs |

Ansible's simplicity, agentless architecture, and YAML-based syntax make it an excellent choice for DevOps automation, especially for teams new to configuration management.
