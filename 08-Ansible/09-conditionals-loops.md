# Ansible Conditionals and Loops

## Conditionals with When

The `when` statement allows conditional task execution.

### Basic Syntax

```yaml
tasks:
  - name: Install nginx on Debian
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"

  - name: Install nginx on RedHat
    yum:
      name: nginx
      state: present
    when: ansible_os_family == "RedHat"
```

### Comparison Operators

```yaml
tasks:
  # Equal
  - name: Check if Ubuntu
    debug:
      msg: "This is Ubuntu"
    when: ansible_distribution == "Ubuntu"

  # Not equal
  - name: Not CentOS
    debug:
      msg: "Not CentOS"
    when: ansible_distribution != "CentOS"

  # Greater than / Less than
  - name: Check version
    debug:
      msg: "Version is high enough"
    when: ansible_distribution_major_version | int >= 20

  # Contains (in)
  - name: Check if webserver
    debug:
      msg: "This is a webserver"
    when: "'webservers' in group_names"

  # String matching
  - name: Check hostname pattern
    debug:
      msg: "Production server"
    when: inventory_hostname is match("prod-.*")
```

### Logical Operators

```yaml
tasks:
  # AND - both conditions must be true
  - name: Install on Ubuntu 22.04
    apt:
      name: nginx
    when:
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_version == "22.04"

  # Alternative AND syntax
  - name: Install on Ubuntu 22.04
    apt:
      name: nginx
    when: ansible_distribution == "Ubuntu" and ansible_distribution_version == "22.04"

  # OR - either condition
  - name: Install on Debian-based
    apt:
      name: nginx
    when: ansible_distribution == "Ubuntu" or ansible_distribution == "Debian"

  # NOT - negation
  - name: Skip maintenance hosts
    debug:
      msg: "Active server"
    when: not (inventory_hostname in groups['maintenance'])

  # Complex conditions
  - name: Complex condition
    debug:
      msg: "Matched"
    when: >
      (ansible_distribution == "Ubuntu" and ansible_distribution_major_version | int >= 20)
      or
      (ansible_distribution == "CentOS" and ansible_distribution_major_version | int >= 8)
```

### Testing Variables

```yaml
tasks:
  # Check if defined
  - name: Use variable if defined
    debug:
      msg: "Value: {{ my_var }}"
    when: my_var is defined

  # Check if undefined
  - name: Set default if undefined
    set_fact:
      my_var: "default_value"
    when: my_var is not defined

  # Check if variable is truthy
  - name: Run if enabled
    command: start_service.sh
    when: enable_feature | bool

  # Check if variable is falsy
  - name: Skip if disabled
    command: stop_service.sh
    when: not (enable_feature | default(false) | bool)

  # Check for empty
  - name: Run if list has items
    debug:
      msg: "List has items"
    when: my_list | length > 0

  # Check for none/null
  - name: Check for none
    debug:
      msg: "Variable is set"
    when: my_var is not none
```

### Conditional with Registered Variables

```yaml
tasks:
  - name: Check if file exists
    stat:
      path: /etc/app/config.yml
    register: config_file

  - name: Create config if missing
    template:
      src: config.yml.j2
      dest: /etc/app/config.yml
    when: not config_file.stat.exists

  - name: Run command
    command: /opt/scripts/check.sh
    register: check_result
    ignore_errors: yes

  - name: Handle success
    debug:
      msg: "Check passed"
    when: check_result.rc == 0

  - name: Handle failure
    debug:
      msg: "Check failed: {{ check_result.stderr }}"
    when: check_result.rc != 0
```

### Tests in Conditions

```yaml
tasks:
  # File tests
  - name: Check if directory
    debug:
      msg: "Is a directory"
    when: mypath is directory

  - name: Check if file
    debug:
      msg: "Is a file"
    when: mypath is file

  - name: Check if link
    debug:
      msg: "Is a symlink"
    when: mypath is link

  # String tests
  - name: Check if string
    debug:
      msg: "Is a string"
    when: myvar is string

  # Number tests
  - name: Check if number
    debug:
      msg: "Is a number"
    when: myvar is number

  # Iterable tests
  - name: Check if iterable
    debug:
      msg: "Is iterable"
    when: myvar is iterable

  # Mapping (dict) test
  - name: Check if dict
    debug:
      msg: "Is a dictionary"
    when: myvar is mapping

  # Version comparison
  - name: Check version
    debug:
      msg: "Version OK"
    when: app_version is version('2.0', '>=')
```

---

## Loops

Iterate over lists, dictionaries, and more.

### Basic Loop

```yaml
tasks:
  # Simple loop
  - name: Install packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - php-fpm
      - mysql-client

  # Loop with variable
  - name: Install packages from variable
    apt:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"
    vars:
      packages:
        - vim
        - curl
        - wget
```

### Loop with Dictionaries

```yaml
tasks:
  # Loop over list of dicts
  - name: Create users
    user:
      name: "{{ item.name }}"
      groups: "{{ item.groups }}"
      shell: "{{ item.shell | default('/bin/bash') }}"
    loop:
      - { name: 'alice', groups: 'developers' }
      - { name: 'bob', groups: 'admins' }
      - { name: 'charlie', groups: 'developers,admins' }

  # Using dict2items for dictionary iteration
  - name: Set sysctl values
    sysctl:
      name: "{{ item.key }}"
      value: "{{ item.value }}"
    loop: "{{ sysctl_settings | dict2items }}"
    vars:
      sysctl_settings:
        net.ipv4.ip_forward: 1
        net.ipv4.tcp_syncookies: 1
        vm.swappiness: 10
```

### Loop Control

```yaml
tasks:
  # Custom loop variable name
  - name: Create users
    user:
      name: "{{ user.name }}"
    loop: "{{ users }}"
    loop_control:
      loop_var: user

  # Index variable
  - name: Show index
    debug:
      msg: "{{ index }}: {{ item }}"
    loop:
      - apple
      - banana
      - cherry
    loop_control:
      index_var: index

  # Label for output (cleaner logs)
  - name: Process servers
    debug:
      msg: "Processing {{ item.name }}"
    loop: "{{ servers }}"
    loop_control:
      label: "{{ item.name }}"  # Shows only name in output

  # Pause between iterations
  - name: Restart services one by one
    service:
      name: "{{ item }}"
      state: restarted
    loop:
      - nginx
      - php-fpm
      - mysql
    loop_control:
      pause: 5  # Wait 5 seconds between each

  # Extended loop info
  - name: Show loop info
    debug:
      msg: |
        Item: {{ item }}
        Index: {{ ansible_loop.index }}
        First: {{ ansible_loop.first }}
        Last: {{ ansible_loop.last }}
        Length: {{ ansible_loop.length }}
    loop: [a, b, c]
    loop_control:
      extended: yes
```

### Nested Loops

```yaml
tasks:
  # Using include_tasks for nested loops
  - name: Process users for each server
    include_tasks: user_setup.yml
    loop: "{{ servers }}"
    loop_control:
      loop_var: server

# user_setup.yml
- name: Create users on {{ server }}
  user:
    name: "{{ item }}"
  loop: "{{ users }}"
  delegate_to: "{{ server }}"
```

```yaml
  # Using product filter
  - name: Create all combinations
    debug:
      msg: "Server: {{ item.0 }}, Port: {{ item.1 }}"
    loop: "{{ servers | product(ports) | list }}"
    vars:
      servers: ['web1', 'web2']
      ports: [80, 443, 8080]
```

### Loop with Conditionals

```yaml
tasks:
  # Skip items in loop
  - name: Install if not excluded
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - apache2
      - mysql
    when: item != 'apache2'

  # Complex condition in loop
  - name: Process valid entries
    debug:
      msg: "Processing {{ item.name }}"
    loop: "{{ entries }}"
    when:
      - item.enabled | bool
      - item.name is defined
```

### Loop with Register

```yaml
tasks:
  - name: Check services
    command: "systemctl status {{ item }}"
    loop:
      - nginx
      - mysql
      - redis
    register: service_status
    ignore_errors: yes

  - name: Show failed services
    debug:
      msg: "{{ item.item }} is not running"
    loop: "{{ service_status.results }}"
    when: item.rc != 0
```

---

## Legacy Loop Syntax (with_*)

Still supported but `loop` is preferred.

### with_items

```yaml
# Legacy
- name: Install packages
  apt:
    name: "{{ item }}"
  with_items:
    - nginx
    - php

# Modern equivalent
- name: Install packages
  apt:
    name: "{{ item }}"
  loop:
    - nginx
    - php
```

### with_dict

```yaml
# Legacy
- name: Create users
  user:
    name: "{{ item.key }}"
    comment: "{{ item.value.name }}"
  with_dict: "{{ users }}"

# Modern equivalent
- name: Create users
  user:
    name: "{{ item.key }}"
    comment: "{{ item.value.name }}"
  loop: "{{ users | dict2items }}"
```

### with_fileglob

```yaml
# Legacy
- name: Copy configs
  copy:
    src: "{{ item }}"
    dest: /etc/app/
  with_fileglob:
    - "files/*.conf"

# Modern equivalent
- name: Copy configs
  copy:
    src: "{{ item }}"
    dest: /etc/app/
  loop: "{{ query('fileglob', 'files/*.conf') }}"
```

### with_sequence

```yaml
# Legacy
- name: Create numbered files
  file:
    path: "/tmp/file{{ item }}"
    state: touch
  with_sequence: start=1 end=5

# Modern equivalent
- name: Create numbered files
  file:
    path: "/tmp/file{{ item }}"
    state: touch
  loop: "{{ range(1, 6) | list }}"
```

### with_subelements

```yaml
# Legacy
- name: Add SSH keys for users
  authorized_key:
    user: "{{ item.0.name }}"
    key: "{{ item.1 }}"
  with_subelements:
    - "{{ users }}"
    - ssh_keys

# Modern equivalent
- name: Add SSH keys for users
  authorized_key:
    user: "{{ item.0.name }}"
    key: "{{ item.1 }}"
  loop: "{{ users | subelements('ssh_keys') }}"
```

---

## Until Loops (Retries)

Retry tasks until a condition is met.

```yaml
tasks:
  # Retry until success
  - name: Wait for service to start
    uri:
      url: http://localhost:8080/health
      status_code: 200
    register: result
    until: result.status == 200
    retries: 10
    delay: 5  # seconds between retries

  # Retry command until output matches
  - name: Wait for cluster to be ready
    command: kubectl get nodes
    register: nodes
    until: "'Ready' in nodes.stdout"
    retries: 30
    delay: 10

  # Retry with complex condition
  - name: Wait for all pods
    command: kubectl get pods -o json
    register: pods
    until: >
      (pods.stdout | from_json).items |
      selectattr('status.phase', 'equalto', 'Running') |
      list | length == expected_pods
    retries: 60
    delay: 5
```

---

## Practical Examples

### OS-Specific Package Installation

```yaml
---
- name: Install packages across OS
  hosts: all
  become: yes

  vars:
    packages_debian:
      - nginx
      - php-fpm
      - mysql-client
    packages_redhat:
      - nginx
      - php-fpm
      - mysql

  tasks:
    - name: Install on Debian
      apt:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop: "{{ packages_debian }}"
      when: ansible_os_family == "Debian"

    - name: Install on RedHat
      yum:
        name: "{{ item }}"
        state: present
      loop: "{{ packages_redhat }}"
      when: ansible_os_family == "RedHat"
```

### User Management with Loops

```yaml
---
- name: Manage users
  hosts: all
  become: yes

  vars:
    users:
      - name: alice
        groups: developers
        ssh_keys:
          - "ssh-rsa AAAA... alice@laptop"
          - "ssh-rsa AAAA... alice@desktop"
        state: present
      - name: bob
        groups: admins
        ssh_keys:
          - "ssh-rsa AAAA... bob@laptop"
        state: present
      - name: charlie
        state: absent

  tasks:
    - name: Create or remove users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups | default(omit) }}"
        state: "{{ item.state }}"
        remove: "{{ 'yes' if item.state == 'absent' else 'no' }}"
      loop: "{{ users }}"

    - name: Add SSH keys
      authorized_key:
        user: "{{ item.0.name }}"
        key: "{{ item.1 }}"
        state: present
      loop: "{{ users | selectattr('state', 'equalto', 'present') | subelements('ssh_keys', skip_missing=True) }}"
```

### Conditional Service Management

```yaml
---
- name: Manage services
  hosts: all
  become: yes

  vars:
    services:
      - name: nginx
        enabled: true
        state: started
        required_for:
          - webservers
      - name: mysql
        enabled: true
        state: started
        required_for:
          - databases
      - name: redis
        enabled: "{{ enable_caching | default(false) }}"
        state: "{{ 'started' if enable_caching | default(false) else 'stopped' }}"

  tasks:
    - name: Manage services
      service:
        name: "{{ item.name }}"
        enabled: "{{ item.enabled }}"
        state: "{{ item.state }}"
      loop: "{{ services }}"
      when: >
        item.required_for is not defined or
        (group_names | intersect(item.required_for) | length > 0)
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **when** | Conditional task execution |
| **loop** | Iterate over lists |
| **loop_control** | Customize loop behavior |
| **until** | Retry until condition met |
| **retries/delay** | Retry configuration |
| **register** | Capture loop results |
| **dict2items** | Convert dict for looping |
| **subelements** | Loop over nested lists |

### Operators Quick Reference

| Operator | Example |
|----------|---------|
| `==` | `when: var == "value"` |
| `!=` | `when: var != "value"` |
| `>`, `<`, `>=`, `<=` | `when: count > 5` |
| `and` | `when: a and b` |
| `or` | `when: a or b` |
| `not` | `when: not condition` |
| `in` | `when: item in list` |
| `is defined` | `when: var is defined` |
| `is match` | `when: var is match("pattern")` |

### Best Practices

1. Use `loop` instead of `with_*` (modern syntax)
2. Use `loop_control` for cleaner output
3. Combine conditions with list syntax for readability
4. Use `until` for polling/waiting scenarios
5. Register loop results for post-processing
