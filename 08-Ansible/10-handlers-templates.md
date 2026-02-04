# Ansible Handlers and Templates

## Handlers

Handlers are special tasks that run only when notified by other tasks.

### Why Handlers?

```
┌─────────────────────────────────────────────────────────────┐
│                    WITHOUT HANDLERS                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Task 1: Update config  ──► Restart service                 │
│  Task 2: Update config  ──► Restart service                 │
│  Task 3: Update config  ──► Restart service                 │
│                                                             │
│  Result: Service restarts 3 times! (inefficient)            │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                     WITH HANDLERS                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Task 1: Update config  ──► notify: Restart service         │
│  Task 2: Update config  ──► notify: Restart service         │
│  Task 3: Update config  ──► notify: Restart service         │
│                             │                               │
│                             ▼                               │
│                    Handler runs ONCE at end                 │
│                                                             │
│  Result: Service restarts only 1 time! (efficient)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Basic Handler

```yaml
---
- name: Configure Web Server
  hosts: webservers
  become: yes

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Copy nginx configuration
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx  # Notify handler

    - name: Copy site configuration
      copy:
        src: site.conf
        dest: /etc/nginx/sites-available/default
      notify: Restart nginx  # Same handler, runs only once

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### Multiple Handlers

```yaml
---
- name: Full web setup
  hosts: webservers
  become: yes

  tasks:
    - name: Update nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - Validate nginx config
        - Reload nginx

    - name: Update PHP config
      template:
        src: php.ini.j2
        dest: /etc/php/8.1/fpm/php.ini
      notify: Restart php-fpm

  handlers:
    - name: Validate nginx config
      command: nginx -t
      changed_when: false

    - name: Reload nginx
      service:
        name: nginx
        state: reloaded

    - name: Restart php-fpm
      service:
        name: php8.1-fpm
        state: restarted
```

### Handler Execution Order

Handlers run in the order they are **defined**, not notified.

```yaml
handlers:
  # Runs first (if notified)
  - name: Stop nginx
    service:
      name: nginx
      state: stopped

  # Runs second (if notified)
  - name: Update certificates
    command: certbot renew

  # Runs third (if notified)
  - name: Start nginx
    service:
      name: nginx
      state: started
```

### Listen Directive

Multiple handlers can listen to the same notification.

```yaml
tasks:
  - name: Update application
    copy:
      src: app/
      dest: /opt/app/
    notify: App updated  # Generic notification

handlers:
  - name: Clear cache
    command: /opt/app/clear_cache.sh
    listen: App updated  # Listens to "App updated"

  - name: Restart workers
    service:
      name: app-workers
      state: restarted
    listen: App updated  # Also listens

  - name: Send notification
    mail:
      to: admin@example.com
      subject: "App updated"
    listen: App updated  # Also listens
```

### Flush Handlers

Force handlers to run immediately.

```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx

  - name: Force handler to run now
    meta: flush_handlers

  - name: Task that needs nginx running
    uri:
      url: http://localhost/health
      status_code: 200
```

### Handlers in Blocks

```yaml
tasks:
  - name: Configure web server
    block:
      - name: Copy nginx config
        copy:
          src: nginx.conf
          dest: /etc/nginx/nginx.conf
        notify: Restart nginx

      - name: Copy site config
        copy:
          src: site.conf
          dest: /etc/nginx/sites-available/default
        notify: Restart nginx

    rescue:
      - name: Restore backup config
        copy:
          src: /etc/nginx/nginx.conf.bak
          dest: /etc/nginx/nginx.conf
        notify: Restart nginx

handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
```

### Handlers in Roles

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

# roles/nginx/tasks/main.yml
---
- name: Update config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Reload nginx
```

---

## Jinja2 Templates

Templates allow dynamic file generation using variables and logic.

### Basic Template

```yaml
# Playbook
---
- name: Configure app
  hosts: webservers
  vars:
    app_name: myapp
    app_port: 8080

  tasks:
    - name: Create config file
      template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
```

```jinja2
{# templates/app.conf.j2 #}
# Application Configuration
# Generated by Ansible - DO NOT EDIT MANUALLY

[application]
name = {{ app_name }}
port = {{ app_port }}
host = {{ ansible_hostname }}
ip = {{ ansible_default_ipv4.address }}
```

### Template Module Options

```yaml
- name: Deploy configuration
  template:
    src: app.conf.j2
    dest: /etc/app/app.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes  # Create backup of existing
    validate: '/usr/bin/app --validate %s'  # Validate before replacing
    force: yes  # Overwrite if exists
```

---

## Jinja2 Syntax

### Variables

```jinja2
{# Simple variable #}
server_name = {{ server_name }}

{# Dictionary access #}
db_host = {{ database.host }}
db_port = {{ database['port'] }}

{# List access #}
first_server = {{ servers[0] }}

{# Default value #}
timeout = {{ timeout | default(30) }}

{# With quotes for strings #}
name = "{{ app_name }}"
```

### Comments

```jinja2
{# This is a comment - not rendered in output #}

{#
   Multi-line
   comment
#}

# This IS rendered (shell/config comment)
```

### Conditionals

```jinja2
{# If statement #}
{% if debug_mode %}
DEBUG = true
LOG_LEVEL = debug
{% else %}
DEBUG = false
LOG_LEVEL = warn
{% endif %}

{# If-elif-else #}
{% if environment == 'production' %}
REPLICAS = 3
{% elif environment == 'staging' %}
REPLICAS = 2
{% else %}
REPLICAS = 1
{% endif %}

{# Inline if #}
SECURE = {{ 'yes' if use_ssl else 'no' }}

{# Check if defined #}
{% if custom_setting is defined %}
CUSTOM = {{ custom_setting }}
{% endif %}
```

### Loops

```jinja2
{# Simple loop #}
{% for server in servers %}
server {{ server }};
{% endfor %}

{# Loop with index #}
{% for user in users %}
# User {{ loop.index }}: {{ user.name }}
{% endfor %}

{# Loop variables #}
{% for item in items %}
{{ loop.index }}     {# 1-indexed counter #}
{{ loop.index0 }}    {# 0-indexed counter #}
{{ loop.first }}     {# True if first iteration #}
{{ loop.last }}      {# True if last iteration #}
{{ loop.length }}    {# Total number of items #}
{% endfor %}

{# Loop with condition #}
{% for user in users if user.active %}
{{ user.name }}
{% endfor %}

{# Loop with else (empty list) #}
{% for server in servers %}
{{ server }}
{% else %}
# No servers configured
{% endfor %}

{# Dictionary loop #}
{% for key, value in settings.items() %}
{{ key }} = {{ value }}
{% endfor %}
```

### Whitespace Control

```jinja2
{# Default - adds newlines #}
{% for item in items %}
{{ item }}
{% endfor %}

{# Remove whitespace with - #}
{% for item in items -%}
{{ item }}
{%- endfor %}

{# Result: items on same line #}
```

---

## Jinja2 Filters

### String Filters

```jinja2
{# Case conversion #}
{{ name | upper }}         {# JOHN #}
{{ name | lower }}         {# john #}
{{ name | capitalize }}    {# John #}
{{ name | title }}         {# John Doe #}

{# String manipulation #}
{{ text | trim }}          {# Remove whitespace #}
{{ text | replace('old', 'new') }}
{{ path | basename }}      {# file.txt from /path/to/file.txt #}
{{ path | dirname }}       {# /path/to from /path/to/file.txt #}

{# Regex #}
{{ text | regex_search('pattern') }}
{{ text | regex_replace('pattern', 'replace') }}

{# Length and truncate #}
{{ text | length }}
{{ text | truncate(50) }}
```

### List Filters

```jinja2
{# Joining #}
{{ items | join(', ') }}   {# a, b, c #}

{# Sorting #}
{{ items | sort }}
{{ items | sort(reverse=true) }}
{{ users | sort(attribute='name') }}

{# Filtering #}
{{ items | unique }}
{{ items | first }}
{{ items | last }}
{{ items | random }}

{# Selection #}
{{ users | selectattr('active', 'equalto', true) | list }}
{{ users | map(attribute='name') | list }}
{{ items | reject('equalto', 'skip') | list }}

{# Math on lists #}
{{ numbers | min }}
{{ numbers | max }}
{{ numbers | sum }}
```

### Number Filters

```jinja2
{# Type conversion #}
{{ value | int }}
{{ value | float }}

{# Rounding #}
{{ 3.7 | round }}          {# 4 #}
{{ 3.7 | round(1) }}       {# 3.7 #}

{# Math #}
{{ value | abs }}
{{ items | length }}
```

### Data Structure Filters

```jinja2
{# JSON/YAML #}
{{ data | to_json }}
{{ data | to_nice_json }}
{{ data | to_yaml }}
{{ data | to_nice_yaml }}
{{ json_string | from_json }}
{{ yaml_string | from_yaml }}

{# Dictionary #}
{{ dict | dict2items }}
{{ items | items2dict }}
{{ dict1 | combine(dict2) }}

{# Hash/Encrypt #}
{{ password | password_hash('sha512') }}
{{ data | hash('sha256') }}
{{ text | b64encode }}
{{ encoded | b64decode }}
```

### Default and Required

```jinja2
{# Default value #}
{{ variable | default('fallback') }}
{{ variable | default(omit) }}  {# Omit parameter if undefined #}

{# Mandatory variable #}
{{ required_var | mandatory }}

{# Default for boolean #}
{{ feature_enabled | default(false) | bool }}
```

### Ansible-Specific Filters

```jinja2
{# IP address manipulation #}
{{ '192.168.1.0/24' | ipaddr('network') }}
{{ ip_list | ipaddr('private') }}

{# File operations #}
{{ '/etc/hosts' | basename }}   {# hosts #}
{{ '/etc/hosts' | dirname }}    {# /etc #}
{{ 'file.tar.gz' | splitext }}  {# ['file.tar', '.gz'] #}

{# Regex #}
{{ hostname | regex_search('^web(\d+)') }}

{# Path joining #}
{{ ['/etc', 'nginx', 'nginx.conf'] | path_join }}

{# Quote for shell #}
{{ filename | quote }}
```

---

## Practical Template Examples

### Nginx Virtual Host

```jinja2
{# templates/nginx-vhost.conf.j2 #}
# {{ ansible_managed }}
# Server: {{ inventory_hostname }}

{% for domain in domains %}
server {
    listen {{ http_port | default(80) }};
    server_name {{ domain.name }}{% if domain.aliases is defined %} {{ domain.aliases | join(' ') }}{% endif %};

{% if domain.ssl | default(false) %}
    listen {{ https_port | default(443) }} ssl;
    ssl_certificate /etc/ssl/certs/{{ domain.name }}.crt;
    ssl_certificate_key /etc/ssl/private/{{ domain.name }}.key;
{% endif %}

    root {{ domain.root | default('/var/www/' + domain.name) }};
    index index.html index.php;

{% if domain.locations is defined %}
{% for location in domain.locations %}
    location {{ location.path }} {
{% for directive in location.directives %}
        {{ directive }};
{% endfor %}
    }

{% endfor %}
{% endif %}
    access_log /var/log/nginx/{{ domain.name }}_access.log;
    error_log /var/log/nginx/{{ domain.name }}_error.log;
}

{% endfor %}
```

### Application Configuration

```jinja2
{# templates/app.yml.j2 #}
# Application Configuration
# Environment: {{ environment }}
# Generated: {{ ansible_date_time.iso8601 }}

app:
  name: {{ app_name }}
  version: {{ app_version }}
  debug: {{ debug | default(false) | lower }}

server:
  host: {{ ansible_default_ipv4.address }}
  port: {{ app_port }}
  workers: {{ ansible_processor_vcpus | default(2) }}

database:
  driver: {{ db_driver | default('postgresql') }}
  host: {{ db_host }}
  port: {{ db_port | default(5432) }}
  name: {{ db_name }}
  user: {{ db_user }}
  # Password is in vault

{% if redis_enabled | default(false) %}
cache:
  driver: redis
  host: {{ redis_host }}
  port: {{ redis_port | default(6379) }}
{% endif %}

logging:
  level: {{ log_level | default('info') }}
  format: json
{% if environment == 'production' %}
  outputs:
    - file
    - syslog
{% else %}
  outputs:
    - console
{% endif %}

{% if feature_flags is defined %}
features:
{% for flag, enabled in feature_flags.items() %}
  {{ flag }}: {{ enabled | lower }}
{% endfor %}
{% endif %}
```

### Dynamic Hosts File

```jinja2
{# templates/hosts.j2 #}
# /etc/hosts - Managed by Ansible
127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback

# Cluster hosts
{% for host in groups['all'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }}  {{ host }} {{ hostvars[host]['ansible_hostname'] }}
{% endfor %}

{% if additional_hosts is defined %}
# Additional hosts
{% for entry in additional_hosts %}
{{ entry.ip }}  {{ entry.names | join(' ') }}
{% endfor %}
{% endif %}
```

### Systemd Service Unit

```jinja2
{# templates/app.service.j2 #}
[Unit]
Description={{ app_description | default(app_name + ' Service') }}
After=network.target
{% if app_requires is defined %}
Requires={{ app_requires | join(' ') }}
{% endif %}

[Service]
Type={{ service_type | default('simple') }}
User={{ app_user | default('app') }}
Group={{ app_group | default('app') }}
WorkingDirectory={{ app_directory }}
ExecStart={{ app_directory }}/bin/{{ app_name }}{% if app_args is defined %} {{ app_args | join(' ') }}{% endif %}

{% if app_env is defined %}
{% for key, value in app_env.items() %}
Environment="{{ key }}={{ value }}"
{% endfor %}
{% endif %}

Restart={{ restart_policy | default('on-failure') }}
RestartSec={{ restart_sec | default(5) }}

{% if memory_limit is defined %}
MemoryLimit={{ memory_limit }}
{% endif %}

[Install]
WantedBy=multi-user.target
```

---

## Best Practices

### Template Organization

```
project/
├── templates/
│   ├── nginx/
│   │   ├── nginx.conf.j2
│   │   └── vhost.conf.j2
│   ├── app/
│   │   ├── config.yml.j2
│   │   └── env.j2
│   └── systemd/
│       └── app.service.j2
```

### Template Header

```jinja2
{#
  Template: nginx.conf.j2
  Description: Main nginx configuration
  Variables required:
    - worker_processes
    - worker_connections
#}
# {{ ansible_managed }}
# DO NOT EDIT - This file is managed by Ansible
```

### Validate Before Deploy

```yaml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: 'nginx -t -c %s'
  notify: Reload nginx
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Handler** | Task that runs when notified |
| **notify** | Trigger a handler |
| **listen** | Multiple handlers for one notification |
| **flush_handlers** | Run handlers immediately |
| **template** | Jinja2 template module |
| **Jinja2** | Template language |
| **{{ }}** | Variable interpolation |
| **{% %}** | Control statements |
| **{# #}** | Comments |
| **Filters** | Transform values |

### Handler Best Practices

1. Use handlers for service restarts/reloads
2. Name handlers descriptively
3. Use `listen` for related handler groups
4. Flush handlers when order matters

### Template Best Practices

1. Add `ansible_managed` comment
2. Validate templates before deployment
3. Use filters for data transformation
4. Keep templates readable with proper indentation
5. Document required variables
