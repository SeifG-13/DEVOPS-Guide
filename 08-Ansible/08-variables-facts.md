# Ansible Variables and Facts

## Variable Types

Variables in Ansible can come from many sources with different precedence levels.

```
┌─────────────────────────────────────────────────────────────┐
│                   VARIABLE SOURCES                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │   Inventory     │  │   Playbook      │                  │
│  │   Variables     │  │   Variables     │                  │
│  │                 │  │                 │                  │
│  │ - host_vars/    │  │ - vars:         │                  │
│  │ - group_vars/   │  │ - vars_files:   │                  │
│  │ - inline        │  │ - vars_prompt:  │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │   Command Line  │  │   Role          │                  │
│  │   Variables     │  │   Variables     │                  │
│  │                 │  │                 │                  │
│  │ - -e / --extra  │  │ - defaults/     │                  │
│  │ - @file.yml     │  │ - vars/         │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │   Facts         │  │   Registered    │                  │
│  │                 │  │   Variables     │                  │
│  │ - ansible_*     │  │                 │                  │
│  │ - setup module  │  │ - register:     │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Defining Variables

### In Playbooks

```yaml
---
- name: Example with variables
  hosts: webservers

  # Simple variables
  vars:
    http_port: 80
    app_name: myapp
    max_connections: 1000

    # List
    packages:
      - nginx
      - php-fpm
      - mysql-client

    # Dictionary
    database:
      host: db.example.com
      port: 3306
      name: appdb
      user: appuser

  tasks:
    - name: Use variables
      debug:
        msg: "App {{ app_name }} runs on port {{ http_port }}"
```

### In Variable Files

```yaml
# vars/common.yml
---
ntp_server: ntp.example.com
timezone: UTC
admin_email: admin@example.com

# vars/production.yml
---
environment: production
debug: false
replicas: 3
```

```yaml
# Using vars_files in playbook
---
- name: Load variables from files
  hosts: all
  vars_files:
    - vars/common.yml
    - vars/{{ environment }}.yml
```

### In Inventory

```ini
# Inline host variables
[webservers]
web1 http_port=80 max_clients=500
web2 http_port=8080 max_clients=1000

# Group variables
[webservers:vars]
nginx_version=1.24
document_root=/var/www/html
```

### In host_vars and group_vars

```
inventory/
├── hosts
├── host_vars/
│   ├── web1.yml
│   └── web2.yml
└── group_vars/
    ├── all.yml
    ├── webservers.yml
    └── databases.yml
```

```yaml
# group_vars/all.yml
---
ntp_server: ntp.example.com
dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# group_vars/webservers.yml
---
http_port: 80
nginx_worker_processes: auto

# host_vars/web1.yml
---
primary_server: true
http_port: 8080  # Override group variable
```

---

## Variable Precedence

Ansible has 22 levels of variable precedence (lowest to highest):

```
┌─────────────────────────────────────────────────────────────┐
│           VARIABLE PRECEDENCE (Low → High)                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1.  command line values (not variables)                    │
│  2.  role defaults (defaults/main.yml)                      │
│  3.  inventory file or script group vars                    │
│  4.  inventory group_vars/all                               │
│  5.  playbook group_vars/all                                │
│  6.  inventory group_vars/*                                 │
│  7.  playbook group_vars/*                                  │
│  8.  inventory file or script host vars                     │
│  9.  inventory host_vars/*                                  │
│  10. playbook host_vars/*                                   │
│  11. host facts / cached set_facts                          │
│  12. play vars                                              │
│  13. play vars_prompt                                       │
│  14. play vars_files                                        │
│  15. role vars (vars/main.yml)                              │
│  16. block vars                                             │
│  17. task vars                                              │
│  18. include_vars                                           │
│  19. set_facts / registered vars                            │
│  20. role parameters                                        │
│  21. include parameters                                     │
│  22. extra vars (-e) ← HIGHEST PRIORITY                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Precedence Rules

```yaml
# Role defaults (lowest for roles)
# roles/nginx/defaults/main.yml
nginx_port: 80  # Easy to override

# Role vars (higher precedence)
# roles/nginx/vars/main.yml
nginx_user: www-data  # Harder to override

# Play vars
- hosts: all
  vars:
    nginx_port: 8080  # Overrides role defaults

# Extra vars (highest - always wins)
# ansible-playbook playbook.yml -e "nginx_port=9090"
```

---

## Using Variables

### Basic Syntax

```yaml
tasks:
  # Simple variable
  - name: Print message
    debug:
      msg: "Hello, {{ username }}"

  # In module arguments
  - name: Install package
    apt:
      name: "{{ package_name }}"
      state: present

  # In file paths
  - name: Copy config
    template:
      src: "{{ app_name }}.conf.j2"
      dest: "/etc/{{ app_name }}/config.conf"
```

### Dictionary Access

```yaml
vars:
  database:
    host: db.example.com
    port: 3306
    credentials:
      user: admin
      password: secret

tasks:
  # Dot notation
  - debug:
      msg: "Host: {{ database.host }}"

  # Bracket notation (safer for special characters)
  - debug:
      msg: "Port: {{ database['port'] }}"

  # Nested access
  - debug:
      msg: "User: {{ database.credentials.user }}"
```

### List Access

```yaml
vars:
  servers:
    - web1.example.com
    - web2.example.com
    - web3.example.com

tasks:
  # Access by index
  - debug:
      msg: "First server: {{ servers[0] }}"

  # Last item
  - debug:
      msg: "Last server: {{ servers[-1] }}"

  # Loop through
  - debug:
      msg: "Server: {{ item }}"
    loop: "{{ servers }}"
```

---

## Registering Variables

Capture task output for later use.

```yaml
tasks:
  # Register command output
  - name: Get disk space
    command: df -h /
    register: disk_space

  - name: Show disk space
    debug:
      var: disk_space

  # Access specific parts
  - name: Show stdout
    debug:
      msg: "{{ disk_space.stdout }}"

  # Use in conditions
  - name: Alert if low space
    debug:
      msg: "Low disk space!"
    when: "'90%' in disk_space.stdout"
```

### Common Register Attributes

```yaml
# After running a command
register: result

# Available attributes:
# result.changed      - whether task changed anything
# result.failed       - whether task failed
# result.rc           - return code (commands)
# result.stdout       - standard output
# result.stdout_lines - stdout as list
# result.stderr       - standard error
# result.msg          - message (some modules)
# result.results      - list of results (loops)
```

### Register with Loops

```yaml
- name: Check services
  service:
    name: "{{ item }}"
  loop:
    - nginx
    - mysql
    - redis
  register: service_status

- name: Show all results
  debug:
    msg: "{{ item.name }} is {{ item.state }}"
  loop: "{{ service_status.results }}"
```

---

## Ansible Facts

Facts are system information gathered from managed nodes.

### Gathering Facts

```yaml
---
- name: Work with facts
  hosts: all
  gather_facts: yes  # Default is yes

  tasks:
    - name: Show all facts
      debug:
        var: ansible_facts

    - name: Show OS family
      debug:
        msg: "OS: {{ ansible_facts['os_family'] }}"
```

### Common Facts

```yaml
# Operating System
ansible_distribution        # Ubuntu, CentOS, etc.
ansible_distribution_version  # 22.04, 8.5, etc.
ansible_os_family          # Debian, RedHat, etc.
ansible_kernel             # Kernel version

# Hardware
ansible_architecture       # x86_64, aarch64
ansible_processor          # CPU info
ansible_processor_cores    # Number of cores
ansible_memtotal_mb       # Total RAM in MB

# Network
ansible_hostname          # Short hostname
ansible_fqdn              # Fully qualified name
ansible_default_ipv4      # Default IPv4 info
ansible_all_ipv4_addresses  # All IPv4 addresses
ansible_interfaces        # Network interfaces

# Storage
ansible_mounts            # Mounted filesystems
ansible_devices           # Block devices

# Environment
ansible_env              # Environment variables
ansible_user_id          # Current user
ansible_user_dir         # Home directory
```

### Using Facts

```yaml
tasks:
  # Conditional based on OS
  - name: Install on Debian
    apt:
      name: nginx
    when: ansible_os_family == "Debian"

  - name: Install on RedHat
    yum:
      name: nginx
    when: ansible_os_family == "RedHat"

  # Use IP address
  - name: Configure app
    template:
      src: app.conf.j2
      dest: /etc/app.conf
    vars:
      server_ip: "{{ ansible_default_ipv4.address }}"

  # Memory-based configuration
  - name: Set Java heap
    lineinfile:
      path: /etc/app/jvm.conf
      line: "HEAP_SIZE={{ (ansible_memtotal_mb * 0.75) | int }}m"
```

### Disabling Fact Gathering

```yaml
# Disable for faster execution
- hosts: all
  gather_facts: no

  tasks:
    - name: Quick task
      command: uptime

# Gather specific facts only
- hosts: all
  gather_facts: no

  tasks:
    - name: Gather only network facts
      setup:
        gather_subset:
          - network

    - name: Gather minimal facts
      setup:
        gather_subset:
          - "!all"
          - "!min"
          - network
```

---

## Custom Facts

### Local Facts (facts.d)

Create custom facts on managed nodes.

```bash
# On managed node, create /etc/ansible/facts.d/custom.fact
# Must be executable or JSON/INI file
```

```ini
# /etc/ansible/facts.d/custom.fact (INI format)
[application]
name=myapp
version=1.2.3
environment=production
```

```json
// /etc/ansible/facts.d/custom.json
{
  "application": {
    "name": "myapp",
    "version": "1.2.3"
  }
}
```

```yaml
# Access custom facts
- debug:
    var: ansible_local.custom.application.name
```

### Set Fact Module

```yaml
tasks:
  # Set a fact during play
  - name: Calculate value
    set_fact:
      app_memory: "{{ (ansible_memtotal_mb * 0.5) | int }}"
      is_production: "{{ inventory_hostname is match('prod-.*') }}"

  - name: Use calculated fact
    debug:
      msg: "Memory allocation: {{ app_memory }}MB"

  # Cacheable facts (persist across plays)
  - name: Set persistent fact
    set_fact:
      deployment_time: "{{ ansible_date_time.iso8601 }}"
      cacheable: yes
```

---

## Magic Variables

Built-in variables always available.

```yaml
# Inventory variables
inventory_hostname      # Current host name (from inventory)
inventory_hostname_short  # Short hostname
ansible_host           # Actual host to connect to
groups                 # All groups and their hosts
group_names           # Groups current host belongs to

# Play variables
ansible_play_hosts    # All hosts in current play
ansible_play_batch    # Current batch of hosts
play_hosts           # Deprecated, use ansible_play_hosts

# Special variables
hostvars              # Variables for all hosts
ansible_version       # Ansible version info
role_path            # Current role path
playbook_dir         # Playbook directory
```

### Using hostvars

```yaml
tasks:
  # Access another host's variables
  - name: Get DB host IP
    debug:
      msg: "DB IP: {{ hostvars['db1.example.com']['ansible_default_ipv4']['address'] }}"

  # Access another host's facts
  - name: Configure app with DB info
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    vars:
      db_host: "{{ hostvars[groups['databases'][0]]['ansible_host'] }}"
```

### Using groups

```yaml
tasks:
  # List all webservers
  - name: Show webservers
    debug:
      msg: "{{ groups['webservers'] }}"

  # Get first database server
  - name: Primary DB
    debug:
      msg: "{{ groups['databases'][0] }}"

  # Check group membership
  - name: Web-specific task
    command: web_task.sh
    when: "'webservers' in group_names"
```

---

## Variable Prompts

Interactive variable input.

```yaml
---
- name: Interactive playbook
  hosts: all

  vars_prompt:
    - name: username
      prompt: "Enter username"
      private: no  # Show input

    - name: password
      prompt: "Enter password"
      private: yes  # Hide input

    - name: environment
      prompt: "Deployment environment"
      default: "staging"
      private: no

    - name: confirm
      prompt: "Type 'yes' to continue"
      private: no

  tasks:
    - name: Proceed if confirmed
      debug:
        msg: "Deploying as {{ username }} to {{ environment }}"
      when: confirm == "yes"
```

---

## include_vars Module

Dynamically load variables from files.

```yaml
tasks:
  # Load from file
  - name: Load common variables
    include_vars:
      file: vars/common.yml

  # Load based on OS
  - name: Load OS-specific vars
    include_vars:
      file: "vars/{{ ansible_os_family }}.yml"

  # Load all files from directory
  - name: Load all config vars
    include_vars:
      dir: vars/configs
      extensions:
        - yml
        - yaml

  # Load with name
  - name: Load and name the vars
    include_vars:
      file: vars/app.yml
      name: app_config

  - debug:
      var: app_config
```

---

## Variable Filters

Transform variables using Jinja2 filters.

```yaml
vars:
  name: "  John Doe  "
  items: [3, 1, 4, 1, 5, 9, 2, 6]
  data: {"key": "value"}

tasks:
  # String filters
  - debug:
      msg: "{{ name | trim }}"            # "John Doe"
  - debug:
      msg: "{{ name | lower }}"           # "john doe"
  - debug:
      msg: "{{ name | upper }}"           # "JOHN DOE"
  - debug:
      msg: "{{ name | replace(' ', '-') }}"  # "John-Doe"

  # List filters
  - debug:
      msg: "{{ items | sort }}"           # [1, 1, 2, 3, 4, 5, 6, 9]
  - debug:
      msg: "{{ items | unique }}"         # [3, 1, 4, 5, 9, 2, 6]
  - debug:
      msg: "{{ items | min }}"            # 1
  - debug:
      msg: "{{ items | max }}"            # 9
  - debug:
      msg: "{{ items | length }}"         # 8

  # Data filters
  - debug:
      msg: "{{ data | to_json }}"         # JSON string
  - debug:
      msg: "{{ data | to_yaml }}"         # YAML string

  # Default value
  - debug:
      msg: "{{ undefined_var | default('fallback') }}"
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **vars** | Variables defined in playbook |
| **vars_files** | Load variables from files |
| **host_vars/** | Host-specific variables |
| **group_vars/** | Group-specific variables |
| **Facts** | System info gathered by setup |
| **set_fact** | Define facts during play |
| **register** | Capture task output |
| **hostvars** | Access other hosts' variables |
| **include_vars** | Dynamically load variables |
| **-e / --extra-vars** | Command-line variables (highest priority) |

### Best Practices

1. Use `group_vars/all.yml` for common variables
2. Use `host_vars/` for host-specific overrides
3. Use role `defaults/` for easy-to-override values
4. Use `-e` for one-time overrides
5. Keep sensitive data in Ansible Vault
