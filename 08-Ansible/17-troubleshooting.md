# Ansible Troubleshooting

## Verbose Output

Increase verbosity to get more information about what Ansible is doing.

### Verbosity Levels

```bash
# Level 1: Basic info
ansible-playbook playbook.yml -v

# Level 2: More details, shows task results
ansible-playbook playbook.yml -vv

# Level 3: Connection details
ansible-playbook playbook.yml -vvv

# Level 4: Full debug (includes SSH debug)
ansible-playbook playbook.yml -vvvv
```

### What Each Level Shows

| Level | Flag | Information |
|-------|------|-------------|
| 0 | (none) | Task status only |
| 1 | `-v` | Task results |
| 2 | `-vv` | Task input parameters |
| 3 | `-vvv` | Connection info, file paths |
| 4 | `-vvvv` | SSH/WinRM debug output |

---

## Debug Module

Use the debug module to inspect variables and flow.

```yaml
tasks:
  # Print variable value
  - name: Show variable
    debug:
      var: my_variable

  # Print message with variables
  - name: Show message
    debug:
      msg: "The value of my_var is: {{ my_var }}"

  # Show all variables for a host
  - name: Show all vars
    debug:
      var: hostvars[inventory_hostname]

  # Show ansible facts
  - name: Show facts
    debug:
      var: ansible_facts

  # Conditional debug
  - name: Debug only when verbose
    debug:
      msg: "Detailed debug info"
      verbosity: 2  # Only shows with -vv or higher
```

### Debugging Registered Variables

```yaml
tasks:
  - name: Run command
    command: ls -la /etc
    register: result

  # See full registered output
  - name: Show full result
    debug:
      var: result

  # Show specific parts
  - name: Show stdout
    debug:
      msg: |
        Return code: {{ result.rc }}
        Stdout: {{ result.stdout }}
        Stderr: {{ result.stderr }}
        Changed: {{ result.changed }}
```

---

## Check Mode (Dry Run)

Test playbooks without making changes.

```bash
# Run in check mode
ansible-playbook playbook.yml --check

# Check mode with diff
ansible-playbook playbook.yml --check --diff
```

### Check Mode in Tasks

```yaml
tasks:
  # Force check mode for task
  - name: Always check
    command: /opt/scripts/verify.sh
    check_mode: yes

  # Never use check mode (always execute)
  - name: Must execute
    command: /opt/scripts/critical.sh
    check_mode: no

  # Detect if running in check mode
  - name: Different behavior in check mode
    debug:
      msg: "This is a dry run"
    when: ansible_check_mode
```

---

## Step Mode

Execute tasks one at a time with confirmation.

```bash
# Step through each task
ansible-playbook playbook.yml --step

# Prompts:
# Perform task: Install nginx (N)o/(y)es/(c)ontinue:
```

---

## Start at Task

Skip to a specific task.

```bash
# Start at specific task name
ansible-playbook playbook.yml --start-at-task="Configure nginx"

# List all tasks first
ansible-playbook playbook.yml --list-tasks
```

---

## Syntax Check

Validate playbook syntax without execution.

```bash
# Check syntax only
ansible-playbook playbook.yml --syntax-check

# Output if OK:
# playbook: playbook.yml

# Output if error:
# ERROR! Syntax Error while loading YAML.
```

---

## Common Errors and Solutions

### SSH Connection Errors

#### Error: "Permission denied (publickey)"

```bash
# Check SSH key permissions
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# Verify SSH works manually
ssh -i ~/.ssh/id_rsa user@host -v

# Check ansible.cfg
[defaults]
private_key_file = ~/.ssh/id_rsa
remote_user = your_user
```

#### Error: "Host key verification failed"

```bash
# Option 1: Add to known_hosts
ssh-keyscan -H hostname >> ~/.ssh/known_hosts

# Option 2: Disable in ansible.cfg (not recommended for production)
[defaults]
host_key_checking = False

# Option 3: Environment variable
export ANSIBLE_HOST_KEY_CHECKING=False
```

#### Error: "Connection timed out"

```bash
# Increase timeout in ansible.cfg
[defaults]
timeout = 60

# Check network connectivity
ping hostname
telnet hostname 22

# Check firewall rules
sudo iptables -L
```

### Python Errors

#### Error: "module not found"

```yaml
# Specify Python interpreter
ansible_python_interpreter: /usr/bin/python3

# In inventory
[all:vars]
ansible_python_interpreter=/usr/bin/python3

# Or per host
webserver ansible_python_interpreter=/usr/bin/python3
```

#### Error: "Python interpreter discovery failed"

```bash
# In ansible.cfg
[defaults]
interpreter_python = auto_silent

# Or specify explicitly
interpreter_python = /usr/bin/python3
```

### Privilege Escalation Errors

#### Error: "Missing sudo password"

```bash
# Provide password at runtime
ansible-playbook playbook.yml -K

# Or configure in inventory (not recommended)
[all:vars]
ansible_become_password=your_password

# Or use password file (better)
ansible-playbook playbook.yml --become-password-file=.become_pass
```

#### Error: "sudo: a password is required"

```yaml
# Ensure user has passwordless sudo
# On target: /etc/sudoers.d/ansible
ansible ALL=(ALL) NOPASSWD:ALL

# Or in playbook
become: yes
become_method: sudo
become_user: root
```

### Variable Errors

#### Error: "undefined variable"

```yaml
# Use default filter
{{ my_var | default('default_value') }}

# Check if defined
when: my_var is defined

# Make variable required
{{ my_var | mandatory }}
```

#### Error: "dict object has no attribute"

```yaml
# Wrong
{{ database.password }}  # If database is undefined

# Safe access with default
{{ database.password | default('') }}

# Or check first
when: database is defined and database.password is defined
```

### Module Errors

#### Error: "No module named X"

```bash
# Check if collection is installed
ansible-galaxy collection list

# Install missing collection
ansible-galaxy collection install community.general

# Check module documentation
ansible-doc module_name
```

#### Error: "apt/yum module not found"

```bash
# Install required Python packages on target
# For apt:
sudo apt install python3-apt

# For yum:
sudo yum install python3-dnf
```

---

## Debugging Connections

### Test Connectivity

```bash
# Basic ping
ansible all -m ping

# With specific inventory
ansible all -i inventory -m ping

# Verbose connection test
ansible all -m ping -vvvv

# Test specific host
ansible webserver -m ping -vvvv
```

### Connection Testing Playbook

```yaml
---
- name: Connection diagnostics
  hosts: all
  gather_facts: no

  tasks:
    - name: Test connectivity
      ping:

    - name: Check Python version
      command: python3 --version
      register: python_version
      changed_when: false

    - name: Show Python version
      debug:
        var: python_version.stdout

    - name: Check user
      command: whoami
      register: current_user
      changed_when: false

    - name: Show user
      debug:
        var: current_user.stdout

    - name: Check sudo access
      command: sudo whoami
      register: sudo_user
      changed_when: false
      become: yes

    - name: Show sudo result
      debug:
        var: sudo_user.stdout
```

---

## Performance Troubleshooting

### Slow Playbook Execution

```bash
# Enable timing information
# In ansible.cfg
[defaults]
callback_whitelist = timer, profile_tasks

# Or via environment
export ANSIBLE_CALLBACKS_ENABLED=timer,profile_tasks

# Run playbook to see timing
ansible-playbook playbook.yml
```

### Profile Output Example

```
PLAY RECAP *************
Wednesday 01 January 2025  10:00:00 +0000 (0:00:02.123)
===============================================================
Install packages ---------------------------------------- 45.23s
Configure nginx ----------------------------------------- 12.45s
Start services ------------------------------------------- 3.21s
Gathering Facts ------------------------------------------ 2.15s
```

### Common Performance Issues

| Issue | Solution |
|-------|----------|
| Slow fact gathering | `gather_facts: no` or `gather_subset` |
| Sequential execution | Increase `forks` in ansible.cfg |
| SSH overhead | Enable `pipelining = True` |
| Large file transfers | Use `synchronize` instead of `copy` |
| Repeated connections | Use `ssh_args = -o ControlMaster=auto` |

### ansible.cfg Performance Settings

```ini
[defaults]
forks = 20
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

---

## Callback Plugins

### Enable Debugging Callbacks

```ini
# ansible.cfg
[defaults]
# Show task timing
callback_whitelist = timer, profile_tasks, profile_roles

# YAML output (more readable)
stdout_callback = yaml

# Minimal output
# stdout_callback = minimal

# JSON output
# stdout_callback = json
```

### Debug Strategy

```yaml
---
- name: Interactive debug
  hosts: all
  strategy: debug

  tasks:
    - name: This will pause on error
      command: /nonexistent/command
```

Debug commands in debug strategy:
- `p task` - Print task info
- `p vars` - Print variables
- `p host` - Print host info
- `r` - Re-run task
- `c` - Continue
- `q` - Quit

---

## Log Files

### Enable Logging

```ini
# ansible.cfg
[defaults]
log_path = /var/log/ansible/ansible.log
```

```bash
# Or via environment
export ANSIBLE_LOG_PATH=/var/log/ansible/ansible.log
```

### Syslog Integration

```ini
# ansible.cfg
[defaults]
callback_whitelist = syslog_json

[callback_syslog_json]
syslog_facility = LOG_LOCAL0
```

---

## Common Debugging Playbook

```yaml
---
- name: Comprehensive Debug Playbook
  hosts: all
  gather_facts: yes

  tasks:
    - name: Show inventory hostname
      debug:
        var: inventory_hostname

    - name: Show groups
      debug:
        var: group_names

    - name: Show OS info
      debug:
        msg: |
          Distribution: {{ ansible_distribution }}
          Version: {{ ansible_distribution_version }}
          Family: {{ ansible_os_family }}

    - name: Show network info
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          FQDN: {{ ansible_fqdn }}
          IP: {{ ansible_default_ipv4.address | default('N/A') }}

    - name: Show memory
      debug:
        msg: "Total Memory: {{ ansible_memtotal_mb }}MB"

    - name: Show disk space
      debug:
        msg: "{{ item.mount }}: {{ (item.size_available / 1073741824) | round(2) }}GB free"
      loop: "{{ ansible_mounts }}"
      loop_control:
        label: "{{ item.mount }}"
      when: item.size_total > 0

    - name: Test sudo access
      command: whoami
      become: yes
      register: sudo_test
      changed_when: false

    - name: Show sudo result
      debug:
        msg: "Running as: {{ sudo_test.stdout }}"

    - name: Check DNS resolution
      command: "nslookup {{ item }}"
      loop:
        - google.com
        - "{{ inventory_hostname }}"
      register: dns_test
      changed_when: false
      ignore_errors: yes

    - name: Show custom variables
      debug:
        msg: |
          All custom vars:
          {{ hostvars[inventory_hostname] | to_nice_yaml }}
      verbosity: 2
```

---

## Quick Reference

### Useful Commands

```bash
# Check connectivity
ansible all -m ping

# Gather and display facts
ansible hostname -m setup

# Run ad-hoc command
ansible all -a "uptime"

# Syntax check
ansible-playbook playbook.yml --syntax-check

# List tasks
ansible-playbook playbook.yml --list-tasks

# List hosts
ansible-playbook playbook.yml --list-hosts

# Dry run
ansible-playbook playbook.yml --check

# Step through
ansible-playbook playbook.yml --step

# Verbose output
ansible-playbook playbook.yml -vvvv
```

### Environment Variables for Debugging

```bash
export ANSIBLE_DEBUG=1
export ANSIBLE_VERBOSITY=4
export ANSIBLE_LOG_PATH=/tmp/ansible.log
export ANSIBLE_STDOUT_CALLBACK=yaml
export ANSIBLE_CALLBACKS_ENABLED=timer,profile_tasks
```

---

## Summary

| Tool/Flag | Purpose |
|-----------|---------|
| `-v/-vvvv` | Increase verbosity |
| `--check` | Dry run |
| `--diff` | Show changes |
| `--step` | Step through tasks |
| `--syntax-check` | Validate syntax |
| `--list-tasks` | List all tasks |
| `--start-at-task` | Skip to specific task |
| `debug` module | Print variables/messages |
| `strategy: debug` | Interactive debugger |
| `profile_tasks` | Show task timing |

### Troubleshooting Checklist

1. ✓ Check SSH connectivity manually
2. ✓ Verify Python is installed on targets
3. ✓ Check inventory file syntax
4. ✓ Validate playbook syntax
5. ✓ Run with increased verbosity
6. ✓ Use check mode first
7. ✓ Inspect variables with debug module
8. ✓ Check Ansible and module versions
9. ✓ Review ansible.cfg settings
10. ✓ Check logs for errors
