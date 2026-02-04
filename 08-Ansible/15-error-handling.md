# Ansible Error Handling

## Default Error Behavior

By default, Ansible stops executing on a host when a task fails and continues with other hosts.

```
┌─────────────────────────────────────────────────────────────┐
│                 DEFAULT ERROR BEHAVIOR                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Task 1: OK ──► Task 2: OK ──► Task 3: FAILED               │
│                                        │                    │
│                                        ▼                    │
│                               Host stops here               │
│                               Other hosts continue          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Ignoring Errors

### ignore_errors

Continue execution even if task fails.

```yaml
tasks:
  - name: This might fail
    command: /opt/scripts/might_fail.sh
    ignore_errors: yes

  - name: This always runs
    debug:
      msg: "Previous task's failure was ignored"
```

### ignore_unreachable

Continue even if host becomes unreachable.

```yaml
tasks:
  - name: Reboot server
    reboot:
      reboot_timeout: 300
    ignore_unreachable: yes

  - name: Continue after reboot
    wait_for_connection:
      timeout: 300
```

---

## Custom Failure Conditions

### failed_when

Define custom failure conditions.

```yaml
tasks:
  # Command that returns non-zero but isn't a failure
  - name: Check application status
    command: /opt/app/healthcheck.sh
    register: health_result
    failed_when: health_result.rc > 1  # Only fail if rc > 1

  # Fail based on output content
  - name: Check for errors in output
    command: /opt/scripts/check.sh
    register: check_result
    failed_when: "'ERROR' in check_result.stdout"

  # Multiple conditions
  - name: Complex failure condition
    command: /opt/scripts/validate.sh
    register: result
    failed_when:
      - result.rc != 0
      - "'WARNING' not in result.stdout"

  # Never fail (careful!)
  - name: Always succeed
    command: /opt/scripts/optional.sh
    failed_when: false
```

### changed_when

Define when a task is considered "changed".

```yaml
tasks:
  # Commands typically always report changed
  - name: Check disk space
    command: df -h /
    register: disk_space
    changed_when: false  # Read-only, never changed

  # Custom changed condition
  - name: Run migration
    command: /opt/app/migrate.sh
    register: migrate_result
    changed_when: "'Migrations applied' in migrate_result.stdout"

  # Based on return code
  - name: Update config
    command: /opt/scripts/update_config.sh
    register: result
    changed_when: result.rc == 0  # Changed only on success
```

---

## Block Error Handling

Blocks provide try-catch-finally style error handling.

### Basic Block with Rescue

```yaml
tasks:
  - name: Handle deployment errors
    block:
      - name: Deploy application
        copy:
          src: app/
          dest: /opt/app/

      - name: Run migrations
        command: /opt/app/migrate.sh

      - name: Start application
        service:
          name: myapp
          state: started

    rescue:
      - name: Rollback on failure
        copy:
          src: /opt/app.backup/
          dest: /opt/app/

      - name: Notify admin
        mail:
          to: admin@example.com
          subject: "Deployment failed"
          body: "Deployment to {{ inventory_hostname }} failed"

    always:
      - name: Cleanup temp files
        file:
          path: /tmp/deploy_temp
          state: absent

      - name: Log deployment attempt
        lineinfile:
          path: /var/log/deployments.log
          line: "{{ ansible_date_time.iso8601 }} - Deployment attempted"
```

### Block Variables

```yaml
tasks:
  - name: Web server setup
    block:
      - name: Install nginx
        apt:
          name: nginx
          state: present

      - name: Start nginx
        service:
          name: nginx
          state: started
    become: yes  # Applied to all tasks in block
    when: "'webservers' in group_names"  # Condition for entire block
```

### Accessing Failure Info in Rescue

```yaml
tasks:
  - name: Risky operation
    block:
      - name: This might fail
        command: /opt/scripts/risky.sh
        register: risky_result

    rescue:
      - name: Show error details
        debug:
          msg: |
            Task '{{ ansible_failed_task.name }}' failed
            Result: {{ ansible_failed_result }}

      - name: Handle specific errors
        debug:
          msg: "Handling connection error"
        when: "'connection refused' in ansible_failed_result.msg | default('')"
```

---

## Any Errors Fatal

Stop all hosts immediately on any failure.

### Play Level

```yaml
---
- name: Critical deployment
  hosts: all
  any_errors_fatal: true  # Stop everything if any host fails

  tasks:
    - name: Critical task
      command: /opt/scripts/critical.sh
```

### Task Level

```yaml
tasks:
  - name: Critical check
    command: /opt/scripts/preflight_check.sh
    any_errors_fatal: true  # Stop all hosts if this fails

  - name: Continue with deployment
    command: /opt/scripts/deploy.sh
```

---

## Max Fail Percentage

Allow a percentage of hosts to fail before stopping.

```yaml
---
- name: Rolling update
  hosts: webservers
  serial: "25%"
  max_fail_percentage: 10  # Stop if >10% of hosts fail

  tasks:
    - name: Update application
      apt:
        name: myapp
        state: latest

    - name: Restart application
      service:
        name: myapp
        state: restarted
```

---

## Assertions

Validate conditions before proceeding.

### assert Module

```yaml
tasks:
  # Simple assertion
  - name: Verify disk space
    assert:
      that:
        - ansible_mounts | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first > 1073741824
      fail_msg: "Insufficient disk space on /"
      success_msg: "Disk space OK"

  # Multiple conditions
  - name: Validate environment
    assert:
      that:
        - environment is defined
        - environment in ['development', 'staging', 'production']
        - db_host is defined
        - db_host | length > 0
      fail_msg: "Invalid environment configuration"

  # Assert with quiet (no success message)
    - name: Quick check
      assert:
        that: ansible_distribution == "Ubuntu"
        quiet: true
```

### fail Module

```yaml
tasks:
  - name: Check prerequisites
    command: /opt/scripts/check_prereqs.sh
    register: prereq_check
    ignore_errors: yes

  - name: Fail if prerequisites not met
    fail:
      msg: "Prerequisites not met: {{ prereq_check.stderr }}"
    when: prereq_check.rc != 0

  # Conditional fail with custom message
  - name: Fail on specific condition
    fail:
      msg: "Cannot deploy to production without approval"
    when:
      - environment == 'production'
      - not deployment_approved | default(false)
```

---

## Debug Module

Print debug information for troubleshooting.

```yaml
tasks:
  - name: Show variable value
    debug:
      var: my_variable

  - name: Show message
    debug:
      msg: "The value is {{ my_variable }}"

  - name: Verbose debug (only with -v)
    debug:
      msg: "Detailed info"
      verbosity: 1

  - name: Very verbose (only with -vv)
    debug:
      var: ansible_facts
      verbosity: 2

  - name: Show registered result
    command: whoami
    register: result

  - debug:
      var: result

  - debug:
      msg: "User is {{ result.stdout }}"
```

---

## Retry Mechanisms

### until Loop

```yaml
tasks:
  # Retry until success
  - name: Wait for service to be ready
    uri:
      url: http://localhost:8080/health
      status_code: 200
    register: result
    until: result.status == 200
    retries: 30
    delay: 10  # seconds between retries

  # Retry command until output matches
  - name: Wait for cluster
    command: kubectl get nodes -o json
    register: nodes
    until: nodes.stdout | from_json | json_query('items[*].status.conditions[?type==`Ready`].status') | select('equalto', 'True') | list | length >= 3
    retries: 60
    delay: 5

  # With failure handling
  - name: Retry with failure
    command: /opt/scripts/flaky_script.sh
    register: result
    until: result.rc == 0
    retries: 5
    delay: 2
    ignore_errors: yes

  - name: Handle retry failure
    fail:
      msg: "Script failed after all retries"
    when: result.rc != 0
```

---

## Practical Examples

### Safe Deployment with Rollback

```yaml
---
- name: Safe Deployment
  hosts: webservers
  become: yes
  serial: 1
  max_fail_percentage: 0

  vars:
    app_dir: /opt/myapp
    backup_dir: /opt/myapp.backup

  tasks:
    - name: Deployment process
      block:
        - name: Create backup
          synchronize:
            src: "{{ app_dir }}/"
            dest: "{{ backup_dir }}/"
          delegate_to: "{{ inventory_hostname }}"

        - name: Stop application
          service:
            name: myapp
            state: stopped

        - name: Deploy new version
          unarchive:
            src: "releases/myapp-{{ version }}.tar.gz"
            dest: "{{ app_dir }}"

        - name: Run migrations
          command: "{{ app_dir }}/bin/migrate"
          register: migration

        - name: Start application
          service:
            name: myapp
            state: started

        - name: Health check
          uri:
            url: "http://localhost:{{ app_port }}/health"
            status_code: 200
          register: health
          until: health.status == 200
          retries: 30
          delay: 2

      rescue:
        - name: Log failure
          debug:
            msg: "Deployment failed: {{ ansible_failed_task.name }}"

        - name: Stop failed application
          service:
            name: myapp
            state: stopped
          ignore_errors: yes

        - name: Restore backup
          synchronize:
            src: "{{ backup_dir }}/"
            dest: "{{ app_dir }}/"
          delegate_to: "{{ inventory_hostname }}"

        - name: Start restored application
          service:
            name: myapp
            state: started

        - name: Fail the play
          fail:
            msg: "Deployment failed and was rolled back"

      always:
        - name: Cleanup backup
          file:
            path: "{{ backup_dir }}"
            state: absent
          when: health is defined and health.status == 200
```

### Pre-flight Checks

```yaml
---
- name: Pre-flight Checks
  hosts: all
  gather_facts: yes

  tasks:
    - name: Check OS compatibility
      assert:
        that:
          - ansible_os_family in ['Debian', 'RedHat']
          - ansible_distribution_major_version | int >= 20
        fail_msg: "Unsupported OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Check disk space
      assert:
        that:
          - item.size_available > 5368709120  # 5GB
        fail_msg: "Insufficient space on {{ item.mount }}"
      loop: "{{ ansible_mounts | selectattr('mount', 'in', ['/', '/var', '/opt']) | list }}"
      loop_control:
        label: "{{ item.mount }}"

    - name: Check memory
      assert:
        that:
          - ansible_memtotal_mb >= 2048
        fail_msg: "Insufficient memory: {{ ansible_memtotal_mb }}MB (need 2048MB)"

    - name: Check connectivity to dependencies
      wait_for:
        host: "{{ item.host }}"
        port: "{{ item.port }}"
        timeout: 5
      loop:
        - { host: "db.example.com", port: 5432 }
        - { host: "redis.example.com", port: 6379 }
      loop_control:
        label: "{{ item.host }}:{{ item.port }}"
```

---

## Summary

| Technique | Use Case |
|-----------|----------|
| `ignore_errors` | Continue on expected failures |
| `failed_when` | Custom failure conditions |
| `changed_when` | Custom change reporting |
| `block/rescue/always` | Try-catch-finally pattern |
| `any_errors_fatal` | Stop all on any failure |
| `max_fail_percentage` | Partial failure tolerance |
| `assert` | Validate conditions |
| `fail` | Explicitly fail with message |
| `until/retries/delay` | Retry mechanism |
| `debug` | Print troubleshooting info |

### Best Practices

1. Use `block/rescue` for critical operations with rollback
2. Always set `changed_when: false` for read-only commands
3. Use `assert` for pre-flight validation
4. Set appropriate `max_fail_percentage` for rolling updates
5. Use `until` for operations that may take time
6. Include meaningful error messages in `fail_msg`
