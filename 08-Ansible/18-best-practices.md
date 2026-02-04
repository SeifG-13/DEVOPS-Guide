# Ansible Best Practices

## Directory Layout

### Recommended Project Structure

```
ansible-project/
├── ansible.cfg                 # Project configuration
├── requirements.yml            # Role/collection dependencies
├── site.yml                    # Master playbook
├── webservers.yml              # Playbook for web tier
├── dbservers.yml               # Playbook for database tier
│
├── inventories/
│   ├── production/
│   │   ├── hosts               # Production inventory
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── vars.yml    # Common variables
│   │   │   │   └── vault.yml   # Encrypted secrets
│   │   │   ├── webservers.yml
│   │   │   └── dbservers.yml
│   │   └── host_vars/
│   │       └── web1.example.com.yml
│   │
│   └── staging/
│       ├── hosts
│       └── group_vars/
│           └── all/
│               ├── vars.yml
│               └── vault.yml
│
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── mysql/
│   └── app/
│
├── library/                    # Custom modules
├── filter_plugins/             # Custom filters
├── files/                      # Static files
├── templates/                  # Jinja2 templates
└── vars/                       # Additional variables
```

### Key Principles

1. **Separation of environments** - Different inventory directories
2. **Separation of concerns** - Role per function
3. **Variables close to usage** - group_vars with inventory
4. **Secrets encrypted** - Vault files separate from regular vars

---

## Naming Conventions

### Playbooks

```yaml
# Use descriptive names
site.yml               # Master playbook
webservers.yml         # Role-based playbook
deploy-app.yml         # Action-based playbook
```

### Roles

```bash
# Lowercase, hyphens for spaces
roles/
├── nginx/
├── mysql-server/
├── app-deploy/
└── security-hardening/
```

### Variables

```yaml
# Use lowercase with underscores
http_port: 80
max_connections: 1000
app_database_host: db.example.com

# Prefix role variables with role name
nginx_worker_processes: auto
nginx_worker_connections: 1024

# Prefix vault variables
vault_db_password: secret
vault_api_key: abc123

# Reference vault vars in regular vars
db_password: "{{ vault_db_password }}"
```

### Tasks

```yaml
tasks:
  # Use descriptive, action-oriented names
  - name: Install nginx packages
  - name: Deploy nginx configuration
  - name: Enable and start nginx service
  - name: Open firewall port 80

  # Bad examples:
  - name: nginx              # Too vague
  - name: Step 1             # Not descriptive
  - name: Do stuff           # Meaningless
```

### Handlers

```yaml
handlers:
  # Use verb + service pattern
  - name: Restart nginx
  - name: Reload nginx
  - name: Restart php-fpm
```

---

## Playbook Best Practices

### Always Name Everything

```yaml
---
- name: Configure Web Servers     # Name the play
  hosts: webservers

  tasks:
    - name: Install nginx         # Name every task
      apt:
        name: nginx
        state: present
```

### Use Native YAML Syntax

```yaml
# Good - YAML syntax
- name: Install packages
  apt:
    name:
      - nginx
      - php-fpm
    state: present
    update_cache: yes

# Avoid - key=value syntax
- name: Install packages
  apt: name=nginx state=present update_cache=yes
```

### Keep Playbooks Focused

```yaml
# Good - Single responsibility
# webservers.yml
---
- name: Configure Web Servers
  hosts: webservers
  roles:
    - common
    - nginx
    - php

# dbservers.yml
---
- name: Configure Database Servers
  hosts: dbservers
  roles:
    - common
    - mysql
```

### Use Tags Strategically

```yaml
tasks:
  - name: Install packages
    apt:
      name: nginx
    tags:
      - install
      - packages

  - name: Configure nginx
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags:
      - configure

  - name: Deploy application
    copy:
      src: app/
      dest: /var/www/
    tags:
      - deploy
      - app
```

```bash
# Run only deployment tasks
ansible-playbook site.yml --tags deploy

# Skip installation
ansible-playbook site.yml --skip-tags install
```

---

## Role Best Practices

### Use defaults/ for User Options

```yaml
# roles/nginx/defaults/main.yml
# Easy to override, well-documented
---
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_ssl_enabled: false
```

### Use vars/ for Internal Constants

```yaml
# roles/nginx/vars/main.yml
# Not meant to be overridden
---
nginx_config_path: /etc/nginx
nginx_user: www-data
nginx_group: www-data
```

### Split Large Task Files

```yaml
# roles/nginx/tasks/main.yml
---
- include_tasks: install.yml
  tags: install

- include_tasks: configure.yml
  tags: configure

- include_tasks: ssl.yml
  when: nginx_ssl_enabled
  tags: ssl
```

### Document Your Roles

```markdown
# roles/nginx/README.md

# Nginx Role

Installs and configures nginx web server.

## Requirements

- Ubuntu 20.04+ or Debian 11+

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| nginx_port | 80 | HTTP port |
| nginx_ssl_enabled | false | Enable SSL |

## Example Playbook

```yaml
- hosts: webservers
  roles:
    - role: nginx
      nginx_port: 8080
```
```

---

## Variable Best Practices

### Use Variable Precedence Wisely

```yaml
# 1. defaults/main.yml - Base settings (lowest priority)
nginx_port: 80

# 2. group_vars/all.yml - Organization-wide settings
ntp_server: ntp.example.com

# 3. group_vars/production.yml - Environment settings
environment: production

# 4. host_vars/web1.yml - Host-specific settings
custom_setting: value

# 5. Extra vars - Runtime overrides (highest priority)
# ansible-playbook site.yml -e "nginx_port=8080"
```

### Separate Secrets from Variables

```yaml
# group_vars/production/vault.yml (encrypted)
vault_db_password: supersecret
vault_api_key: abc123

# group_vars/production/vars.yml (plaintext)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
environment: production
```

### Use Defaults to Avoid Undefined

```yaml
# Use default filter
{{ optional_var | default('fallback') }}

# Boolean default
{{ feature_enabled | default(false) | bool }}

# List default
{{ custom_packages | default([]) }}

# Dictionary default
{{ config_options | default({}) }}
```

---

## Security Best Practices

### Ansible Vault

```bash
# Encrypt sensitive files
ansible-vault encrypt group_vars/production/vault.yml

# Use different passwords per environment
ansible-vault encrypt --vault-id prod@prompt secrets_prod.yml
ansible-vault encrypt --vault-id dev@prompt secrets_dev.yml
```

### Least Privilege

```yaml
# Only become when necessary
tasks:
  - name: Read config (no sudo needed)
    command: cat /etc/app/config
    become: no

  - name: Install package (needs sudo)
    apt:
      name: nginx
    become: yes
```

### Secure File Permissions

```yaml
- name: Deploy private key
  copy:
    src: private.key
    dest: /etc/ssl/private/app.key
    owner: root
    group: root
    mode: '0600'  # Restrictive permissions
```

### No Sensitive Data in Logs

```yaml
- name: Set database password
  mysql_user:
    name: app
    password: "{{ vault_db_password }}"
  no_log: true  # Don't log this task's output
```

### Validate Before Deploying

```yaml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: 'nginx -t -c %s'  # Validate before replacing
```

---

## Idempotency

### Write Idempotent Tasks

```yaml
# Good - Idempotent
- name: Ensure nginx is installed
  apt:
    name: nginx
    state: present  # Only install if missing

# Good - Idempotent
- name: Ensure service is running
  service:
    name: nginx
    state: started  # Only start if not running

# Bad - Not idempotent
- name: Start nginx
  command: systemctl start nginx
  # Runs every time, may error if already running
```

### Use Proper State Management

```yaml
# Files
- file:
    path: /opt/app
    state: directory  # Creates if missing, no-op if exists

# Packages
- apt:
    name: nginx
    state: present   # Install
    # state: latest  # Upgrade
    # state: absent  # Remove

# Services
- service:
    name: nginx
    state: started   # Start
    # state: stopped # Stop
    # state: restarted # Always restart (not idempotent)
```

### creates/removes for Commands

```yaml
# Only run if file doesn't exist
- name: Initialize database
  command: /opt/app/init_db.sh
  args:
    creates: /opt/app/database.db

# Only run if file exists
- name: Run cleanup
  command: /opt/app/cleanup.sh
  args:
    removes: /tmp/cleanup_needed
```

---

## Performance Best Practices

### Optimize Fact Gathering

```yaml
# Disable if not needed
- hosts: all
  gather_facts: no

# Gather only what you need
- hosts: all
  gather_facts: yes
  gather_subset:
    - hardware
    - network

# Use fact caching
# ansible.cfg
# [defaults]
# fact_caching = jsonfile
# fact_caching_connection = /tmp/facts
# fact_caching_timeout = 86400
```

### Increase Parallelism

```ini
# ansible.cfg
[defaults]
forks = 20  # Run on 20 hosts simultaneously

[ssh_connection]
pipelining = True  # Reduce SSH operations
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### Use Efficient Modules

```yaml
# Good - Single module call
- name: Install packages
  apt:
    name:
      - nginx
      - php
      - mysql-client
    state: present

# Bad - Multiple module calls
- name: Install nginx
  apt: name=nginx
- name: Install php
  apt: name=php
- name: Install mysql-client
  apt: name=mysql-client
```

### Async for Long Tasks

```yaml
- name: Run long backup
  command: /opt/scripts/backup.sh
  async: 3600  # Max runtime
  poll: 0      # Fire and forget

- name: Do other work
  # ...

- name: Check backup status
  async_status:
    jid: "{{ backup_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 60
```

---

## Testing Best Practices

### Use Check Mode

```bash
# Always test before applying
ansible-playbook site.yml --check --diff
```

### Syntax Validation

```bash
# Check syntax
ansible-playbook site.yml --syntax-check

# Lint playbooks
ansible-lint site.yml
```

### Use Molecule for Role Testing

```bash
# Initialize Molecule scenario
molecule init scenario --driver-name docker

# Run tests
molecule test

# Converge only (apply playbook)
molecule converge

# Verify (run tests)
molecule verify
```

### CI/CD Integration

```yaml
# .github/workflows/ansible.yml
name: Ansible CI

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install ansible-lint
        run: pip install ansible-lint
      - name: Run ansible-lint
        run: ansible-lint site.yml

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install dependencies
        run: pip install molecule molecule-docker
      - name: Run Molecule tests
        run: molecule test
```

---

## Version Control Best Practices

### .gitignore

```gitignore
# Sensitive files
*.vault_pass
.vault_pass*
vault_password*

# Runtime files
*.retry
*.pyc
__pycache__/

# Local configs
.ansible.cfg

# Temporary files
*.tmp
*.bak
```

### Git Hooks

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Check for unencrypted vault files
for file in $(git diff --cached --name-only | grep -E 'vault\.yml$'); do
  if ! head -1 "$file" | grep -q '^\$ANSIBLE_VAULT'; then
    echo "ERROR: Unencrypted vault file: $file"
    exit 1
  fi
done

# Run ansible-lint
ansible-lint *.yml roles/*/tasks/*.yml
```

---

## Documentation Best Practices

### Document Variables

```yaml
# group_vars/all.yml
---
# ============================================
# General Settings
# ============================================

# Environment name (development, staging, production)
environment: production

# Admin email for notifications
admin_email: admin@example.com

# ============================================
# Database Settings
# ============================================

# Database hostname
db_host: db.example.com

# Database port (default: 5432)
db_port: 5432
```

### Template Comments

```jinja2
{#
  Template: nginx.conf.j2
  Purpose: Main nginx configuration
  Required vars:
    - nginx_worker_processes
    - nginx_worker_connections
    - nginx_port
#}
# {{ ansible_managed }}
# Do not edit manually
```

---

## Summary Checklist

### Project Setup
- [ ] Use recommended directory structure
- [ ] Separate inventories per environment
- [ ] Use roles for reusable components
- [ ] Keep secrets in encrypted vault files

### Playbooks
- [ ] Name all plays and tasks
- [ ] Use native YAML syntax
- [ ] Implement proper tags
- [ ] Keep playbooks focused

### Variables
- [ ] Use defaults/ for configurable options
- [ ] Prefix role variables
- [ ] Separate vault variables
- [ ] Use defaults to avoid undefined

### Security
- [ ] Encrypt sensitive data with Vault
- [ ] Use no_log for sensitive tasks
- [ ] Validate configurations before deploying
- [ ] Follow least privilege principle

### Quality
- [ ] Write idempotent tasks
- [ ] Test with --check before applying
- [ ] Use ansible-lint
- [ ] Implement CI/CD testing

### Performance
- [ ] Optimize fact gathering
- [ ] Use appropriate forks setting
- [ ] Enable pipelining
- [ ] Use efficient module patterns
