# Ansible Inventory

## What is Inventory?

Inventory is the collection of hosts (nodes) that Ansible manages. It defines:
- Which hosts to target
- How to organize hosts into groups
- Variables specific to hosts or groups
- Connection parameters

---

## Static Inventory

### INI Format

The default and most common format.

```ini
# Basic inventory file: hosts or inventory

# Ungrouped hosts
server1.example.com
192.168.1.100

# Grouped hosts
[webservers]
web1.example.com
web2.example.com
web3.example.com

[databases]
db1.example.com
db2.example.com

[loadbalancers]
lb1.example.com
```

### YAML Format

More readable for complex inventories.

```yaml
# inventory.yml
all:
  hosts:
    server1.example.com:
    192.168.1.100:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
        web3.example.com:
    databases:
      hosts:
        db1.example.com:
        db2.example.com:
    loadbalancers:
      hosts:
        lb1.example.com:
```

---

## Host Definitions

### Basic Host Entry

```ini
# Just hostname
web1.example.com

# With IP address alias
web1 ansible_host=192.168.1.10

# With multiple parameters
web1 ansible_host=192.168.1.10 ansible_port=2222 ansible_user=admin
```

### Host Ranges

```ini
# Numeric range
web[1:5].example.com    # web1, web2, web3, web4, web5

# Alphabetic range
db-[a:c].example.com    # db-a, db-b, db-c

# With padding
web[01:50].example.com  # web01, web02, ... web50

# Step pattern
web[0:10:2].example.com # web0, web2, web4, web6, web8, web10
```

### Connection Parameters

```ini
[webservers]
# Linux host via SSH
web1 ansible_host=192.168.1.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/web_key

# Windows host via WinRM
win1 ansible_host=192.168.1.20 ansible_user=Administrator ansible_password=SecurePass ansible_connection=winrm

# Local connection
localhost ansible_connection=local

# Docker container
container1 ansible_connection=docker ansible_host=my_container

# Different SSH port
web2 ansible_host=192.168.1.11 ansible_port=2222
```

---

## Host Groups

### Basic Groups

```ini
[webservers]
web1.example.com
web2.example.com

[databases]
db1.example.com
db2.example.com

[monitoring]
prometheus.example.com
grafana.example.com
```

### Nested Groups (Children)

```ini
[webservers]
web1.example.com
web2.example.com

[databases]
db1.example.com

[atlanta]
web1.example.com
db1.example.com

[boston]
web2.example.com

# Parent group with children
[usa:children]
atlanta
boston

# Production includes web and db
[production:children]
webservers
databases
```

### YAML Nested Groups

```yaml
all:
  children:
    usa:
      children:
        atlanta:
          hosts:
            web1.example.com:
            db1.example.com:
        boston:
          hosts:
            web2.example.com:
    production:
      children:
        webservers:
          hosts:
            web1.example.com:
            web2.example.com:
        databases:
          hosts:
            db1.example.com:
```

---

## Host Variables

Variables assigned to specific hosts.

### Inline Variables (INI)

```ini
[webservers]
web1 ansible_host=192.168.1.10 http_port=80 max_connections=500
web2 ansible_host=192.168.1.11 http_port=8080 max_connections=1000
```

### YAML Host Variables

```yaml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.10
          http_port: 80
          max_connections: 500
        web2:
          ansible_host: 192.168.1.11
          http_port: 8080
          max_connections: 1000
```

### host_vars Directory

Create separate files for each host.

```
inventory/
├── hosts
└── host_vars/
    ├── web1.yml
    └── web2.yml
```

```yaml
# host_vars/web1.yml
---
ansible_host: 192.168.1.10
http_port: 80
max_connections: 500
ssl_enabled: true
ssl_certificate: /etc/ssl/web1.crt
```

---

## Group Variables

Variables assigned to all hosts in a group.

### Inline Variables (INI)

```ini
[webservers]
web1.example.com
web2.example.com

[webservers:vars]
http_port=80
nginx_version=1.24
document_root=/var/www/html
```

### YAML Group Variables

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
      vars:
        http_port: 80
        nginx_version: "1.24"
        document_root: /var/www/html
```

### group_vars Directory

```
inventory/
├── hosts
├── group_vars/
│   ├── all.yml           # Variables for all hosts
│   ├── webservers.yml    # Variables for webservers group
│   └── databases.yml     # Variables for databases group
└── host_vars/
    └── web1.yml
```

```yaml
# group_vars/all.yml
---
ansible_user: ubuntu
ansible_ssh_private_key_file: ~/.ssh/ansible_key
ntp_server: ntp.example.com
dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# group_vars/webservers.yml
---
http_port: 80
nginx_version: "1.24"
php_version: "8.2"
```

---

## Inventory Patterns

Patterns for targeting hosts in commands.

### Basic Patterns

```bash
# All hosts
ansible all -m ping

# Specific group
ansible webservers -m ping

# Specific host
ansible web1.example.com -m ping

# Multiple groups
ansible webservers:databases -m ping

# Intersection (hosts in both groups)
ansible 'webservers:&production' -m ping

# Exclusion (webservers except those in maintenance)
ansible 'webservers:!maintenance' -m ping
```

### Pattern Examples

| Pattern | Description |
|---------|-------------|
| `all` | All hosts |
| `*` | All hosts |
| `webservers` | All hosts in webservers group |
| `web*.example.com` | Wildcard match |
| `web[1:3].example.com` | Range match |
| `webservers:databases` | Union (OR) |
| `webservers:&staging` | Intersection (AND) |
| `webservers:!maintenance` | Exclusion (NOT) |
| `~web[0-9]+` | Regex match |

### Complex Pattern Examples

```bash
# Hosts in webservers OR databases
ansible 'webservers:databases' -m ping

# Hosts in webservers AND in atlanta
ansible 'webservers:&atlanta' -m ping

# Hosts in webservers but NOT in maintenance
ansible 'webservers:!maintenance' -m ping

# Complex: (webservers or databases) and usa, excluding maintenance
ansible '(webservers:databases):&usa:!maintenance' -m ping

# Regex: hosts starting with web and ending with digit
ansible '~web.*[0-9]$' -m ping

# First host in webservers group
ansible 'webservers[0]' -m ping

# First two hosts
ansible 'webservers[0:1]' -m ping
```

---

## Dynamic Inventory

Automatically generated inventory from external sources.

### AWS EC2 Dynamic Inventory

```yaml
# aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2

# Filter instances
filters:
  tag:Environment: production
  instance-state-name: running

# Group by tags
keyed_groups:
  - key: tags.Application
    prefix: app
  - key: tags.Environment
    prefix: env
  - key: instance_type
    prefix: type

# Host variables from EC2
hostnames:
  - private-ip-address
compose:
  ansible_host: private_ip_address
```

```bash
# Test dynamic inventory
ansible-inventory -i aws_ec2.yml --graph
ansible-inventory -i aws_ec2.yml --list
```

### Azure Dynamic Inventory

```yaml
# azure_rm.yml
plugin: azure.azcollection.azure_rm
auth_source: auto

include_vm_resource_groups:
  - production-rg
  - staging-rg

keyed_groups:
  - prefix: tag
    key: tags
  - prefix: location
    key: location

hostnames:
  - private_ip_address

conditional_groups:
  webservers: "'web' in tags.Role"
  databases: "'db' in tags.Role"
```

### Custom Dynamic Inventory Script

```python
#!/usr/bin/env python3
# dynamic_inventory.py

import json
import argparse

def get_inventory():
    return {
        "webservers": {
            "hosts": ["web1.example.com", "web2.example.com"],
            "vars": {
                "http_port": 80
            }
        },
        "databases": {
            "hosts": ["db1.example.com"]
        },
        "_meta": {
            "hostvars": {
                "web1.example.com": {
                    "ansible_host": "192.168.1.10"
                },
                "web2.example.com": {
                    "ansible_host": "192.168.1.11"
                },
                "db1.example.com": {
                    "ansible_host": "192.168.1.20"
                }
            }
        }
    }

def get_host(hostname):
    inventory = get_inventory()
    return inventory.get("_meta", {}).get("hostvars", {}).get(hostname, {})

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true")
    parser.add_argument("--host", type=str)
    args = parser.parse_args()

    if args.list:
        print(json.dumps(get_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host(args.host), indent=2))
```

```bash
# Make executable
chmod +x dynamic_inventory.py

# Test
./dynamic_inventory.py --list
ansible -i dynamic_inventory.py all -m ping
```

---

## Multiple Inventories

### Directory-Based Inventory

```
inventory/
├── production/
│   ├── hosts
│   ├── group_vars/
│   │   └── all.yml
│   └── host_vars/
│       └── web1.yml
├── staging/
│   ├── hosts
│   └── group_vars/
│       └── all.yml
└── development/
    └── hosts
```

```bash
# Use specific inventory
ansible-playbook -i inventory/production playbook.yml

# Use multiple inventories
ansible-playbook -i inventory/production -i inventory/staging playbook.yml

# Merge inventories with directory
ansible-playbook -i inventory/ playbook.yml
```

### Environment-Specific Variables

```yaml
# inventory/production/group_vars/all.yml
---
environment: production
debug_mode: false
log_level: warn
replicas: 3

# inventory/staging/group_vars/all.yml
---
environment: staging
debug_mode: true
log_level: debug
replicas: 1
```

---

## Inventory Plugins

Built-in plugins for various sources.

| Plugin | Source |
|--------|--------|
| `amazon.aws.aws_ec2` | AWS EC2 instances |
| `azure.azcollection.azure_rm` | Azure VMs |
| `google.cloud.gcp_compute` | GCP instances |
| `kubernetes.core.k8s` | Kubernetes pods |
| `vmware.vmware_rest.vmware_vm_inventory` | VMware VMs |
| `community.docker.docker_containers` | Docker containers |
| `constructed` | Build groups from existing inventory |

### Constructed Inventory

Create groups based on variables.

```yaml
# constructed.yml
plugin: constructed
strict: False

# Create groups based on variables
groups:
  # Group by OS family
  debian: ansible_facts.os_family == 'Debian'
  redhat: ansible_facts.os_family == 'RedHat'

  # Group by custom variable
  production_web: inventory_hostname.startswith('prod') and 'webservers' in group_names

# Add composed variables
compose:
  ansible_connection: "'local' if inventory_hostname == 'localhost' else 'ssh'"
```

---

## Inventory Commands

### View and Debug Inventory

```bash
# List all hosts
ansible-inventory -i inventory --list

# Show inventory as graph
ansible-inventory -i inventory --graph

# Show graph with variables
ansible-inventory -i inventory --graph --vars

# Show specific host details
ansible-inventory -i inventory --host web1.example.com

# Validate inventory YAML
ansible-inventory -i inventory --list --yaml
```

### Example Output

```bash
$ ansible-inventory -i inventory --graph
@all:
  |--@ungrouped:
  |--@webservers:
  |  |--web1.example.com
  |  |--web2.example.com
  |--@databases:
  |  |--db1.example.com
  |--@production:
  |  |--@webservers:
  |  |--@databases:
```

---

## Best Practices

### 1. Organize by Environment

```
inventories/
├── production/
├── staging/
└── development/
```

### 2. Use group_vars and host_vars

```
# Instead of inline variables
[webservers:vars]
http_port=80

# Use files
group_vars/webservers.yml
```

### 3. Meaningful Group Names

```ini
# Good
[frontend_servers]
[backend_api]
[cache_servers]

# Avoid
[group1]
[servers]
[misc]
```

### 4. Document Your Inventory

```ini
# Production Web Servers
# Region: US-East
# Contact: webteam@example.com
[webservers]
web1.example.com  # Primary
web2.example.com  # Secondary
```

### 5. Use Aliases for Clarity

```ini
[webservers]
primary_web   ansible_host=192.168.1.10
secondary_web ansible_host=192.168.1.11
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Static Inventory** | INI or YAML files with host definitions |
| **Dynamic Inventory** | Scripts/plugins that query external sources |
| **Groups** | Logical organization of hosts |
| **Host Variables** | Variables specific to a host |
| **Group Variables** | Variables for all hosts in a group |
| **Patterns** | Target specific hosts/groups |
| **Nested Groups** | Groups containing other groups |
| **host_vars/** | Directory for host-specific variables |
| **group_vars/** | Directory for group-specific variables |

A well-organized inventory is the foundation of effective Ansible automation.
