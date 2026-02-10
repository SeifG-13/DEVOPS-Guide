# Ansible Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET Deployment with Ansible

### Install .NET Runtime Playbook
```yaml
---
- name: Install .NET Runtime on Linux
  hosts: webservers
  become: yes

  vars:
    dotnet_version: "8.0"

  tasks:
    - name: Install prerequisites
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
        state: present
        update_cache: yes

    - name: Add Microsoft package repository
      shell: |
        curl -sSL https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -o packages-microsoft-prod.deb
        dpkg -i packages-microsoft-prod.deb
        rm packages-microsoft-prod.deb
      args:
        creates: /etc/apt/sources.list.d/microsoft-prod.list

    - name: Install .NET Runtime
      apt:
        name: "aspnetcore-runtime-{{ dotnet_version }}"
        state: present
        update_cache: yes
```

---

## .NET Application Deployment

### Complete Deployment Playbook
```yaml
---
- name: Deploy .NET Application
  hosts: webservers
  become: yes

  vars:
    app_name: myapi
    app_user: www-data
    app_dir: /var/www/{{ app_name }}
    app_port: 5000
    dotnet_environment: Production

  tasks:
    - name: Create application directory
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'

    - name: Stop existing service
      systemd:
        name: "{{ app_name }}"
        state: stopped
      ignore_errors: yes

    - name: Deploy application files
      copy:
        src: "{{ playbook_dir }}/publish/"
        dest: "{{ app_dir }}/"
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'
      notify: Restart application

    - name: Create systemd service
      template:
        src: templates/dotnet-app.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        mode: '0644'
      notify:
        - Reload systemd
        - Restart application

    - name: Create environment file
      template:
        src: templates/app-env.j2
        dest: /etc/{{ app_name }}/environment
        mode: '0600'
        owner: root
        group: "{{ app_user }}"
      notify: Restart application

    - name: Enable and start service
      systemd:
        name: "{{ app_name }}"
        state: started
        enabled: yes

    - name: Wait for application to start
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: result
      until: result.status == 200
      retries: 10
      delay: 5

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Restart application
      systemd:
        name: "{{ app_name }}"
        state: restarted
```

### Systemd Service Template
```jinja2
{# templates/dotnet-app.service.j2 #}
[Unit]
Description={{ app_name }} .NET Application
After=network.target

[Service]
WorkingDirectory={{ app_dir }}
ExecStart=/usr/bin/dotnet {{ app_dir }}/{{ app_name }}.dll
Restart=always
RestartSec=10
User={{ app_user }}
EnvironmentFile=/etc/{{ app_name }}/environment
Environment=ASPNETCORE_URLS=http://0.0.0.0:{{ app_port }}

# Security settings
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

### Environment File Template
```jinja2
{# templates/app-env.j2 #}
ASPNETCORE_ENVIRONMENT={{ dotnet_environment }}
DOTNET_PRINT_TELEMETRY_MESSAGE=false
ConnectionStrings__DefaultConnection={{ db_connection_string }}
JWT__Secret={{ jwt_secret }}
```

---

## Configuration Management

### Deploy appsettings.json
```yaml
- name: Deploy application configuration
  template:
    src: templates/appsettings.Production.json.j2
    dest: "{{ app_dir }}/appsettings.Production.json"
    owner: "{{ app_user }}"
    mode: '0640'
  notify: Restart application

# templates/appsettings.Production.json.j2
{
  "Logging": {
    "LogLevel": {
      "Default": "{{ log_level | default('Information') }}"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "{{ db_connection_string }}"
  },
  "AllowedHosts": "{{ allowed_hosts | default('*') }}"
}
```

### Using Ansible Vault for Secrets
```bash
# Create encrypted vars file
ansible-vault create group_vars/production/vault.yml
```

```yaml
# group_vars/production/vault.yml (encrypted)
vault_db_password: "supersecretpassword"
vault_jwt_secret: "your-jwt-secret-key"

# group_vars/production/vars.yml
db_connection_string: "Server={{ db_host }};Database={{ db_name }};User={{ db_user }};Password={{ vault_db_password }}"
jwt_secret: "{{ vault_jwt_secret }}"
```

---

## Nginx Reverse Proxy Setup

```yaml
- name: Configure Nginx reverse proxy
  hosts: webservers
  become: yes

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Configure Nginx for .NET app
      template:
        src: templates/nginx-dotnet.conf.j2
        dest: /etc/nginx/sites-available/{{ app_name }}
      notify: Reload Nginx

    - name: Enable site
      file:
        src: /etc/nginx/sites-available/{{ app_name }}
        dest: /etc/nginx/sites-enabled/{{ app_name }}
        state: link
      notify: Reload Nginx

    - name: Remove default site
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded
```

### Nginx Template
```jinja2
{# templates/nginx-dotnet.conf.j2 #}
server {
    listen 80;
    server_name {{ domain_name }};

    location / {
        proxy_pass http://localhost:{{ app_port }};
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## Database Migrations

### Run EF Core Migrations
```yaml
- name: Run database migrations
  hosts: localhost
  connection: local

  vars:
    project_path: "{{ playbook_dir }}/../src/MyApp"

  tasks:
    - name: Run EF Core migrations
      command: dotnet ef database update
      args:
        chdir: "{{ project_path }}"
      environment:
        ConnectionStrings__DefaultConnection: "{{ db_connection_string }}"
      register: migration_result

    - name: Display migration output
      debug:
        var: migration_result.stdout_lines
```

### Backup Database Before Migration
```yaml
- name: Backup database before migration
  hosts: dbservers
  become: yes

  tasks:
    - name: Create backup
      command: >
        pg_dump -U {{ db_user }} -d {{ db_name }} -f /backup/{{ db_name }}_{{ ansible_date_time.date }}.sql
      environment:
        PGPASSWORD: "{{ db_password }}"
```

---

## Interview Q&A

### Q1: How do you deploy a .NET application with Ansible?
**A:**
1. Install .NET runtime on target servers
2. Create systemd service file
3. Copy published application files
4. Configure environment variables/appsettings
5. Set up reverse proxy (Nginx)
6. Start and enable the service
7. Health check to verify deployment

### Q2: How do you handle .NET application secrets in Ansible?
**A:**
- Use Ansible Vault for encrypted variables
- Deploy environment files with restricted permissions (0600)
- Use EnvironmentFile directive in systemd
- Never commit secrets to version control
```bash
ansible-vault encrypt_string 'mysecret' --name 'db_password'
```

### Q3: How do you manage different .NET environments with Ansible?
**A:**
```
inventories/
  production/
    hosts
    group_vars/
      all.yml          # ASPNETCORE_ENVIRONMENT=Production
  staging/
    hosts
    group_vars/
      all.yml          # ASPNETCORE_ENVIRONMENT=Staging
```
Use inventory-specific variables for environment configuration.

### Q4: How do you handle zero-downtime deployments?
**A:**
```yaml
# Rolling update with serial
- hosts: webservers
  serial: 1  # Update one server at a time

  tasks:
    - name: Remove from load balancer
      # ...

    - name: Deploy application
      # ...

    - name: Health check
      uri:
        url: "http://localhost:{{ app_port }}/health"
      register: health
      until: health.status == 200

    - name: Add back to load balancer
      # ...
```

### Q5: How do you rollback a .NET deployment with Ansible?
**A:**
```yaml
- name: Rollback deployment
  hosts: webservers
  vars:
    previous_version: "{{ lookup('file', 'previous_version.txt') }}"

  tasks:
    - name: Stop current service
      systemd:
        name: "{{ app_name }}"
        state: stopped

    - name: Restore previous version
      copy:
        src: "backups/{{ previous_version }}/"
        dest: "{{ app_dir }}/"

    - name: Start service
      systemd:
        name: "{{ app_name }}"
        state: started
```

---

## Complete Role Structure for .NET

```
roles/dotnet-app/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install-dotnet.yml
│   ├── deploy.yml
│   └── configure.yml
├── templates/
│   ├── dotnet-app.service.j2
│   ├── appsettings.json.j2
│   └── nginx.conf.j2
└── vars/
    └── main.yml
```

### defaults/main.yml
```yaml
app_name: myapp
app_port: 5000
app_user: www-data
app_dir: /var/www/{{ app_name }}
dotnet_version: "8.0"
dotnet_environment: Production
```

### tasks/main.yml
```yaml
---
- include_tasks: install-dotnet.yml
  when: install_dotnet | default(true)

- include_tasks: deploy.yml

- include_tasks: configure.yml
```

---

## Best Practices

1. **Use roles** - Organize .NET deployment into reusable roles
2. **Encrypt secrets** - Use Ansible Vault for connection strings, API keys
3. **Health checks** - Always verify deployment with health endpoints
4. **Backup first** - Backup application and database before deployment
5. **Use handlers** - Restart services only when config changes
6. **Serial deployment** - Rolling updates for zero downtime
7. **Version control** - Keep Ansible playbooks in Git
8. **Test in staging** - Never deploy untested changes to production
