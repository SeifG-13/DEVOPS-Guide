# Ansible Playbooks Advanced

## Tags

Tags allow selective execution of tasks.

### Defining Tags

```yaml
---
- name: Web Server Setup
  hosts: webservers
  become: yes

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
      tags:
        - install
        - packages
        - nginx

    - name: Copy configuration
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      tags:
        - configure
        - nginx

    - name: Start nginx
      service:
        name: nginx
        state: started
      tags:
        - service
        - nginx

    # Multiple tags inline
    - name: Deploy application
      copy:
        src: app/
        dest: /var/www/app/
      tags: [deploy, app]
```

### Using Tags

```bash
# Run only specific tags
ansible-playbook playbook.yml --tags "install"
ansible-playbook playbook.yml --tags "install,configure"

# Skip specific tags
ansible-playbook playbook.yml --skip-tags "deploy"

# List all tags
ansible-playbook playbook.yml --list-tags

# Run untagged tasks only
ansible-playbook playbook.yml --tags untagged

# Run all tasks (including untagged)
ansible-playbook playbook.yml --tags all
```

### Special Tags

```yaml
tasks:
  # Always runs regardless of tag selection
  - name: Gather custom facts
    setup:
    tags: always

  # Never runs unless explicitly called
  - name: Dangerous operation
    command: rm -rf /old_data
    tags: never
```

### Tag Inheritance

```yaml
# Tags on blocks are inherited by tasks
- name: Database setup
  block:
    - name: Install MySQL
      apt:
        name: mysql-server

    - name: Start MySQL
      service:
        name: mysql
        state: started
  tags: database  # Both tasks get this tag
```

---

## Includes and Imports

### Task Includes (Dynamic)

```yaml
# main.yml
---
- name: Setup Server
  hosts: all
  tasks:
    - name: Include common tasks
      include_tasks: tasks/common.yml

    - name: Include OS-specific tasks
      include_tasks: "tasks/{{ ansible_os_family }}.yml"

    - name: Include tasks with variables
      include_tasks: tasks/app.yml
      vars:
        app_name: myapp
        app_port: 8080
```

```yaml
# tasks/common.yml
---
- name: Install common packages
  apt:
    name:
      - vim
      - curl
    state: present

- name: Configure timezone
  timezone:
    name: UTC
```

### Task Imports (Static)

```yaml
# main.yml
---
- name: Setup Server
  hosts: all
  tasks:
    # Static import - processed at playbook parse time
    - import_tasks: tasks/common.yml

    # With tags (applied to all imported tasks)
    - import_tasks: tasks/security.yml
      tags: security
```

### Difference: include_tasks vs import_tasks

| Feature | include_tasks | import_tasks |
|---------|--------------|--------------|
| Processing | Runtime (dynamic) | Parse time (static) |
| Variables | Can use variables in filename | Cannot use variables |
| Conditionals | Applied to include itself | Applied to each task |
| Loops | Can be used with loops | Cannot be used with loops |
| Tags | Must use `apply` keyword | Tags work directly |
| `--list-tasks` | Not shown | Shown |

### Include with Loop

```yaml
- name: Include tasks for each user
  include_tasks: tasks/create_user.yml
  loop:
    - alice
    - bob
    - charlie
  loop_control:
    loop_var: username
```

### Playbook Imports

```yaml
# site.yml (main playbook)
---
- import_playbook: playbooks/common.yml
- import_playbook: playbooks/webservers.yml
- import_playbook: playbooks/databases.yml

# Conditional import
- import_playbook: playbooks/monitoring.yml
  when: deploy_monitoring | default(true)
```

---

## Blocks

Blocks group tasks and allow shared attributes.

### Basic Block

```yaml
tasks:
  - name: Web server configuration
    block:
      - name: Install nginx
        apt:
          name: nginx
          state: present

      - name: Copy config
        copy:
          src: nginx.conf
          dest: /etc/nginx/nginx.conf

      - name: Start nginx
        service:
          name: nginx
          state: started
    become: yes  # Applied to all tasks in block
    when: "'webserver' in group_names"  # Condition for all
```

### Block Error Handling

```yaml
tasks:
  - name: Handle errors gracefully
    block:
      - name: Attempt risky operation
        command: /opt/scripts/risky_script.sh

      - name: This runs if above succeeds
        debug:
          msg: "Script succeeded"

    rescue:
      - name: This runs if block fails
        debug:
          msg: "Script failed, running recovery"

      - name: Recovery action
        command: /opt/scripts/recovery.sh

    always:
      - name: This always runs
        debug:
          msg: "Cleanup complete"

      - name: Send notification
        mail:
          to: admin@example.com
          subject: "Deployment status"
          body: "Deployment finished"
```

### Nested Blocks

```yaml
tasks:
  - name: Outer block
    block:
      - name: Inner block
        block:
          - name: Nested task
            command: echo "Deep nesting"
        rescue:
          - debug:
              msg: "Inner rescue"
    rescue:
      - debug:
          msg: "Outer rescue"
```

---

## Delegation

Run tasks on different hosts than the play targets.

### delegate_to

```yaml
tasks:
  # Run on localhost instead of target
  - name: Add host to load balancer
    command: /scripts/add_to_lb.sh {{ inventory_hostname }}
    delegate_to: localhost

  # Run on specific host
  - name: Update DNS record
    nsupdate:
      server: dns.example.com
      zone: example.com
      record: "{{ inventory_hostname }}"
      value: "{{ ansible_default_ipv4.address }}"
    delegate_to: dns.example.com

  # Delegate to another group member
  - name: Sync files from primary
    synchronize:
      src: /var/www/
      dest: /var/www/
    delegate_to: "{{ groups['webservers'][0] }}"
```

### local_action

```yaml
tasks:
  # Shorthand for delegate_to: localhost
  - name: Wait for server to come up
    local_action:
      module: wait_for
      host: "{{ inventory_hostname }}"
      port: 22
      delay: 10
      timeout: 300

  # Equivalent to
  - name: Wait for server
    wait_for:
      host: "{{ inventory_hostname }}"
      port: 22
    delegate_to: localhost
```

### Delegation with Facts

```yaml
tasks:
  # By default, delegated tasks use delegated host's facts
  - name: Show IP of delegate
    debug:
      var: ansible_default_ipv4.address
    delegate_to: other_host
    delegate_facts: true  # Store facts for delegate host
```

---

## Run Once

Execute task only once for the entire play.

```yaml
tasks:
  # Run only on first host
  - name: Initialize cluster
    command: /opt/cluster/init.sh
    run_once: true

  # Run once on specific host
  - name: Create shared resource
    command: create_resource.sh
    run_once: true
    delegate_to: primary.example.com

  # Run once per group
  - name: Register with master
    command: register.sh
    run_once: true
    delegate_to: "{{ groups['masters'][0] }}"
```

---

## Serial Execution (Rolling Updates)

Control how many hosts run at a time.

### Basic Serial

```yaml
---
- name: Rolling Update
  hosts: webservers
  serial: 1  # One host at a time

  tasks:
    - name: Disable in load balancer
      command: lb_remove.sh {{ inventory_hostname }}
      delegate_to: localhost

    - name: Update application
      apt:
        name: myapp
        state: latest

    - name: Enable in load balancer
      command: lb_add.sh {{ inventory_hostname }}
      delegate_to: localhost
```

### Serial Options

```yaml
# Fixed number
serial: 2

# Percentage
serial: "25%"

# Batch sizes (increasing)
serial:
  - 1      # First batch: 1 host
  - 5      # Second batch: 5 hosts
  - "25%"  # Remaining: 25% at a time

# Combination
serial:
  - 1
  - 10%
  - 100%
```

### Max Fail Percentage

```yaml
---
- name: Deploy with failure tolerance
  hosts: webservers
  serial: "25%"
  max_fail_percentage: 10  # Stop if >10% fail

  tasks:
    - name: Deploy application
      command: deploy.sh
```

---

## Async Tasks

Run long-running tasks without blocking.

### Fire and Forget

```yaml
tasks:
  # Start and don't wait
  - name: Start long backup
    command: /opt/backup/full_backup.sh
    async: 3600  # Timeout after 1 hour
    poll: 0      # Don't wait (fire and forget)
```

### Async with Polling

```yaml
tasks:
  # Start and check periodically
  - name: Run database migration
    command: /opt/scripts/migrate.sh
    async: 1800   # Timeout after 30 minutes
    poll: 30      # Check every 30 seconds
    register: migration_result
```

### Check Async Status

```yaml
tasks:
  - name: Start long operation
    command: /opt/scripts/long_task.sh
    async: 3600
    poll: 0
    register: long_task

  - name: Do other work
    command: echo "Other tasks..."

  - name: Check on long task
    async_status:
      jid: "{{ long_task.ansible_job_id }}"
    register: job_result
    until: job_result.finished
    retries: 60
    delay: 30
```

### Multiple Async Tasks

```yaml
tasks:
  # Start multiple tasks
  - name: Start backups on all servers
    command: "/opt/backup/backup.sh {{ item }}"
    async: 7200
    poll: 0
    loop:
      - database
      - files
      - logs
    register: backup_jobs

  - name: Wait for all backups
    async_status:
      jid: "{{ item.ansible_job_id }}"
    loop: "{{ backup_jobs.results }}"
    register: job_results
    until: job_results.finished
    retries: 120
    delay: 60
```

---

## Strategy Plugins

Control how Ansible executes tasks across hosts.

### Linear Strategy (Default)

```yaml
---
- name: Linear execution
  hosts: all
  strategy: linear  # Wait for all hosts before next task

  tasks:
    - name: Task 1
      command: echo "First"
    # All hosts complete Task 1 before Task 2 starts
    - name: Task 2
      command: echo "Second"
```

### Free Strategy

```yaml
---
- name: Free execution
  hosts: all
  strategy: free  # Each host runs as fast as possible

  tasks:
    - name: Task 1
      command: echo "First"
    # Hosts don't wait for each other
    - name: Task 2
      command: echo "Second"
```

### Host Pinned Strategy

```yaml
---
- name: Host pinned execution
  hosts: all
  strategy: host_pinned  # One worker per host

  tasks:
    - name: Run tasks
      command: echo "Pinned to host"
```

### Debug Strategy

```yaml
---
- name: Interactive debugging
  hosts: all
  strategy: debug  # Interactive debugger

  tasks:
    - name: Debug this
      command: might_fail.sh
```

---

## Throttling

Limit concurrent execution.

```yaml
tasks:
  # Limit concurrent tasks
  - name: API call with rate limiting
    uri:
      url: "https://api.example.com/update"
      method: POST
    throttle: 5  # Max 5 concurrent

  # Limit at play level
---
- name: Limited play
  hosts: all
  throttle: 10  # Max 10 hosts at once
  tasks:
    - name: Resource intensive task
      command: heavy_operation.sh
```

---

## Practical Examples

### Zero-Downtime Deployment

```yaml
---
- name: Zero-Downtime Deployment
  hosts: webservers
  serial: 1
  max_fail_percentage: 0

  tasks:
    - name: Disable server in load balancer
      uri:
        url: "http://lb.example.com/api/servers/{{ inventory_hostname }}/disable"
        method: POST
      delegate_to: localhost

    - name: Wait for connections to drain
      wait_for:
        timeout: 30
      delegate_to: localhost

    - name: Stop application
      service:
        name: myapp
        state: stopped

    - name: Deploy new version
      unarchive:
        src: "https://releases.example.com/myapp-{{ version }}.tar.gz"
        dest: /opt/myapp
        remote_src: yes

    - name: Run migrations
      command: /opt/myapp/migrate.sh
      run_once: true

    - name: Start application
      service:
        name: myapp
        state: started

    - name: Health check
      uri:
        url: "http://localhost:8080/health"
        status_code: 200
      retries: 10
      delay: 5
      until: health_check is success
      register: health_check

    - name: Enable server in load balancer
      uri:
        url: "http://lb.example.com/api/servers/{{ inventory_hostname }}/enable"
        method: POST
      delegate_to: localhost
```

### Conditional Include with Error Handling

```yaml
---
- name: Multi-OS Setup
  hosts: all
  become: yes

  tasks:
    - name: OS-specific setup
      block:
        - name: Include OS tasks
          include_tasks: "tasks/{{ ansible_os_family | lower }}.yml"

        - name: Verify installation
          command: verify_install.sh
          register: verify

      rescue:
        - name: Log failure
          debug:
            msg: "Setup failed for {{ inventory_hostname }}"

        - name: Attempt recovery
          include_tasks: tasks/recovery.yml

      always:
        - name: Report status
          debug:
            msg: "Host {{ inventory_hostname }} - {{ 'OK' if verify is success else 'FAILED' }}"
```

---

## Summary

| Feature | Description |
|---------|-------------|
| **Tags** | Selective task execution |
| **include_tasks** | Dynamic runtime inclusion |
| **import_tasks** | Static parse-time import |
| **Blocks** | Group tasks, error handling |
| **delegate_to** | Run on different host |
| **run_once** | Execute once for play |
| **serial** | Rolling updates |
| **async/poll** | Background tasks |
| **strategy** | Execution behavior |
| **throttle** | Limit concurrency |

### Best Practices

1. Use tags for flexible playbook execution
2. Prefer `import_tasks` for static includes, `include_tasks` for dynamic
3. Use blocks for error handling and shared attributes
4. Implement rolling updates with `serial` for production
5. Use `async` for long-running tasks to avoid timeouts
