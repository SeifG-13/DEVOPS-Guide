# Ansible Playbooks Basics

## What is a Playbook?

A playbook is a YAML file containing one or more plays that define automation tasks. Playbooks are the foundation of Ansible automation.

```
┌─────────────────────────────────────────┐
│              PLAYBOOK                    │
├─────────────────────────────────────────┤
│                                         │
│  ┌───────────────────────────────────┐ │
│  │            PLAY 1                 │ │
│  │  hosts: webservers                │ │
│  │  ┌─────────────────────────────┐ │ │
│  │  │  Task 1: Install nginx      │ │ │
│  │  │  Task 2: Copy config        │ │ │
│  │  │  Task 3: Start service      │ │ │
│  │  └─────────────────────────────┘ │ │
│  └───────────────────────────────────┘ │
│                                         │
│  ┌───────────────────────────────────┐ │
│  │            PLAY 2                 │ │
│  │  hosts: databases                 │ │
│  │  ┌─────────────────────────────┐ │ │
│  │  │  Task 1: Install MySQL      │ │ │
│  │  │  Task 2: Configure DB       │ │ │
│  │  └─────────────────────────────┘ │ │
│  └───────────────────────────────────┘ │
│                                         │
└─────────────────────────────────────────┘
```

---

## YAML Basics

### Syntax Rules

```yaml
# Comments start with #

# Key-value pairs
name: webserver
port: 80

# Lists (two styles)
packages:
  - nginx
  - php
  - mysql

# Or inline
packages: [nginx, php, mysql]

# Dictionaries (nested)
server:
  name: web1
  ip: 192.168.1.10
  port: 80

# Or inline
server: {name: web1, ip: 192.168.1.10, port: 80}

# Multi-line strings
description: |
  This is a multi-line
  string that preserves
  line breaks.

# Folded string (newlines become spaces)
description: >
  This is a long string
  that will be folded
  into a single line.

# Boolean values
enabled: true
debug: false
active: yes
disabled: no
```

### Common YAML Mistakes

```yaml
# BAD - inconsistent indentation
tasks:
  - name: Task 1
     command: echo "wrong"  # Extra space!

# GOOD
tasks:
  - name: Task 1
    command: echo "correct"

# BAD - missing quotes for special characters
message: This has a : colon  # Breaks parsing

# GOOD
message: "This has a : colon"

# BAD - tabs instead of spaces
tasks:
	- name: Using tab  # Use spaces only!

# GOOD
tasks:
  - name: Using spaces
```

---

## Playbook Structure

### Minimal Playbook

```yaml
---
- name: My First Playbook
  hosts: all
  tasks:
    - name: Print message
      debug:
        msg: "Hello, Ansible!"
```

### Complete Playbook Structure

```yaml
---
# Playbook with all common elements
- name: Configure Web Servers
  hosts: webservers
  become: yes
  gather_facts: yes

  vars:
    http_port: 80
    app_name: myapp

  vars_files:
    - vars/common.yml

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes

  post_tasks:
    - name: Verify nginx is running
      command: systemctl is-active nginx
      changed_when: false

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

---

## Running Playbooks

### Basic Execution

```bash
# Run playbook
ansible-playbook playbook.yml

# Specify inventory
ansible-playbook -i inventory playbook.yml

# Limit to specific hosts
ansible-playbook playbook.yml --limit webservers
ansible-playbook playbook.yml --limit web1.example.com

# Dry run (check mode)
ansible-playbook playbook.yml --check

# Show differences
ansible-playbook playbook.yml --check --diff

# Syntax check only
ansible-playbook playbook.yml --syntax-check

# List tasks
ansible-playbook playbook.yml --list-tasks

# List hosts
ansible-playbook playbook.yml --list-hosts

# List tags
ansible-playbook playbook.yml --list-tags
```

### Execution Options

```bash
# Verbose output
ansible-playbook playbook.yml -v
ansible-playbook playbook.yml -vv
ansible-playbook playbook.yml -vvv

# Ask for sudo password
ansible-playbook playbook.yml -K

# Ask for SSH password
ansible-playbook playbook.yml -k

# Extra variables
ansible-playbook playbook.yml -e "version=1.2.3"
ansible-playbook playbook.yml -e "@vars.json"

# Start at specific task
ansible-playbook playbook.yml --start-at-task="Install nginx"

# Step through tasks
ansible-playbook playbook.yml --step

# Run specific tags
ansible-playbook playbook.yml --tags "install,configure"

# Skip tags
ansible-playbook playbook.yml --skip-tags "test"
```

---

## Play Keywords

### Host Selection

```yaml
---
# Target all hosts
- hosts: all

# Target specific group
- hosts: webservers

# Target multiple groups
- hosts: webservers:databases

# Target specific host
- hosts: web1.example.com

# Exclude hosts
- hosts: webservers:!maintenance

# Intersection
- hosts: webservers:&production
```

### Common Play Keywords

```yaml
---
- name: Play Name                    # Descriptive name
  hosts: webservers                  # Target hosts
  become: yes                        # Use sudo
  become_user: root                  # Sudo to which user
  become_method: sudo                # How to become
  gather_facts: yes                  # Collect system facts
  connection: ssh                    # Connection type
  remote_user: ubuntu                # SSH user
  serial: 2                          # Rolling updates
  max_fail_percentage: 25            # Failure threshold
  any_errors_fatal: true             # Stop on any error
  ignore_errors: no                  # Don't ignore errors
  ignore_unreachable: no             # Don't ignore unreachable
  order: sorted                      # Host order
  strategy: linear                   # Execution strategy
  environment:                       # Environment variables
    PATH: /opt/bin:{{ ansible_env.PATH }}
```

---

## Task Basics

### Task Structure

```yaml
tasks:
  # Minimal task
  - apt:
      name: nginx
      state: present

  # Task with name (recommended)
  - name: Install nginx
    apt:
      name: nginx
      state: present

  # Task with multiple parameters
  - name: Copy configuration file
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
      owner: root
      group: root
      mode: '0644'
      backup: yes
```

### Task Keywords

```yaml
tasks:
  - name: Install packages
    apt:
      name: "{{ item }}"
      state: present
    loop:                            # Loop over items
      - nginx
      - php
    become: yes                      # Sudo for this task
    when: ansible_os_family == "Debian"  # Conditional
    register: install_result         # Save result
    ignore_errors: yes               # Continue on error
    changed_when: false              # Never report changed
    failed_when: "'error' in result.stderr"  # Custom failure
    notify: Restart nginx            # Trigger handler
    tags:                            # Tags for selective run
      - packages
      - install
    delegate_to: localhost           # Run on different host
    run_once: true                   # Run only once
    retries: 3                       # Retry count
    delay: 5                         # Delay between retries
    until: result is success         # Retry condition
```

---

## Basic Modules

### Package Management

```yaml
# APT (Debian/Ubuntu)
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Install multiple packages
  apt:
    name:
      - nginx
      - php-fpm
      - mysql-server
    state: present

- name: Remove package
  apt:
    name: apache2
    state: absent
    purge: yes

# YUM/DNF (RHEL/CentOS)
- name: Install httpd
  yum:
    name: httpd
    state: present

- name: Update all packages
  yum:
    name: "*"
    state: latest
```

### File Management

```yaml
# Copy file
- name: Copy configuration
  copy:
    src: app.conf
    dest: /etc/app/app.conf
    owner: root
    group: root
    mode: '0644'

# Copy content directly
- name: Create config from content
  copy:
    content: |
      server {
        listen 80;
        server_name example.com;
      }
    dest: /etc/nginx/sites-available/example.conf

# Create directory
- name: Create app directory
  file:
    path: /opt/app
    state: directory
    owner: app
    group: app
    mode: '0755'

# Create symlink
- name: Enable site
  file:
    src: /etc/nginx/sites-available/example.conf
    dest: /etc/nginx/sites-enabled/example.conf
    state: link

# Delete file
- name: Remove old config
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
```

### Service Management

```yaml
# Start and enable service
- name: Start nginx
  service:
    name: nginx
    state: started
    enabled: yes

# Restart service
- name: Restart nginx
  service:
    name: nginx
    state: restarted

# Reload configuration
- name: Reload nginx
  service:
    name: nginx
    state: reloaded

# Using systemd module
- name: Ensure nginx is running
  systemd:
    name: nginx
    state: started
    enabled: yes
    daemon_reload: yes
```

### User Management

```yaml
# Create user
- name: Create deploy user
  user:
    name: deploy
    shell: /bin/bash
    groups: sudo,docker
    append: yes
    create_home: yes

# Create user with SSH key
- name: Create user with key
  user:
    name: deploy
    generate_ssh_key: yes
    ssh_key_bits: 4096

# Remove user
- name: Remove old user
  user:
    name: olduser
    state: absent
    remove: yes
```

### Command Execution

```yaml
# Command module (no shell)
- name: Check disk space
  command: df -h /
  register: disk_space
  changed_when: false

# Shell module (with shell features)
- name: Get running processes
  shell: ps aux | grep nginx | wc -l
  register: nginx_processes

# Script module
- name: Run custom script
  script: scripts/setup.sh

# Raw module (no Python required)
- name: Install Python
  raw: apt-get install -y python3
```

---

## Check Mode (Dry Run)

### Using Check Mode

```bash
# Run in check mode
ansible-playbook playbook.yml --check

# Check mode with diff
ansible-playbook playbook.yml --check --diff
```

### Check Mode in Tasks

```yaml
# Always run in check mode
- name: Verify configuration
  command: nginx -t
  check_mode: yes

# Never run in check mode
- name: This always runs
  command: echo "Always executed"
  check_mode: no

# Task that works in check mode
- name: Check if file exists
  stat:
    path: /etc/nginx/nginx.conf
  register: config_file
  # stat module supports check mode naturally
```

---

## Diff Mode

### Showing Changes

```bash
# Show what would change
ansible-playbook playbook.yml --diff

# Combined with check
ansible-playbook playbook.yml --check --diff
```

### Diff Output Example

```
TASK [Update nginx config] *****
--- before: /etc/nginx/nginx.conf
+++ after: /tmp/ansible-tmp/nginx.conf
@@ -1,5 +1,5 @@
 worker_processes auto;
-worker_connections 768;
+worker_connections 1024;

changed: [web1.example.com]
```

---

## Complete Examples

### Web Server Setup

```yaml
---
- name: Setup Web Server
  hosts: webservers
  become: yes
  gather_facts: yes

  vars:
    nginx_port: 80
    document_root: /var/www/html

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Create document root
      file:
        path: "{{ document_root }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Copy index.html
      copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head><title>Welcome</title></head>
          <body><h1>Hello from {{ inventory_hostname }}</h1></body>
          </html>
        dest: "{{ document_root }}/index.html"
        owner: www-data
        group: www-data

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes
```

### Database Server Setup

```yaml
---
- name: Setup MySQL Server
  hosts: databases
  become: yes

  vars:
    mysql_root_password: "SecurePassword123"

  tasks:
    - name: Install MySQL
      apt:
        name:
          - mysql-server
          - python3-mysqldb
        state: present
        update_cache: yes

    - name: Start MySQL
      service:
        name: mysql
        state: started
        enabled: yes

    - name: Set root password
      mysql_user:
        name: root
        password: "{{ mysql_root_password }}"
        host_all: yes
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Remove anonymous users
      mysql_user:
        name: ''
        host_all: yes
        state: absent
        login_user: root
        login_password: "{{ mysql_root_password }}"

    - name: Create application database
      mysql_db:
        name: appdb
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
```

### Multi-Play Playbook

```yaml
---
# Play 1: Common setup for all servers
- name: Common Setup
  hosts: all
  become: yes
  tasks:
    - name: Update packages
      apt:
        upgrade: dist
        update_cache: yes

    - name: Install common tools
      apt:
        name:
          - vim
          - curl
          - wget
          - htop
        state: present

# Play 2: Web servers
- name: Configure Web Servers
  hosts: webservers
  become: yes
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

# Play 3: Database servers
- name: Configure Database Servers
  hosts: databases
  become: yes
  tasks:
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Playbook** | YAML file containing plays |
| **Play** | Maps hosts to tasks |
| **Task** | Single action using a module |
| **Module** | Unit of work (apt, copy, service) |
| **become** | Privilege escalation (sudo) |
| **gather_facts** | Collect system information |
| **Check mode** | Dry run without changes |
| **Diff mode** | Show what would change |

### Best Practices

1. Always name your plays and tasks
2. Use `--check` before running for real
3. Start simple, add complexity gradually
4. Use meaningful variable names
5. Keep playbooks focused and modular
