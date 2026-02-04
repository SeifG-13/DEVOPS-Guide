# Ansible Roles

## What are Roles?

Roles are a way to organize playbooks into reusable, shareable components. They provide a structured way to group tasks, variables, files, templates, and handlers.

```
┌─────────────────────────────────────────────────────────────┐
│                      ROLE STRUCTURE                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  roles/nginx/                                               │
│  ├── tasks/           # Main task list                      │
│  │   └── main.yml                                          │
│  ├── handlers/        # Handler definitions                 │
│  │   └── main.yml                                          │
│  ├── templates/       # Jinja2 templates                    │
│  │   └── nginx.conf.j2                                     │
│  ├── files/           # Static files                        │
│  │   └── index.html                                        │
│  ├── vars/            # Role variables (high precedence)    │
│  │   └── main.yml                                          │
│  ├── defaults/        # Default variables (low precedence)  │
│  │   └── main.yml                                          │
│  ├── meta/            # Role metadata and dependencies      │
│  │   └── main.yml                                          │
│  ├── library/         # Custom modules                      │
│  ├── module_utils/    # Module utilities                    │
│  ├── lookup_plugins/  # Custom lookup plugins               │
│  └── README.md        # Documentation                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Creating Roles

### Using ansible-galaxy

```bash
# Create role skeleton
ansible-galaxy init nginx

# Create in specific path
ansible-galaxy init --init-path=./roles nginx

# Result:
# roles/nginx/
# ├── README.md
# ├── defaults/
# │   └── main.yml
# ├── files/
# ├── handlers/
# │   └── main.yml
# ├── meta/
# │   └── main.yml
# ├── tasks/
# │   └── main.yml
# ├── templates/
# ├── tests/
# │   ├── inventory
# │   └── test.yml
# └── vars/
#     └── main.yml
```

### Manual Creation

```bash
mkdir -p roles/nginx/{tasks,handlers,templates,files,vars,defaults,meta}
touch roles/nginx/{tasks,handlers,vars,defaults,meta}/main.yml
```

---

## Role Structure Explained

### tasks/main.yml

The main entry point for role tasks.

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
  notify: Restart nginx

- name: Deploy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
  notify: Reload nginx

- name: Deploy site configuration
  template:
    src: site.conf.j2
    dest: /etc/nginx/sites-available/{{ site_name }}
  notify: Reload nginx

- name: Enable site
  file:
    src: /etc/nginx/sites-available/{{ site_name }}
    dest: /etc/nginx/sites-enabled/{{ site_name }}
    state: link
  notify: Reload nginx

- name: Ensure nginx is running
  service:
    name: nginx
    state: started
    enabled: yes
```

### handlers/main.yml

Handlers specific to the role.

```yaml
# roles/nginx/handlers/main.yml
---
- name: Restart nginx
  service:
    name: nginx
    state: restarted

- name: Reload nginx
  service:
    name: nginx
    state: reloaded

- name: Validate nginx config
  command: nginx -t
  changed_when: false
```

### defaults/main.yml

Default variables (easily overridden).

```yaml
# roles/nginx/defaults/main.yml
---
# These can be overridden by inventory or playbook variables

nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024

nginx_sites:
  - name: default
    server_name: localhost
    root: /var/www/html

nginx_ssl_enabled: false
nginx_ssl_certificate: ""
nginx_ssl_certificate_key: ""

nginx_gzip_enabled: true
nginx_gzip_types:
  - text/plain
  - text/css
  - application/json
  - application/javascript
```

### vars/main.yml

Role variables (higher precedence, harder to override).

```yaml
# roles/nginx/vars/main.yml
---
# Internal role variables - not meant to be overridden

nginx_packages:
  - nginx
  - nginx-extras

nginx_config_path: /etc/nginx
nginx_sites_available: /etc/nginx/sites-available
nginx_sites_enabled: /etc/nginx/sites-enabled
```

### templates/

Jinja2 templates.

```jinja2
{# roles/nginx/templates/nginx.conf.j2 #}
# {{ ansible_managed }}

user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

{% if nginx_gzip_enabled %}
    gzip on;
    gzip_types {{ nginx_gzip_types | join(' ') }};
{% endif %}

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### files/

Static files to copy.

```html
<!-- roles/nginx/files/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Welcome</title>
</head>
<body>
    <h1>Server is running!</h1>
</body>
</html>
```

### meta/main.yml

Role metadata and dependencies.

```yaml
# roles/nginx/meta/main.yml
---
galaxy_info:
  author: Your Name
  description: Nginx web server role
  company: Your Company
  license: MIT
  min_ansible_version: "2.10"

  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: Debian
      versions:
        - bullseye
        - bookworm

  galaxy_tags:
    - nginx
    - web
    - server

dependencies:
  # List role dependencies here
  - role: common
  - role: firewall
    vars:
      firewall_allow_ports:
        - 80
        - 443
```

---

## Using Roles

### In Playbooks

```yaml
# Method 1: roles section
---
- name: Configure web servers
  hosts: webservers
  become: yes

  roles:
    - nginx
    - php
    - mysql

# Method 2: With variables
---
- name: Configure web servers
  hosts: webservers
  become: yes

  roles:
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_worker_processes: 4

    - role: php
      vars:
        php_version: "8.1"

# Method 3: With conditions and tags
---
- name: Configure web servers
  hosts: webservers
  become: yes

  roles:
    - role: nginx
      when: "'webservers' in group_names"
      tags: ['web', 'nginx']

    - role: mysql
      when: "'databases' in group_names"
      tags: ['db', 'mysql']
```

### include_role (Dynamic)

```yaml
tasks:
  - name: Include nginx role
    include_role:
      name: nginx
    vars:
      nginx_port: 8080

  - name: Conditionally include role
    include_role:
      name: "{{ role_name }}"
    loop:
      - nginx
      - php
    loop_control:
      loop_var: role_name
    when: install_web_stack | bool
```

### import_role (Static)

```yaml
tasks:
  - name: Import nginx role
    import_role:
      name: nginx
    vars:
      nginx_port: 8080
    tags: nginx

  # Import specific tasks file from role
  - name: Import only install tasks
    import_role:
      name: nginx
      tasks_from: install.yml
```

### Difference: include_role vs import_role

| Feature | include_role | import_role |
|---------|--------------|-------------|
| Processing | Runtime (dynamic) | Parse time (static) |
| Loops | Yes | No |
| Conditionals | Applied to include | Applied to each task |
| Tags | Requires apply keyword | Direct |
| `--list-tasks` | Not shown | Shown |

---

## Role Dependencies

### Defining Dependencies

```yaml
# roles/webapp/meta/main.yml
---
dependencies:
  # Simple dependency
  - role: common

  # Dependency with variables
  - role: nginx
    vars:
      nginx_port: 80

  # Conditional dependency
  - role: mysql
    when: database_type == 'mysql'

  # Dependency from Galaxy
  - src: geerlingguy.docker
    version: "6.0.0"
```

### Dependency Execution

```
┌─────────────────────────────────────────────────────────────┐
│                  DEPENDENCY EXECUTION ORDER                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Playbook calls: webapp                                     │
│       │                                                     │
│       ▼                                                     │
│  1. common (dependency of webapp)                           │
│       │                                                     │
│       ▼                                                     │
│  2. nginx (dependency of webapp)                            │
│       │                                                     │
│       ▼                                                     │
│  3. webapp (the role itself)                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Allow Duplicates

```yaml
# By default, dependencies run only once
# To allow multiple executions:

# roles/nginx/meta/main.yml
---
allow_duplicates: true

dependencies:
  - role: common
```

---

## Role Variables Precedence

```
┌─────────────────────────────────────────────────────────────┐
│              ROLE VARIABLE PRECEDENCE                        │
│                 (Lowest to Highest)                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Role defaults (defaults/main.yml)     ← Lowest          │
│       │                                                     │
│       ▼                                                     │
│  2. Inventory variables                                     │
│       │                                                     │
│       ▼                                                     │
│  3. Playbook vars                                           │
│       │                                                     │
│       ▼                                                     │
│  4. Role vars (vars/main.yml)                               │
│       │                                                     │
│       ▼                                                     │
│  5. Role parameters (in playbook)                           │
│       │                                                     │
│       ▼                                                     │
│  6. Extra vars (-e)                       ← Highest         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Best Practices for Variables

```yaml
# defaults/main.yml - Use for configurable options
---
nginx_port: 80                    # User should change this
nginx_worker_processes: auto       # User might change this
nginx_ssl_enabled: false           # User's choice

# vars/main.yml - Use for internal constants
---
nginx_user: www-data               # Don't want users changing this
nginx_config_path: /etc/nginx      # Platform-specific constant
_nginx_version: "{{ nginx_version | default('stable') }}"  # Computed
```

---

## Organizing Tasks in Roles

### Split Tasks into Files

```yaml
# roles/nginx/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- include_tasks: install.yml
- include_tasks: configure.yml
- include_tasks: ssl.yml
  when: nginx_ssl_enabled
- include_tasks: sites.yml
```

```yaml
# roles/nginx/tasks/install.yml
---
- name: Install nginx packages
  package:
    name: "{{ nginx_packages }}"
    state: present
```

```yaml
# roles/nginx/tasks/configure.yml
---
- name: Deploy main configuration
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_path }}/nginx.conf"
  notify: Reload nginx
```

```yaml
# roles/nginx/tasks/ssl.yml
---
- name: Create SSL directory
  file:
    path: "{{ nginx_ssl_path }}"
    state: directory
    mode: '0700'

- name: Deploy SSL certificates
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    mode: '0600'
  loop:
    - { src: "{{ nginx_ssl_certificate }}", dest: "{{ nginx_ssl_path }}/cert.pem" }
    - { src: "{{ nginx_ssl_certificate_key }}", dest: "{{ nginx_ssl_path }}/key.pem" }
  notify: Reload nginx
```

### OS-Specific Variables

```yaml
# roles/nginx/vars/Debian.yml
---
nginx_packages:
  - nginx
  - nginx-extras
nginx_service: nginx

# roles/nginx/vars/RedHat.yml
---
nginx_packages:
  - nginx
nginx_service: nginx
```

---

## Complete Role Example

### Directory Structure

```
roles/webapp/
├── defaults/
│   └── main.yml
├── files/
│   └── app.tar.gz
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   ├── configure.yml
│   └── deploy.yml
├── templates/
│   ├── app.conf.j2
│   └── app.service.j2
├── vars/
│   └── main.yml
└── README.md
```

### defaults/main.yml

```yaml
---
app_name: myapp
app_version: "1.0.0"
app_user: app
app_group: app
app_port: 8080
app_directory: "/opt/{{ app_name }}"

app_environment: production
app_debug: false
app_log_level: info

app_database:
  host: localhost
  port: 5432
  name: "{{ app_name }}"
```

### tasks/main.yml

```yaml
---
- name: Include install tasks
  include_tasks: install.yml
  tags: install

- name: Include configure tasks
  include_tasks: configure.yml
  tags: configure

- name: Include deploy tasks
  include_tasks: deploy.yml
  tags: deploy
```

### tasks/install.yml

```yaml
---
- name: Create application user
  user:
    name: "{{ app_user }}"
    group: "{{ app_group }}"
    system: yes
    create_home: no
    shell: /usr/sbin/nologin

- name: Create application directory
  file:
    path: "{{ app_directory }}"
    state: directory
    owner: "{{ app_user }}"
    group: "{{ app_group }}"
    mode: '0755'

- name: Install application dependencies
  apt:
    name:
      - python3
      - python3-pip
      - python3-venv
    state: present
```

### tasks/configure.yml

```yaml
---
- name: Deploy application configuration
  template:
    src: app.conf.j2
    dest: "{{ app_directory }}/config/app.conf"
    owner: "{{ app_user }}"
    group: "{{ app_group }}"
    mode: '0640'
  notify: Restart app

- name: Deploy systemd service
  template:
    src: app.service.j2
    dest: /etc/systemd/system/{{ app_name }}.service
    mode: '0644'
  notify:
    - Reload systemd
    - Restart app
```

### tasks/deploy.yml

```yaml
---
- name: Extract application archive
  unarchive:
    src: "app-{{ app_version }}.tar.gz"
    dest: "{{ app_directory }}"
    owner: "{{ app_user }}"
    group: "{{ app_group }}"
  notify: Restart app

- name: Install Python requirements
  pip:
    requirements: "{{ app_directory }}/requirements.txt"
    virtualenv: "{{ app_directory }}/venv"
    virtualenv_python: python3

- name: Ensure app is running
  service:
    name: "{{ app_name }}"
    state: started
    enabled: yes
```

### handlers/main.yml

```yaml
---
- name: Reload systemd
  systemd:
    daemon_reload: yes

- name: Restart app
  service:
    name: "{{ app_name }}"
    state: restarted

- name: Stop app
  service:
    name: "{{ app_name }}"
    state: stopped
```

### meta/main.yml

```yaml
---
galaxy_info:
  author: DevOps Team
  description: Deploy custom web application
  license: MIT
  min_ansible_version: "2.10"
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy

dependencies:
  - role: common
  - role: nginx
    vars:
      nginx_sites:
        - name: "{{ app_name }}"
          server_name: "{{ app_domain }}"
          proxy_pass: "http://127.0.0.1:{{ app_port }}"
```

---

## Role Testing

### Test Playbook

```yaml
# roles/webapp/tests/test.yml
---
- name: Test webapp role
  hosts: localhost
  become: yes

  vars:
    app_name: testapp
    app_version: "1.0.0"
    app_environment: test

  roles:
    - webapp

  post_tasks:
    - name: Verify service is running
      command: systemctl is-active {{ app_name }}
      changed_when: false

    - name: Test application endpoint
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
```

### Using Molecule

```bash
# Install molecule
pip install molecule molecule-docker

# Initialize molecule scenario
cd roles/webapp
molecule init scenario --driver-name docker

# Run tests
molecule test
```

---

## Summary

| Directory | Purpose |
|-----------|---------|
| `tasks/` | Main task definitions |
| `handlers/` | Handler definitions |
| `templates/` | Jinja2 templates |
| `files/` | Static files to copy |
| `vars/` | High-precedence variables |
| `defaults/` | Low-precedence variables |
| `meta/` | Metadata and dependencies |

### Best Practices

1. Use `defaults/` for user-configurable options
2. Use `vars/` for internal constants
3. Split large tasks into multiple files
4. Document role in README.md
5. Define dependencies in `meta/main.yml`
6. Test roles with molecule or test playbooks
7. Use semantic versioning for role versions
8. Prefix private variables with underscore
