# Ansible Cheat Sheet for DevOps Engineers

## Quick Reference Commands

### Basic Commands
```bash
# Check connectivity
ansible all -m ping

# Run ad-hoc commands
ansible all -a "uptime"
ansible webservers -m shell -a "df -h"
ansible dbservers -m apt -a "name=nginx state=present" -b

# Run playbooks
ansible-playbook playbook.yml
ansible-playbook playbook.yml -i inventory.ini
ansible-playbook playbook.yml --check          # Dry run
ansible-playbook playbook.yml --diff           # Show changes
ansible-playbook playbook.yml -v               # Verbose
ansible-playbook playbook.yml -vvv             # More verbose
ansible-playbook playbook.yml --limit webservers
ansible-playbook playbook.yml --tags "install,configure"
ansible-playbook playbook.yml --skip-tags "notify"

# Inventory
ansible-inventory --list -i inventory.ini
ansible-inventory --graph

# Vault
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file vault.txt

# Galaxy
ansible-galaxy init role_name
ansible-galaxy install username.role_name
ansible-galaxy collection install community.general
```

---

## Inventory

### INI Format
```ini
# inventory.ini
[webservers]
web1.example.com
web2.example.com ansible_host=192.168.1.10

[dbservers]
db1.example.com ansible_user=admin

[production:children]
webservers
dbservers

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### YAML Format
```yaml
# inventory.yml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
          ansible_host: 192.168.1.10
    dbservers:
      hosts:
        db1.example.com:
          ansible_user: admin
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

---

## Playbooks

### Basic Playbook Structure
```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    doc_root: /var/www/html

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Start nginx service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Copy index.html
      template:
        src: templates/index.html.j2
        dest: "{{ doc_root }}/index.html"
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### Variables
```yaml
# In playbook
vars:
  app_name: myapp
  app_port: 8080

# In vars file (vars/main.yml)
db_host: localhost
db_port: 5432

# In inventory
[webservers:vars]
http_port=80

# Command line
ansible-playbook playbook.yml -e "app_port=9090"
ansible-playbook playbook.yml -e "@vars.yml"
```

### Conditionals
```yaml
- name: Install Apache on Debian
  apt:
    name: apache2
    state: present
  when: ansible_os_family == "Debian"

- name: Install Apache on RedHat
  yum:
    name: httpd
    state: present
  when: ansible_os_family == "RedHat"

# Multiple conditions
- name: Configure if Ubuntu 22
  template:
    src: config.j2
    dest: /etc/myapp/config
  when:
    - ansible_distribution == "Ubuntu"
    - ansible_distribution_version == "22.04"
```

### Loops
```yaml
# Simple loop
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl

# Loop with dict
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: 'user1', groups: 'admin' }
    - { name: 'user2', groups: 'developers' }

# Loop with index
- name: Create files
  file:
    path: "/tmp/file{{ item }}"
    state: touch
  loop: "{{ range(1, 5) | list }}"
```

---

## Common Modules

### Package Management
```yaml
# APT (Debian/Ubuntu)
- apt:
    name: nginx
    state: present
    update_cache: yes

# YUM (RHEL/CentOS)
- yum:
    name: httpd
    state: latest

# PIP
- pip:
    name: flask
    state: present
```

### Files & Templates
```yaml
# Copy file
- copy:
    src: files/config.conf
    dest: /etc/myapp/config.conf
    owner: root
    mode: '0644'

# Template
- template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

# File operations
- file:
    path: /var/www/html
    state: directory
    owner: www-data
    mode: '0755'

# Line in file
- lineinfile:
    path: /etc/hosts
    line: "192.168.1.10 myserver"
    state: present

# Block in file
- blockinfile:
    path: /etc/ssh/sshd_config
    block: |
      PermitRootLogin no
      PasswordAuthentication no
```

### Services
```yaml
- service:
    name: nginx
    state: started
    enabled: yes

- systemd:
    name: myapp
    state: restarted
    daemon_reload: yes
```

### Commands & Shell
```yaml
# Command (simple)
- command: /usr/bin/make install
  args:
    chdir: /opt/myapp

# Shell (supports pipes, redirects)
- shell: cat /etc/passwd | grep root
  register: result

# Script
- script: scripts/setup.sh
```

### Docker
```yaml
- docker_container:
    name: myapp
    image: myapp:latest
    state: started
    ports:
      - "8080:80"
    volumes:
      - /data:/app/data
    env:
      APP_ENV: production
```

---

## Roles

### Role Structure
```
roles/
  webserver/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    ├── vars/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    └── meta/
        └── main.yml
```

### Using Roles
```yaml
- hosts: webservers
  roles:
    - common
    - webserver
    - { role: database, db_port: 5432 }
```

---

## Interview Q&A

### Q1: What is Ansible and how does it work?
**A:** Ansible is an agentless automation tool that uses SSH to connect to hosts and execute tasks. Key features:
- Agentless (no daemon needed on managed nodes)
- Idempotent (safe to run multiple times)
- YAML-based playbooks
- Push-based model

### Q2: What is the difference between a playbook and a role?
**A:**
- **Playbook**: YAML file defining automation tasks for hosts
- **Role**: Reusable, modular unit of automation with defined structure (tasks, handlers, vars, templates)

### Q3: What is idempotency in Ansible?
**A:** Running the same playbook multiple times produces the same result. Ansible checks current state before making changes. If state is already correct, no action is taken.

### Q4: How do you handle secrets in Ansible?
**A:**
- **Ansible Vault**: Encrypt sensitive files/variables
- **External secret managers**: HashiCorp Vault, AWS Secrets Manager
- **Environment variables**: For CI/CD pipelines
```bash
ansible-vault encrypt secrets.yml
ansible-playbook playbook.yml --vault-password-file vault.txt
```

### Q5: What is the difference between command and shell modules?
**A:**
- **command**: Executes commands without shell processing (no pipes, redirects)
- **shell**: Executes through shell (supports pipes, redirects, variables)
Use `command` when possible for security.

### Q6: What are handlers and when do you use them?
**A:** Handlers are tasks triggered by `notify` directive. They run once at the end of the play, even if notified multiple times. Common use: restart services after config changes.

### Q7: What is Ansible Galaxy?
**A:** Repository of community-contributed roles and collections. Use to share and reuse automation content.
```bash
ansible-galaxy install geerlingguy.docker
ansible-galaxy collection install community.general
```

### Q8: How do you debug Ansible playbooks?
**A:**
```yaml
# Debug module
- debug:
    var: my_variable
    msg: "Value is {{ my_variable }}"

# Register and display
- command: cat /etc/hosts
  register: hosts_content
- debug:
    var: hosts_content.stdout

# Run with verbosity
ansible-playbook playbook.yml -vvv
```

### Q9: What is the difference between include and import?
**A:**
- **import**: Static, processed at playbook parse time
- **include**: Dynamic, processed at runtime
Use `import` for static content, `include` when using variables in filenames.

### Q10: How do you manage different environments?
**A:**
```
inventories/
  production/
    hosts
    group_vars/
  staging/
    hosts
    group_vars/

ansible-playbook -i inventories/production playbook.yml
```

---

## Best Practices

1. **Use roles** - Organize playbooks into reusable components
2. **Use Ansible Vault** - Encrypt sensitive data
3. **Use version control** - Track changes to playbooks
4. **Test in staging** - Never run untested playbooks in production
5. **Use check mode** - `--check` before applying
6. **Name all tasks** - Descriptive task names for debugging
7. **Use handlers** - For service restarts
8. **Keep playbooks simple** - One purpose per playbook

---

## Directory Structure
```
ansible/
├── ansible.cfg
├── inventory/
│   ├── production
│   └── staging
├── group_vars/
│   ├── all.yml
│   └── webservers.yml
├── host_vars/
│   └── web1.example.com.yml
├── roles/
│   ├── common/
│   └── webserver/
├── playbooks/
│   ├── site.yml
│   └── webservers.yml
└── files/
    └── templates/
```

### ansible.cfg
```ini
[defaults]
inventory = inventory/production
remote_user = ansible
private_key_file = ~/.ssh/ansible_key
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```
