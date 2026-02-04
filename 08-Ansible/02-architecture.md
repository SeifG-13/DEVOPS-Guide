# Ansible Architecture

## Overview

Ansible uses a simple, agentless architecture where a control node manages multiple target nodes over SSH.

```
┌────────────────────────────────────────────────────────────────────┐
│                      ANSIBLE ARCHITECTURE                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────┐                         │
│  │          CONTROL NODE                │                         │
│  │                                      │                         │
│  │  ┌────────────┐  ┌────────────────┐ │                         │
│  │  │  Ansible   │  │   Inventory    │ │                         │
│  │  │   Engine   │  │                │ │                         │
│  │  └────────────┘  └────────────────┘ │                         │
│  │                                      │                         │
│  │  ┌────────────┐  ┌────────────────┐ │                         │
│  │  │  Playbooks │  │    Modules     │ │                         │
│  │  │   (YAML)   │  │                │ │                         │
│  │  └────────────┘  └────────────────┘ │                         │
│  │                                      │                         │
│  │  ┌────────────┐  ┌────────────────┐ │                         │
│  │  │  Plugins   │  │  ansible.cfg   │ │                         │
│  │  │            │  │                │ │                         │
│  │  └────────────┘  └────────────────┘ │                         │
│  └──────────────────────────────────────┘                         │
│                        │                                           │
│                        │ SSH / WinRM                               │
│                        ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    MANAGED NODES                             │  │
│  │                                                              │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│  │  │   Linux     │  │   Linux     │  │   Windows   │         │  │
│  │  │   Server    │  │   Server    │  │   Server    │         │  │
│  │  │             │  │             │  │             │         │  │
│  │  │  - Python   │  │  - Python   │  │  - WinRM    │         │  │
│  │  │  - SSH      │  │  - SSH      │  │  - PowerShell│        │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Control Node

The control node is the machine where Ansible is installed and from which all tasks and playbooks are executed.

### Requirements

| Requirement | Details |
|-------------|---------|
| **Operating System** | Linux, macOS, or WSL (Windows Subsystem for Linux) |
| **Python** | Python 3.8 or higher |
| **SSH Client** | OpenSSH or compatible |
| **Network** | Access to managed nodes |

### What's on the Control Node

```
Control Node
├── Ansible Installation
│   ├── ansible (command)
│   ├── ansible-playbook
│   ├── ansible-galaxy
│   ├── ansible-vault
│   └── ansible-doc
├── Configuration
│   └── ansible.cfg
├── Inventory
│   ├── hosts (static)
│   └── dynamic scripts
├── Playbooks
│   └── *.yml files
├── Roles
│   └── role directories
└── Variables
    ├── group_vars/
    └── host_vars/
```

### Control Node Responsibilities

1. **Parse and validate playbooks**
2. **Read inventory** to determine target hosts
3. **Compile modules** and copy to managed nodes
4. **Execute tasks** via SSH/WinRM
5. **Collect results** and display output

---

## Managed Nodes

Managed nodes are the target machines that Ansible configures and manages.

### Linux/Unix Requirements

| Requirement | Details |
|-------------|---------|
| **SSH Server** | OpenSSH running and accessible |
| **Python** | Python 2.7+ or 3.5+ (for most modules) |
| **User Account** | With sudo privileges (for privileged tasks) |

### Windows Requirements

| Requirement | Details |
|-------------|---------|
| **WinRM** | Windows Remote Management enabled |
| **PowerShell** | PowerShell 3.0+ |
| **.NET Framework** | .NET 4.0+ |

### Network Devices

```yaml
# Network devices may use different connection types
ansible_connection: network_cli  # For CLI-based devices
ansible_connection: netconf      # For NETCONF-enabled devices
ansible_connection: httpapi      # For REST API devices
```

---

## Core Components

### 1. Inventory

Defines the hosts and groups Ansible manages.

```
┌─────────────────────────────────────────┐
│              INVENTORY                   │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────────┐                   │
│  │   All Hosts     │                   │
│  └────────┬────────┘                   │
│           │                             │
│     ┌─────┴─────┐                      │
│     │           │                      │
│  ┌──▼──┐     ┌──▼──┐                  │
│  │ web │     │ db  │  ◄── Groups      │
│  └──┬──┘     └──┬──┘                  │
│     │           │                      │
│  ┌──▼──┐     ┌──▼──┐                  │
│  │web1 │     │db1  │  ◄── Hosts       │
│  │web2 │     │db2  │                  │
│  └─────┘     └─────┘                  │
│                                         │
└─────────────────────────────────────────┘
```

**Types:**
- **Static Inventory**: INI or YAML files
- **Dynamic Inventory**: Scripts that query external sources (AWS, Azure, etc.)

### 2. Modules

Self-contained units of code that perform specific tasks.

```
┌─────────────────────────────────────────────────────────┐
│                    MODULE CATEGORIES                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   System    │  │    Files    │  │   Network   │    │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │
│  │ user        │  │ copy        │  │ uri         │    │
│  │ group       │  │ file        │  │ get_url     │    │
│  │ service     │  │ template    │  │ nmcli       │    │
│  │ cron        │  │ lineinfile  │  │             │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Packages   │  │   Cloud     │  │  Database   │    │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │
│  │ apt         │  │ ec2         │  │ mysql_db    │    │
│  │ yum         │  │ azure_rm    │  │ postgresql  │    │
│  │ pip         │  │ gcp_compute │  │ mongodb     │    │
│  │ dnf         │  │             │  │             │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 3. Plugins

Extend Ansible's core functionality.

| Plugin Type | Purpose | Examples |
|-------------|---------|----------|
| **Connection** | How to connect to hosts | ssh, winrm, local, docker |
| **Callback** | Control output display | json, yaml, minimal |
| **Lookup** | Retrieve data from sources | file, env, password |
| **Filter** | Transform data | to_json, regex_replace |
| **Inventory** | Dynamic inventory sources | aws_ec2, azure_rm |
| **Strategy** | Execution strategy | linear, free, debug |

### 4. Playbooks

YAML files that define automation workflows.

```yaml
---
# Playbook structure
- name: Play Name           # Play
  hosts: webservers         # Target hosts
  become: yes               # Privilege escalation

  vars:                     # Variables
    app_port: 8080

  tasks:                    # Tasks
    - name: Install nginx
      apt:
        name: nginx
        state: present

  handlers:                 # Handlers
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

---

## Communication Flow

### SSH Communication (Linux)

```
┌─────────────┐                              ┌─────────────┐
│   Control   │                              │   Managed   │
│    Node     │                              │    Node     │
└──────┬──────┘                              └──────┬──────┘
       │                                            │
       │  1. Establish SSH Connection               │
       │ ─────────────────────────────────────────► │
       │                                            │
       │  2. Create temp directory                  │
       │ ─────────────────────────────────────────► │
       │                                            │
       │  3. Copy module code (Python)              │
       │ ─────────────────────────────────────────► │
       │                                            │
       │  4. Execute module                         │
       │ ─────────────────────────────────────────► │
       │                                            │
       │  5. Return JSON results                    │
       │ ◄───────────────────────────────────────── │
       │                                            │
       │  6. Remove temp directory                  │
       │ ─────────────────────────────────────────► │
       │                                            │
```

### WinRM Communication (Windows)

```
┌─────────────┐                              ┌─────────────┐
│   Control   │                              │   Windows   │
│    Node     │                              │    Node     │
└──────┬──────┘                              └──────┬──────┘
       │                                            │
       │  1. Establish WinRM Connection (HTTPS)     │
       │ ─────────────────────────────────────────► │
       │                                            │
       │  2. Copy PowerShell module                 │
       │ ─────────────────────────────────────────► │
       │                                            │
       │  3. Execute PowerShell script              │
       │ ─────────────────────────────────────────► │
       │                                            │
       │  4. Return JSON results                    │
       │ ◄───────────────────────────────────────── │
       │                                            │
```

---

## Execution Model

### Task Execution Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    EXECUTION FLOW                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Parse Playbook                                          │
│     │                                                       │
│     ▼                                                       │
│  2. Load Inventory                                          │
│     │                                                       │
│     ▼                                                       │
│  3. For each Play:                                          │
│     │                                                       │
│     ├──► 3a. Gather Facts (if enabled)                     │
│     │                                                       │
│     ├──► 3b. Load Variables                                │
│     │        - group_vars                                   │
│     │        - host_vars                                    │
│     │        - play vars                                    │
│     │                                                       │
│     ├──► 3c. For each Task:                                │
│     │        │                                              │
│     │        ├──► Evaluate conditions (when)               │
│     │        ├──► Template variables                        │
│     │        ├──► Execute module on hosts                  │
│     │        └──► Collect results                          │
│     │                                                       │
│     └──► 3d. Run Handlers (if notified)                    │
│                                                             │
│  4. Display Summary                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Execution Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **linear** | Execute task on all hosts before next task | Default, predictable |
| **free** | Each host runs as fast as possible | Speed, independent hosts |
| **debug** | Interactive debugging | Troubleshooting |

```yaml
# Using free strategy
- hosts: all
  strategy: free
  tasks:
    - name: Task 1
      ...
```

---

## Configuration Hierarchy

Ansible looks for configuration in this order:

```
┌─────────────────────────────────────────────────────────┐
│              CONFIGURATION PRECEDENCE                    │
│                  (Highest to Lowest)                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. ANSIBLE_CONFIG (environment variable)               │
│     │                                                   │
│     ▼                                                   │
│  2. ./ansible.cfg (current directory)                   │
│     │                                                   │
│     ▼                                                   │
│  3. ~/.ansible.cfg (home directory)                     │
│     │                                                   │
│     ▼                                                   │
│  4. /etc/ansible/ansible.cfg (system-wide)              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Key Configuration Options

```ini
[defaults]
inventory = ./inventory
remote_user = ansible
private_key_file = ~/.ssh/ansible_key
host_key_checking = False
forks = 10
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
control_path = /tmp/ansible-%%h-%%r
```

---

## Directory Structure

### Recommended Project Layout

```
ansible-project/
├── ansible.cfg              # Configuration
├── inventory/
│   ├── production           # Production inventory
│   ├── staging              # Staging inventory
│   └── group_vars/
│       ├── all.yml          # Variables for all hosts
│       ├── webservers.yml   # Variables for webservers
│       └── databases.yml    # Variables for databases
├── playbooks/
│   ├── site.yml             # Main playbook
│   ├── webservers.yml       # Web server playbook
│   └── databases.yml        # Database playbook
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   ├── files/
│   │   ├── vars/
│   │   ├── defaults/
│   │   └── meta/
│   ├── nginx/
│   └── mysql/
└── files/
    └── scripts/
```

---

## Summary

| Component | Description |
|-----------|-------------|
| **Control Node** | Machine running Ansible commands |
| **Managed Node** | Target machines being configured |
| **Inventory** | List of hosts and groups |
| **Modules** | Units of work (apt, copy, service) |
| **Plugins** | Extend Ansible functionality |
| **Playbooks** | YAML automation definitions |
| **SSH/WinRM** | Communication protocols |
| **ansible.cfg** | Configuration file |

The agentless architecture makes Ansible simple to deploy and maintain, while the modular design allows for extensive customization and scalability.
