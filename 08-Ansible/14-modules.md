# Ansible Modules

## What are Modules?

Modules are the units of work in Ansible. Each module is designed to perform a specific task, from installing packages to managing cloud resources.

```
┌─────────────────────────────────────────────────────────────┐
│                    MODULE EXECUTION                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Control Node                      Managed Node             │
│  ┌─────────────┐                  ┌─────────────┐          │
│  │   Task:     │                  │             │          │
│  │   apt:      │ ──── SSH ────►  │  Python     │          │
│  │     name:   │                  │  executes   │          │
│  │       nginx │                  │  apt module │          │
│  └─────────────┘                  └─────────────┘          │
│                                          │                  │
│  ┌─────────────┐                        │                  │
│  │   Result:   │ ◄─── JSON ────────────┘                  │
│  │   changed:  │                                           │
│  │     true    │                                           │
│  └─────────────┘                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Module Categories

### File Modules

#### copy

```yaml
# Copy file to remote
- name: Copy configuration file
  copy:
    src: app.conf                    # Local file
    dest: /etc/app/app.conf          # Remote destination
    owner: root
    group: root
    mode: '0644'
    backup: yes                       # Create backup if exists

# Copy content directly
- name: Create file from content
  copy:
    content: |
      server {
        listen 80;
        server_name example.com;
      }
    dest: /etc/nginx/sites-available/example
    mode: '0644'
```

#### template

```yaml
# Deploy Jinja2 template
- name: Deploy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: 'nginx -t -c %s'       # Validate before replacing
    backup: yes
```

#### file

```yaml
# Create directory
- name: Create application directory
  file:
    path: /opt/app
    state: directory
    owner: app
    group: app
    mode: '0755'
    recurse: yes                      # Apply recursively

# Create symlink
- name: Enable site
  file:
    src: /etc/nginx/sites-available/mysite
    dest: /etc/nginx/sites-enabled/mysite
    state: link

# Delete file or directory
- name: Remove old files
  file:
    path: /tmp/old_file
    state: absent

# Touch file (create if not exists)
- name: Touch file
  file:
    path: /var/log/app.log
    state: touch
    mode: '0644'
```

#### lineinfile

```yaml
# Ensure line exists in file
- name: Enable IP forwarding
  lineinfile:
    path: /etc/sysctl.conf
    regexp: '^net.ipv4.ip_forward'
    line: 'net.ipv4.ip_forward = 1'

# Remove line matching pattern
- name: Remove comment
  lineinfile:
    path: /etc/app.conf
    regexp: '^#.*disabled'
    state: absent

# Insert after specific line
- name: Add nameserver
  lineinfile:
    path: /etc/resolv.conf
    insertafter: '^search'
    line: 'nameserver 8.8.8.8'

# Insert before specific line
- name: Add header
  lineinfile:
    path: /etc/hosts
    insertbefore: BOF                 # Beginning of file
    line: '# Managed by Ansible'
```

#### blockinfile

```yaml
# Insert/update block of text
- name: Add SSH config block
  blockinfile:
    path: /etc/ssh/sshd_config
    block: |
      Match Group developers
        PasswordAuthentication no
        PubkeyAuthentication yes
    marker: "# {mark} ANSIBLE MANAGED BLOCK - developers"

# Remove block
- name: Remove block
  blockinfile:
    path: /etc/hosts
    marker: "# {mark} CUSTOM HOSTS"
    state: absent
```

#### fetch

```yaml
# Fetch file from remote to local
- name: Fetch log file
  fetch:
    src: /var/log/app.log
    dest: /tmp/logs/
    flat: yes                         # Don't create hostname directory

# Fetch with directory structure
- name: Backup configs
  fetch:
    src: /etc/nginx/nginx.conf
    dest: ./backups/                  # Creates backups/hostname/etc/nginx/nginx.conf
```

#### synchronize

```yaml
# Rsync wrapper
- name: Sync application files
  synchronize:
    src: /local/app/
    dest: /opt/app/
    delete: yes                       # Delete files not in source
    recursive: yes

# Push to remote
- name: Deploy files
  synchronize:
    src: ./dist/
    dest: /var/www/html/
    mode: push

# Pull from remote
- name: Backup files
  synchronize:
    src: /var/www/html/
    dest: ./backup/
    mode: pull
```

---

### Package Modules

#### apt (Debian/Ubuntu)

```yaml
# Install package
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

# Install multiple packages
- name: Install packages
  apt:
    name:
      - nginx
      - php-fpm
      - mysql-client
    state: present

# Install specific version
- name: Install specific version
  apt:
    name: nginx=1.18.0-0ubuntu1
    state: present

# Remove package
- name: Remove apache
  apt:
    name: apache2
    state: absent
    purge: yes                        # Remove config files too
    autoremove: yes                   # Remove unused dependencies

# Update all packages
- name: Upgrade all packages
  apt:
    upgrade: dist
    update_cache: yes

# Add repository
- name: Add repository
  apt_repository:
    repo: ppa:ondrej/php
    state: present
```

#### yum/dnf (RHEL/CentOS)

```yaml
# Install package
- name: Install httpd
  yum:
    name: httpd
    state: present

# Install from URL
- name: Install package from URL
  yum:
    name: https://example.com/package.rpm
    state: present

# Install group
- name: Install development tools
  yum:
    name: "@Development Tools"
    state: present

# Enable repository
- name: Enable EPEL
  yum:
    name: epel-release
    state: present
```

#### pip

```yaml
# Install Python package
- name: Install Flask
  pip:
    name: flask
    state: present

# Install specific version
- name: Install specific version
  pip:
    name: flask==2.0.0
    state: present

# Install from requirements.txt
- name: Install requirements
  pip:
    requirements: /opt/app/requirements.txt

# Install in virtualenv
- name: Install in virtualenv
  pip:
    name: flask
    virtualenv: /opt/app/venv
    virtualenv_python: python3

# Install multiple packages
- name: Install packages
  pip:
    name:
      - flask
      - gunicorn
      - redis
    state: latest
```

#### package (Generic)

```yaml
# Works across distributions
- name: Install package
  package:
    name: git
    state: present
```

---

### Service Modules

#### service

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

# Stop service
- name: Stop nginx
  service:
    name: nginx
    state: stopped
```

#### systemd

```yaml
# Start service with daemon reload
- name: Start application
  systemd:
    name: myapp
    state: started
    enabled: yes
    daemon_reload: yes

# Reload systemd
- name: Reload systemd
  systemd:
    daemon_reload: yes

# Mask service
- name: Mask service
  systemd:
    name: bluetooth
    masked: yes
```

---

### User and Group Modules

#### user

```yaml
# Create user
- name: Create deploy user
  user:
    name: deploy
    comment: "Deployment User"
    shell: /bin/bash
    groups: sudo,docker
    append: yes                       # Append to groups (don't replace)
    create_home: yes
    state: present

# Create system user
- name: Create service user
  user:
    name: myapp
    system: yes
    shell: /usr/sbin/nologin
    home: /opt/myapp
    create_home: no

# Generate SSH key
- name: Create user with SSH key
  user:
    name: deploy
    generate_ssh_key: yes
    ssh_key_bits: 4096
    ssh_key_file: .ssh/id_rsa

# Set password
- name: Set user password
  user:
    name: deploy
    password: "{{ 'password' | password_hash('sha512') }}"

# Remove user
- name: Remove user
  user:
    name: olduser
    state: absent
    remove: yes                       # Remove home directory
```

#### group

```yaml
# Create group
- name: Create developers group
  group:
    name: developers
    state: present

# Create with specific GID
- name: Create group with GID
  group:
    name: mygroup
    gid: 1500
    state: present
```

#### authorized_key

```yaml
# Add SSH key
- name: Add SSH key
  authorized_key:
    user: deploy
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    state: present

# Add key from URL
- name: Add key from GitHub
  authorized_key:
    user: deploy
    key: https://github.com/username.keys
    state: present

# Multiple keys
- name: Add SSH keys
  authorized_key:
    user: deploy
    key: "{{ item }}"
    state: present
  loop:
    - "ssh-rsa AAAA... alice@laptop"
    - "ssh-rsa AAAA... bob@desktop"
```

---

### Command Modules

#### command

```yaml
# Run command (no shell)
- name: Check disk space
  command: df -h /
  register: disk_space
  changed_when: false                 # Never report as changed

# With arguments
- name: Run with args
  command:
    cmd: /opt/app/bin/migrate
    chdir: /opt/app                   # Change directory first

# Only run if file doesn't exist
- name: Initialize database
  command: /opt/app/init_db.sh
  args:
    creates: /opt/app/db.sqlite       # Skip if file exists

# Only run if file exists
- name: Cleanup
  command: /opt/app/cleanup.sh
  args:
    removes: /tmp/cleanup_needed      # Skip if file doesn't exist
```

#### shell

```yaml
# Run shell command (supports pipes, redirects)
- name: Get process count
  shell: ps aux | grep nginx | wc -l
  register: nginx_processes

# With bash explicitly
- name: Run bash script
  shell: |
    source /opt/app/env.sh
    ./deploy.sh
  args:
    executable: /bin/bash
    chdir: /opt/app

# With environment variables
- name: Run with env
  shell: echo $MY_VAR
  environment:
    MY_VAR: "my_value"
```

#### raw

```yaml
# Run without Python (for bootstrap)
- name: Install Python
  raw: apt-get install -y python3
  become: yes

# On network devices
- name: Show version
  raw: show version
```

#### script

```yaml
# Run local script on remote
- name: Run setup script
  script: scripts/setup.sh
  args:
    creates: /opt/app/.initialized

# With arguments
- name: Run script with args
  script: scripts/configure.sh --env production
```

---

### Network Modules

#### uri

```yaml
# GET request
- name: Check API health
  uri:
    url: http://localhost:8080/health
    method: GET
    status_code: 200
  register: health_check

# POST request with JSON
- name: Create resource
  uri:
    url: https://api.example.com/items
    method: POST
    headers:
      Authorization: "Bearer {{ api_token }}"
      Content-Type: application/json
    body:
      name: "new-item"
      value: 123
    body_format: json
    status_code: [200, 201]
  register: api_response

# Download file
- name: Download file
  uri:
    url: https://example.com/file.tar.gz
    dest: /tmp/file.tar.gz
    mode: '0644'
```

#### get_url

```yaml
# Download file
- name: Download application
  get_url:
    url: https://releases.example.com/app-1.0.tar.gz
    dest: /tmp/app.tar.gz
    mode: '0644'
    checksum: sha256:abc123def456...

# With authentication
- name: Download with auth
  get_url:
    url: https://private.example.com/file
    dest: /tmp/file
    url_username: user
    url_password: pass
```

#### wait_for

```yaml
# Wait for port to be available
- name: Wait for application
  wait_for:
    host: localhost
    port: 8080
    delay: 5
    timeout: 300

# Wait for file
- name: Wait for file
  wait_for:
    path: /var/run/app.pid
    state: present

# Wait for port to close
- name: Wait for shutdown
  wait_for:
    port: 8080
    state: stopped
    timeout: 60
```

---

### Cloud Modules

#### amazon.aws.ec2_instance

```yaml
# Create EC2 instance
- name: Launch EC2 instance
  amazon.aws.ec2_instance:
    name: web-server
    instance_type: t3.micro
    image_id: ami-12345678
    key_name: my-key
    vpc_subnet_id: subnet-abc123
    security_groups:
      - web-sg
    tags:
      Environment: production
    state: running
  register: ec2
```

#### azure.azcollection.azure_rm_virtualmachine

```yaml
# Create Azure VM
- name: Create Azure VM
  azure.azcollection.azure_rm_virtualmachine:
    resource_group: myResourceGroup
    name: web-vm
    vm_size: Standard_B1s
    admin_username: adminuser
    ssh_password_enabled: false
    ssh_public_keys:
      - path: /home/adminuser/.ssh/authorized_keys
        key_data: "{{ ssh_public_key }}"
    image:
      offer: UbuntuServer
      publisher: Canonical
      sku: '22.04-LTS'
      version: latest
```

---

### Database Modules

#### mysql_db

```yaml
# Create database
- name: Create database
  mysql_db:
    name: myapp
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"

# Import database
- name: Import database
  mysql_db:
    name: myapp
    state: import
    target: /tmp/dump.sql
```

#### mysql_user

```yaml
# Create user with privileges
- name: Create database user
  mysql_user:
    name: myapp
    password: "{{ db_password }}"
    priv: 'myapp.*:ALL'
    host: '%'
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"
```

#### postgresql_db

```yaml
# Create PostgreSQL database
- name: Create database
  postgresql_db:
    name: myapp
    owner: myapp
    state: present
  become: yes
  become_user: postgres
```

---

### System Modules

#### cron

```yaml
# Create cron job
- name: Schedule backup
  cron:
    name: "Daily backup"
    minute: "0"
    hour: "2"
    job: "/opt/scripts/backup.sh"
    user: root

# Remove cron job
- name: Remove old job
  cron:
    name: "Old job"
    state: absent
```

#### sysctl

```yaml
# Set kernel parameter
- name: Enable IP forwarding
  sysctl:
    name: net.ipv4.ip_forward
    value: '1'
    sysctl_set: yes
    state: present
    reload: yes
```

#### hostname

```yaml
# Set hostname
- name: Set hostname
  hostname:
    name: web1.example.com
```

#### timezone

```yaml
# Set timezone
- name: Set timezone
  timezone:
    name: America/New_York
```

#### reboot

```yaml
# Reboot machine
- name: Reboot server
  reboot:
    reboot_timeout: 300
    msg: "Rebooting for kernel update"
```

---

## Finding Module Documentation

```bash
# List all modules
ansible-doc -l

# Search modules
ansible-doc -l | grep docker

# View module documentation
ansible-doc apt
ansible-doc copy
ansible-doc amazon.aws.ec2_instance

# Show examples only
ansible-doc apt -s
```

---

## Custom Modules

### Simple Custom Module

```python
#!/usr/bin/python
# library/my_module.py

from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(type='str', required=True),
            state=dict(type='str', default='present', choices=['present', 'absent']),
        ),
        supports_check_mode=True
    )

    name = module.params['name']
    state = module.params['state']

    # Your logic here
    changed = False
    result = {'name': name, 'state': state}

    module.exit_json(changed=changed, **result)

if __name__ == '__main__':
    main()
```

```yaml
# Use custom module
- name: Use my module
  my_module:
    name: example
    state: present
```

---

## Summary

| Category | Common Modules |
|----------|---------------|
| **Files** | copy, template, file, lineinfile, blockinfile |
| **Packages** | apt, yum, dnf, pip, package |
| **Services** | service, systemd |
| **Users** | user, group, authorized_key |
| **Commands** | command, shell, raw, script |
| **Network** | uri, get_url, wait_for |
| **Cloud** | ec2_instance, azure_rm_* |
| **Database** | mysql_db, mysql_user, postgresql_db |
| **System** | cron, sysctl, hostname, reboot |

### Key Principles

1. **Idempotency**: Modules should produce same result regardless of runs
2. **Check Mode**: Support `--check` for dry runs
3. **Return Values**: Use `register` to capture output
4. **FQCN**: Use fully qualified names for collection modules
