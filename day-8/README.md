# Day 8: Error Handling in Ansible

## Introduction

Error handling is crucial for building robust and reliable Ansible automation. By default, Ansible stops execution when a task fails. However, you often need more control over how failures are handled, reported, and recovered from.

This guide covers all the techniques for managing errors in your playbooks.

---

## Default Behavior

By default, when a task fails:

1. Ansible stops executing on that host
2. The host is removed from the list of active hosts
3. Other hosts continue executing (unless `any_errors_fatal` is set)
4. Handlers are NOT run for the failed host

```yaml
---
- name: Default error behavior
  hosts: all
  
  tasks:
    - name: This might fail
      ansible.builtin.command: /bin/false
      
    - name: This won't run if above fails
      ansible.builtin.debug:
        msg: "This task is skipped on failed hosts"
```

---

## Ignoring Errors

### ignore_errors

Continue execution even if a task fails:

```yaml
tasks:
  - name: This task might fail
    ansible.builtin.command: /bin/false
    ignore_errors: yes
    
  - name: This will still run
    ansible.builtin.debug:
      msg: "Continued despite the error"
```

### ignore_unreachable

Continue even if hosts become unreachable:

```yaml
tasks:
  - name: Task that might make host unreachable
    ansible.builtin.command: /sbin/shutdown -r now
    ignore_unreachable: yes
    
  - name: This runs even if host is unreachable
    ansible.builtin.wait_for_connection:
      timeout: 300
```

---

## Defining Failure

### failed_when

Define custom conditions for when a task should be considered failed:

```yaml
tasks:
  # Fail based on return code
  - name: Run script
    ansible.builtin.command: /opt/scripts/check.sh
    register: result
    failed_when: result.rc != 0 and result.rc != 2
    
  # Fail based on output content
  - name: Check application status
    ansible.builtin.command: /opt/app/status.sh
    register: app_status
    failed_when: "'ERROR' in app_status.stdout"
    
  # Multiple conditions
  - name: Complex failure check
    ansible.builtin.shell: /opt/scripts/validate.sh
    register: validation
    failed_when:
      - validation.rc != 0
      - "'CRITICAL' in validation.stderr"
      
  # Never fail (always succeeds)
  - name: Always succeed
    ansible.builtin.command: might-fail-command
    failed_when: false
```

### Combining with ignore_errors

```yaml
tasks:
  - name: Run with custom failure detection
    ansible.builtin.shell: |
      /opt/scripts/process.sh
      echo "EXIT_CODE=$?"
    register: result
    ignore_errors: yes
    
  - name: Handle the result
    ansible.builtin.debug:
      msg: "Process {{ 'succeeded' if result.rc == 0 else 'failed' }}"
```

---

## Defining Change

### changed_when

Control when a task reports as "changed":

```yaml
tasks:
  # Never report as changed
  - name: Check disk space
    ansible.builtin.command: df -h /
    register: disk_space
    changed_when: false
    
  # Change based on output
  - name: Run migration
    ansible.builtin.command: /opt/app/migrate.sh
    register: migration
    changed_when: "'Migrations applied' in migration.stdout"
    
  # Multiple conditions
  - name: Update configuration
    ansible.builtin.shell: /opt/scripts/update-config.sh
    register: config_update
    changed_when:
      - config_update.rc == 0
      - "'Updated' in config_update.stdout"
      
  # Always report as changed
  - name: Force change notification
    ansible.builtin.command: /opt/scripts/deploy.sh
    changed_when: true
```

### Combining failed_when and changed_when

```yaml
tasks:
  - name: Complex task with custom status
    ansible.builtin.shell: /opt/scripts/sync.sh
    register: sync_result
    changed_when: sync_result.rc == 0 and 'files synchronized' in sync_result.stdout
    failed_when: sync_result.rc not in [0, 2]  # 2 means "no changes needed"
```

---

## Block Error Handling

Blocks provide try/catch/finally style error handling:

### Basic Block Structure

```yaml
tasks:
  - name: Handle errors with blocks
    block:
      - name: Task that might fail
        ansible.builtin.command: /opt/scripts/risky-operation.sh
        
      - name: Another task in the block
        ansible.builtin.copy:
          src: config.yml
          dest: /etc/app/config.yml
          
    rescue:
      - name: This runs if block fails
        ansible.builtin.debug:
          msg: "An error occurred, running recovery"
          
      - name: Cleanup or recovery task
        ansible.builtin.command: /opt/scripts/rollback.sh
        
    always:
      - name: This always runs
        ansible.builtin.debug:
          msg: "Block completed (success or failure)"
```

### Practical Block Example

```yaml
---
- name: Deploy Application with Error Handling
  hosts: webservers
  become: yes
  
  tasks:
    - name: Application deployment
      block:
        - name: Stop application
          ansible.builtin.service:
            name: myapp
            state: stopped
            
        - name: Backup current version
          ansible.builtin.archive:
            path: /opt/myapp
            dest: /opt/backups/myapp-{{ ansible_date_time.epoch }}.tar.gz
            
        - name: Deploy new version
          ansible.builtin.unarchive:
            src: myapp-latest.tar.gz
            dest: /opt/myapp
            
        - name: Run database migrations
          ansible.builtin.command: /opt/myapp/bin/migrate
          
        - name: Start application
          ansible.builtin.service:
            name: myapp
            state: started
            
        - name: Health check
          ansible.builtin.uri:
            url: http://localhost:8080/health
            status_code: 200
          retries: 5
          delay: 10
          
      rescue:
        - name: Deployment failed - rolling back
          ansible.builtin.debug:
            msg: "Deployment failed! Initiating rollback..."
            
        - name: Stop failed deployment
          ansible.builtin.service:
            name: myapp
            state: stopped
          ignore_errors: yes
          
        - name: Restore from backup
          ansible.builtin.unarchive:
            src: /opt/backups/myapp-latest.tar.gz
            dest: /opt/myapp
            remote_src: yes
            
        - name: Start previous version
          ansible.builtin.service:
            name: myapp
            state: started
            
        - name: Notify team of failure
          ansible.builtin.mail:
            to: team@example.com
            subject: "Deployment failed on {{ inventory_hostname }}"
            body: "Deployment failed and was rolled back."
          delegate_to: localhost
          
      always:
        - name: Cleanup temporary files
          ansible.builtin.file:
            path: /tmp/deploy-temp
            state: absent
            
        - name: Log deployment attempt
          ansible.builtin.lineinfile:
            path: /var/log/deployments.log
            line: "{{ ansible_date_time.iso8601 }} - Deployment {{ 'succeeded' if ansible_failed_task is not defined else 'failed' }}"
            create: yes
```

### Nested Blocks

```yaml
tasks:
  - name: Outer block
    block:
      - name: Inner block for database
        block:
          - name: Database operation
            ansible.builtin.command: /opt/db/migrate.sh
        rescue:
          - name: Handle database error
            ansible.builtin.debug:
              msg: "Database operation failed"
              
      - name: Inner block for cache
        block:
          - name: Cache operation
            ansible.builtin.command: /opt/cache/warm.sh
        rescue:
          - name: Handle cache error
            ansible.builtin.debug:
              msg: "Cache operation failed"
              
    rescue:
      - name: Handle overall failure
        ansible.builtin.debug:
          msg: "Overall operation failed"
```

### Block Variables

Variables defined in block/rescue/always are scoped:

```yaml
tasks:
  - name: Block with variables
    block:
      - name: Set fact in block
        ansible.builtin.set_fact:
          deployment_status: started
          
      - name: Risky task
        ansible.builtin.command: /opt/deploy.sh
        
      - name: Update status
        ansible.builtin.set_fact:
          deployment_status: completed
          
    rescue:
      - name: Mark as failed
        ansible.builtin.set_fact:
          deployment_status: failed
          
    always:
      - name: Report status
        ansible.builtin.debug:
          msg: "Deployment status: {{ deployment_status }}"
```

---

## Handling Unreachable Hosts

### max_fail_percentage

Fail the play if too many hosts fail:

```yaml
---
- name: Rolling update
  hosts: webservers
  max_fail_percentage: 30  # Fail if more than 30% of hosts fail
  serial: 5
  
  tasks:
    - name: Update application
      ansible.builtin.apt:
        name: myapp
        state: latest
```

### any_errors_fatal

Stop entire play if ANY host fails:

```yaml
---
- name: Critical operation
  hosts: all
  any_errors_fatal: true
  
  tasks:
    - name: Critical task
      ansible.builtin.command: /opt/critical-operation.sh
      # If this fails on ANY host, entire play stops
```

### Per-Task any_errors_fatal

```yaml
tasks:
  - name: Non-critical task
    ansible.builtin.command: /opt/optional.sh
    ignore_errors: yes
    
  - name: Critical checkpoint
    ansible.builtin.command: /opt/validate-state.sh
    any_errors_fatal: true  # Stop everything if this fails
    
  - name: Continue with deployment
    ansible.builtin.command: /opt/deploy.sh
```

---

## Retry and Until

### Retry Failed Tasks

```yaml
tasks:
  # Basic retry
  - name: Wait for service to start
    ansible.builtin.uri:
      url: http://localhost:8080/health
      status_code: 200
    register: result
    until: result.status == 200
    retries: 10
    delay: 5  # seconds between retries
    
  # Retry with command
  - name: Wait for database
    ansible.builtin.command: pg_isready -h localhost
    register: pg_ready
    until: pg_ready.rc == 0
    retries: 30
    delay: 2
    
  # Complex retry condition
  - name: Wait for cluster sync
    ansible.builtin.shell: /opt/cluster/status.sh
    register: cluster_status
    until:
      - cluster_status.rc == 0
      - "'SYNCHRONIZED' in cluster_status.stdout"
    retries: 60
    delay: 10
```

### Retry Examples

```yaml
tasks:
  # Wait for file to exist
  - name: Wait for lock file to be removed
    ansible.builtin.stat:
      path: /var/run/myapp.lock
    register: lock_file
    until: not lock_file.stat.exists
    retries: 30
    delay: 10
    
  # Wait for port to be available
  - name: Wait for application port
    ansible.builtin.wait_for:
      port: 8080
      state: started
      timeout: 300
      
  # Wait for SSH after reboot
  - name: Reboot the server
    ansible.builtin.reboot:
      reboot_timeout: 600
      
  # Manual wait_for_connection
  - name: Wait for connection after reboot
    ansible.builtin.wait_for_connection:
      delay: 30
      timeout: 300
```

---

## Assertions

### assert Module

Validate conditions and fail with custom messages:

```yaml
tasks:
  # Simple assertion
  - name: Verify minimum memory
    ansible.builtin.assert:
      that:
        - ansible_memtotal_mb >= 2048
      fail_msg: "Server needs at least 2GB RAM"
      success_msg: "Memory check passed"
      
  # Multiple conditions
  - name: Pre-deployment checks
    ansible.builtin.assert:
      that:
        - ansible_distribution == "Ubuntu"
        - ansible_distribution_major_version | int >= 20
        - ansible_processor_vcpus >= 2
        - free_disk_space_gb | int >= 10
      fail_msg: "System does not meet requirements"
      
  # Assert variable values
  - name: Validate configuration
    ansible.builtin.assert:
      that:
        - app_environment in ['development', 'staging', 'production']
        - app_port | int > 0 and app_port | int < 65536
        - database_host is defined
        - database_host | length > 0
      fail_msg: "Invalid configuration values"
```

### fail Module

Explicitly fail with a message:

```yaml
tasks:
  - name: Check disk space
    ansible.builtin.shell: df -BG / | tail -1 | awk '{print $4}' | tr -d 'G'
    register: disk_space
    changed_when: false
    
  - name: Fail if disk space is low
    ansible.builtin.fail:
      msg: "Insufficient disk space: {{ disk_space.stdout }}GB available"
    when: disk_space.stdout | int < 10
    
  # Conditional failure
  - name: Prevent production changes on Friday
    ansible.builtin.fail:
      msg: "No deployments on Fridays!"
    when:
      - app_environment == 'production'
      - ansible_date_time.weekday == 'Friday'
```

---

## Error Information

### Accessing Error Details

```yaml
tasks:
  - name: Risky operation
    block:
      - name: This might fail
        ansible.builtin.command: /opt/risky.sh
        
    rescue:
      - name: Show error information
        ansible.builtin.debug:
          msg: |
            Failed task: {{ ansible_failed_task.name }}
            Failed result: {{ ansible_failed_result }}
            
      - name: Log the error
        ansible.builtin.lineinfile:
          path: /var/log/ansible-errors.log
          line: "{{ ansible_date_time.iso8601 }} - {{ ansible_failed_task.name }}: {{ ansible_failed_result.msg | default('Unknown error') }}"
          create: yes
        delegate_to: localhost
```

### ansible_failed_* Variables

Available in rescue blocks:

| Variable | Description |
|----------|-------------|
| `ansible_failed_task` | The task that failed |
| `ansible_failed_result` | The result of the failed task |

```yaml
rescue:
  - name: Detailed error report
    ansible.builtin.debug:
      msg: |
        Task Name: {{ ansible_failed_task.name }}
        Task Action: {{ ansible_failed_task.action }}
        Return Code: {{ ansible_failed_result.rc | default('N/A') }}
        Stdout: {{ ansible_failed_result.stdout | default('N/A') }}
        Stderr: {{ ansible_failed_result.stderr | default('N/A') }}
        Message: {{ ansible_failed_result.msg | default('N/A') }}
```

---

## Force Handlers

By default, handlers don't run if a play fails. Override this:

### Command Line

```bash
ansible-playbook playbook.yml --force-handlers
```

### In Playbook

```yaml
---
- name: Play with forced handlers
  hosts: all
  force_handlers: true
  
  tasks:
    - name: Configure service
      ansible.builtin.template:
        src: config.j2
        dest: /etc/myapp/config.yml
      notify: Restart myapp
      
    - name: This might fail
      ansible.builtin.command: /bin/false
      
  handlers:
    - name: Restart myapp
      ansible.builtin.service:
        name: myapp
        state: restarted
      # Handler runs even if play failed
```

### meta: flush_handlers

Force handlers to run at a specific point:

```yaml
tasks:
  - name: Update config
    ansible.builtin.template:
      src: config.j2
      dest: /etc/myapp/config.yml
    notify: Restart service
    
  - name: Run handlers now
    ansible.builtin.meta: flush_handlers
    
  - name: Risky operation (handlers already ran)
    ansible.builtin.command: /opt/risky.sh
```

---

## Graceful Degradation

### Optional Tasks

```yaml
tasks:
  # Try preferred method, fall back to alternative
  - name: Install from preferred repository
    ansible.builtin.apt:
      name: myapp
      state: present
    register: preferred_install
    ignore_errors: yes
    
  - name: Fall back to alternative repository
    ansible.builtin.apt:
      deb: https://backup.example.com/myapp.deb
    when: preferred_install is failed
```

### Feature Flags

```yaml
vars:
  enable_monitoring: true
  enable_backups: true
  
tasks:
  - name: Configure monitoring
    block:
      - name: Install monitoring agent
        ansible.builtin.apt:
          name: monitoring-agent
          state: present
          
      - name: Configure agent
        ansible.builtin.template:
          src: monitoring.conf.j2
          dest: /etc/monitoring/agent.conf
    when: enable_monitoring
    ignore_errors: "{{ not enable_monitoring_required | default(false) }}"
```

### Circuit Breaker Pattern

```yaml
tasks:
  - name: Check if service is healthy
    ansible.builtin.uri:
      url: http://{{ service_host }}/health
      status_code: 200
    register: health_check
    ignore_errors: yes
    
  - name: Proceed only if service is healthy
    block:
      - name: Perform operations
        ansible.builtin.command: /opt/operations.sh
    when: health_check is succeeded
    
  - name: Handle unhealthy service
    ansible.builtin.debug:
      msg: "Service unhealthy, skipping operations"
    when: health_check is failed
```

---

## Error Handling Patterns

### Pattern 1: Validate Before Execute

```yaml
tasks:
  - name: Pre-flight checks
    block:
      - name: Check disk space
        ansible.builtin.shell: df -BG / | awk 'NR==2 {print $4}' | tr -d 'G'
        register: disk_space
        changed_when: false
        failed_when: disk_space.stdout | int < 5
        
      - name: Check memory
        ansible.builtin.assert:
          that: ansible_memfree_mb > 512
          fail_msg: "Insufficient memory"
          
      - name: Check connectivity
        ansible.builtin.wait_for:
          host: "{{ database_host }}"
          port: 5432
          timeout: 5
          
  - name: Execute main tasks
    ansible.builtin.include_tasks: main-tasks.yml
```

### Pattern 2: Checkpoint and Rollback

```yaml
vars:
  checkpoint_file: /var/run/deployment-checkpoint

tasks:
  - name: Create checkpoint
    ansible.builtin.copy:
      content: "{{ ansible_date_time.epoch }}"
      dest: "{{ checkpoint_file }}"
      
  - name: Deployment tasks
    block:
      - name: Step 1
        ansible.builtin.command: /opt/step1.sh
        
      - name: Step 2
        ansible.builtin.command: /opt/step2.sh
        
      - name: Step 3
        ansible.builtin.command: /opt/step3.sh
        
    rescue:
      - name: Rollback to checkpoint
        ansible.builtin.command: /opt/rollback.sh {{ lookup('file', checkpoint_file) }}
        
    always:
      - name: Remove checkpoint
        ansible.builtin.file:
          path: "{{ checkpoint_file }}"
          state: absent
```

### Pattern 3: Retry with Exponential Backoff

```yaml
tasks:
  - name: Retry with increasing delays
    ansible.builtin.uri:
      url: http://api.example.com/resource
      method: POST
      body: "{{ request_body | to_json }}"
      body_format: json
      status_code: [200, 201]
    register: api_response
    until: api_response is succeeded
    retries: 5
    delay: "{{ item }}"
    loop: [1, 2, 4, 8, 16]  # Exponential backoff
    loop_control:
      index_var: retry_index
```

### Pattern 4: Conditional Error Handling

```yaml
tasks:
  - name: Attempt primary method
    ansible.builtin.command: /opt/primary-method.sh
    register: primary_result
    ignore_errors: yes
    
  - name: Handle based on error type
    block:
      - name: Retry for transient errors
        ansible.builtin.command: /opt/primary-method.sh
        when: "'TRANSIENT' in primary_result.stderr"
        
      - name: Use fallback for permanent errors
        ansible.builtin.command: /opt/fallback-method.sh
        when: "'PERMANENT' in primary_result.stderr"
        
      - name: Escalate unknown errors
        ansible.builtin.fail:
          msg: "Unknown error: {{ primary_result.stderr }}"
        when:
          - "'TRANSIENT' not in primary_result.stderr"
          - "'PERMANENT' not in primary_result.stderr"
    when: primary_result is failed
```

---

## Debugging Failures

### Run in Check Mode

```bash
# Dry run - see what would happen
ansible-playbook playbook.yml --check

# With diff output
ansible-playbook playbook.yml --check --diff
```

### Increase Verbosity

```bash
# Verbose levels
ansible-playbook playbook.yml -v      # Show task results
ansible-playbook playbook.yml -vv     # Show task input
ansible-playbook playbook.yml -vvv    # Show connection info
ansible-playbook playbook.yml -vvvv   # Show plugin loading
```

### Start at Failed Task

```bash
# Start at a specific task
ansible-playbook playbook.yml --start-at-task="Task Name"

# Step through tasks one at a time
ansible-playbook playbook.yml --step
```

### Debug Strategy

```yaml
---
- name: Debug playbook
  hosts: all
  strategy: debug  # Enter debugger on failure
  
  tasks:
    - name: Failing task
      ansible.builtin.command: /bin/false
```

Debug commands:
- `p task` - Print current task
- `p result` - Print task result
- `p vars` - Print all variables
- `r` - Retry the task
- `c` - Continue to next task
- `q` - Quit debugger

---

## Complete Example

```yaml
---
- name: Robust Application Deployment
  hosts: webservers
  become: yes
  max_fail_percentage: 25
  
  vars:
    app_name: mywebapp
    app_version: "{{ deploy_version | default('latest') }}"
    rollback_enabled: true
    health_check_retries: 10
    health_check_delay: 5
    
  pre_tasks:
    - name: Pre-deployment validation
      block:
        - name: Check disk space
          ansible.builtin.shell: df -BG /opt | tail -1 | awk '{print $4}' | tr -d 'G'
          register: disk_space
          changed_when: false
          
        - name: Validate disk space
          ansible.builtin.assert:
            that: disk_space.stdout | int >= 5
            fail_msg: "Need at least 5GB free space"
            
        - name: Verify deployment target
          ansible.builtin.stat:
            path: /opt/{{ app_name }}
          register: app_dir
          
        - name: Create app directory if missing
          ansible.builtin.file:
            path: /opt/{{ app_name }}
            state: directory
            owner: www-data
            mode: '0755'
          when: not app_dir.stat.exists
          
  tasks:
    - name: Deploy application
      block:
        - name: Create backup
          ansible.builtin.archive:
            path: /opt/{{ app_name }}
            dest: /opt/backups/{{ app_name }}-{{ ansible_date_time.epoch }}.tar.gz
          when: rollback_enabled
          
        - name: Stop application
          ansible.builtin.service:
            name: "{{ app_name }}"
            state: stopped
          register: stop_result
          failed_when:
            - stop_result is failed
            - "'could not find' not in stop_result.msg | default('')"
            
        - name: Deploy new version
          ansible.builtin.unarchive:
            src: "https://releases.example.com/{{ app_name }}/{{ app_version }}.tar.gz"
            dest: /opt/{{ app_name }}
            remote_src: yes
          notify: Restart application
          
        - name: Update configuration
          ansible.builtin.template:
            src: app-config.yml.j2
            dest: /opt/{{ app_name }}/config.yml
            owner: www-data
            mode: '0640'
          notify: Restart application
          
        - name: Run handlers now
          ansible.builtin.meta: flush_handlers
          
        - name: Health check
          ansible.builtin.uri:
            url: http://localhost:8080/health
            status_code: 200
          register: health
          until: health.status == 200
          retries: "{{ health_check_retries }}"
          delay: "{{ health_check_delay }}"
          
      rescue:
        - name: Deployment failed
          ansible.builtin.debug:
            msg: |
              Deployment failed!
              Task: {{ ansible_failed_task.name | default('Unknown') }}
              Error: {{ ansible_failed_result.msg | default('Unknown error') }}
              
        - name: Rollback
          block:
            - name: Find latest backup
              ansible.builtin.find:
                paths: /opt/backups
                patterns: "{{ app_name }}-*.tar.gz"
              register: backups
              
            - name: Restore from backup
              ansible.builtin.unarchive:
                src: "{{ (backups.files | sort(attribute='mtime') | last).path }}"
                dest: /opt/{{ app_name }}
                remote_src: yes
              when: backups.files | length > 0
              
            - name: Restart previous version
              ansible.builtin.service:
                name: "{{ app_name }}"
                state: restarted
          when: rollback_enabled
          
        - name: Mark host as failed
          ansible.builtin.fail:
            msg: "Deployment failed and was rolled back"
            
      always:
        - name: Log deployment result
          ansible.builtin.lineinfile:
            path: /var/log/deployments.log
            line: "{{ ansible_date_time.iso8601 }} {{ inventory_hostname }} {{ app_version }} {{ 'SUCCESS' if ansible_failed_task is not defined else 'FAILED' }}"
            create: yes
          delegate_to: localhost
          become: no
          
  handlers:
    - name: Restart application
      ansible.builtin.service:
        name: "{{ app_name }}"
        state: restarted
```

---

## Summary

| Technique | Use Case |
|-----------|----------|
| `ignore_errors` | Continue despite failures |
| `failed_when` | Custom failure conditions |
| `changed_when` | Custom change detection |
| `block/rescue/always` | Try/catch/finally pattern |
| `any_errors_fatal` | Stop on first failure |
| `max_fail_percentage` | Tolerate some failures |
| `until/retries/delay` | Retry until success |
| `assert` | Validate conditions |
| `fail` | Explicit failure |
| `force_handlers` | Run handlers despite failures |

### Best Practices

1. **Validate early** - Check prerequisites before making changes
2. **Use blocks** - Group related tasks for unified error handling
3. **Enable rollback** - Always have a recovery plan
4. **Log everything** - Record successes and failures
5. **Be specific** - Use custom `failed_when` for accurate error detection
6. **Test failure paths** - Verify rescue blocks work correctly
7. **Set timeouts** - Don't wait forever for operations
8. **Graceful degradation** - Handle partial failures appropriately

Master error handling to build resilient, production-ready automation!

---

## Practice Exercises

### Exercise 1: Basic Error Handling

Create a playbook that attempts to install a package and handles potential failures:

```yaml
---
# exercise1.yml - Practice basic error handling
- name: Package Installation with Error Handling
  hosts: localhost
  connection: local
  become: yes
  
  vars:
    packages_to_install:
      - nginx
      - nonexistent-package-12345
      - curl
  
  tasks:
    # TODO: Install each package and handle failures gracefully
    # - Use a loop to iterate over packages
    # - Use ignore_errors to continue on failure
    # - Register results and display which packages failed
    
    - name: Attempt to install packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages_to_install }}"
      register: install_results
      ignore_errors: yes
      
    - name: Report failed installations
      ansible.builtin.debug:
        msg: "Failed to install: {{ item.item }}"
      loop: "{{ install_results.results }}"
      when: item is failed
```

**Challenge**: Modify the playbook to retry failed packages once before giving up.

---

### Exercise 2: Block/Rescue/Always

Create a deployment playbook with proper rollback:

```yaml
---
# exercise2.yml - Practice block/rescue/always
- name: Application Deployment Exercise
  hosts: localhost
  connection: local
  
  vars:
    app_name: mywebapp
    app_version: "1.0.0"
    deploy_dir: /tmp/deploy-exercise
  
  tasks:
    - name: Setup deployment directory
      ansible.builtin.file:
        path: "{{ deploy_dir }}"
        state: directory
        
    - name: Deployment with rollback capability
      block:
        # TODO: Add deployment steps
        - name: Create version file
          ansible.builtin.copy:
            content: "Version: {{ app_version }}\n"
            dest: "{{ deploy_dir }}/version.txt"
            
        - name: Simulate deployment step
          ansible.builtin.command: "echo 'Deploying {{ app_name }}'"
          changed_when: false
          
        # Uncomment to simulate failure:
        # - name: Simulated failure
        #   ansible.builtin.command: /bin/false
          
        - name: Mark deployment as successful
          ansible.builtin.copy:
            content: "Deployment successful at {{ ansible_date_time.iso8601 }}\n"
            dest: "{{ deploy_dir }}/status.txt"
            
      rescue:
        - name: Rollback - remove version file
          ansible.builtin.file:
            path: "{{ deploy_dir }}/version.txt"
            state: absent
            
        - name: Log rollback
          ansible.builtin.copy:
            content: "Rollback executed at {{ ansible_date_time.iso8601 }}\n"
            dest: "{{ deploy_dir }}/rollback.log"
            
        - name: Notify about failure
          ansible.builtin.debug:
            msg: "Deployment failed! Rollback completed."
            
      always:
        - name: Display final status
          ansible.builtin.debug:
            msg: "Deployment process completed for {{ app_name }}"
```

**Challenge**: Add a pre-deployment backup step that saves the previous version.

---

### Exercise 3: Retry Until Success

Create a playbook that waits for a service to become available:

```yaml
---
# exercise3.yml - Practice retry/until
- name: Wait for Service Exercise
  hosts: localhost
  connection: local
  
  vars:
    service_port: 8080
    max_retries: 5
    retry_delay: 2
  
  tasks:
    # Start a simple HTTP server in background for testing
    - name: Start test HTTP server
      ansible.builtin.shell: |
        cd /tmp && python3 -m http.server {{ service_port }} &
        echo $! > /tmp/http-server.pid
      async: 10
      poll: 0
      changed_when: false
      
    - name: Wait for HTTP server to be ready
      ansible.builtin.uri:
        url: "http://localhost:{{ service_port }}"
        method: GET
        status_code: 200
      register: result
      until: result.status == 200
      retries: "{{ max_retries }}"
      delay: "{{ retry_delay }}"
      
    - name: Confirm service is available
      ansible.builtin.debug:
        msg: "Service is responding on port {{ service_port }}"
        
    - name: Cleanup - stop test server
      ansible.builtin.shell: |
        if [ -f /tmp/http-server.pid ]; then
          kill $(cat /tmp/http-server.pid) 2>/dev/null || true
          rm /tmp/http-server.pid
        fi
      changed_when: false
```

**Challenge**: Add a fallback that tries an alternate port if the primary fails.

---

### Exercise 4: Custom Failure Conditions

Create a playbook that validates system requirements:

```yaml
---
# exercise4.yml - Practice custom failure conditions
- name: System Requirements Validation
  hosts: localhost
  connection: local
  gather_facts: yes
  
  vars:
    min_memory_mb: 512
    min_disk_gb: 1
    required_commands:
      - python3
      - curl
      - git
  
  tasks:
    - name: Check memory requirements
      ansible.builtin.assert:
        that:
          - ansible_memtotal_mb >= min_memory_mb
        fail_msg: "Insufficient memory: {{ ansible_memtotal_mb }}MB < {{ min_memory_mb }}MB required"
        success_msg: "Memory check passed: {{ ansible_memtotal_mb }}MB available"
        
    - name: Check for required commands
      ansible.builtin.command: "which {{ item }}"
      loop: "{{ required_commands }}"
      register: command_check
      failed_when: false
      changed_when: false
      
    - name: Report missing commands
      ansible.builtin.fail:
        msg: "Missing required command: {{ item.item }}"
      loop: "{{ command_check.results }}"
      when: item.rc != 0
      
    - name: Check disk space
      ansible.builtin.shell: df -BG / | tail -1 | awk '{print $4}' | tr -d 'G'
      register: disk_space
      changed_when: false
      failed_when: disk_space.stdout | int < min_disk_gb
      
    - name: All requirements passed
      ansible.builtin.debug:
        msg: "System meets all requirements!"
```

**Challenge**: Add network connectivity checks (ping to external host).

---

### Exercise 5: Complete Error Handling Project

Build a complete playbook that:
1. Validates prerequisites
2. Creates backups
3. Deploys an application
4. Handles failures with rollback
5. Sends notifications

```yaml
---
# exercise5.yml - Complete error handling project
- name: Production-Ready Deployment
  hosts: localhost
  connection: local
  
  vars:
    app_name: demo-app
    deploy_base: /tmp/demo-deploy
    current_version: "1.0.0"
    new_version: "1.1.0"
    simulate_failure: false  # Set to true to test failure handling
  
  tasks:
    # Phase 1: Prerequisites
    - name: "Phase 1: Validate Prerequisites"
      block:
        - name: Create base directories
          ansible.builtin.file:
            path: "{{ item }}"
            state: directory
          loop:
            - "{{ deploy_base }}"
            - "{{ deploy_base }}/backups"
            - "{{ deploy_base }}/app"
            - "{{ deploy_base }}/logs"
            
        - name: Create initial app version (if not exists)
          ansible.builtin.copy:
            content: |
              App: {{ app_name }}
              Version: {{ current_version }}
              Deployed: {{ ansible_date_time.iso8601 }}
            dest: "{{ deploy_base }}/app/version.txt"
            force: no
            
      rescue:
        - name: Prerequisites failed
          ansible.builtin.fail:
            msg: "Failed to set up prerequisites. Cannot continue."
            
    # Phase 2: Backup
    - name: "Phase 2: Create Backup"
      block:
        - name: Create backup of current version
          ansible.builtin.archive:
            path: "{{ deploy_base }}/app"
            dest: "{{ deploy_base }}/backups/backup-{{ ansible_date_time.epoch }}.tar.gz"
          register: backup_result
          
        - name: Record backup location
          ansible.builtin.set_fact:
            backup_file: "{{ backup_result.dest }}"
            
      rescue:
        - name: Backup failed - abort deployment
          ansible.builtin.fail:
            msg: "Backup failed. Deployment aborted for safety."
            
    # Phase 3: Deployment
    - name: "Phase 3: Deploy New Version"
      block:
        - name: Update version file
          ansible.builtin.copy:
            content: |
              App: {{ app_name }}
              Version: {{ new_version }}
              Deployed: {{ ansible_date_time.iso8601 }}
              Previous: {{ current_version }}
            dest: "{{ deploy_base }}/app/version.txt"
            
        - name: Simulate deployment task
          ansible.builtin.command: "echo 'Deploying {{ new_version }}...'"
          changed_when: true
          
        - name: Simulate failure (if enabled)
          ansible.builtin.fail:
            msg: "Simulated deployment failure!"
          when: simulate_failure
          
        - name: Verify deployment
          ansible.builtin.command: "cat {{ deploy_base }}/app/version.txt"
          register: verify_result
          failed_when: "new_version not in verify_result.stdout"
          changed_when: false
          
        - name: Mark deployment successful
          ansible.builtin.set_fact:
            deployment_status: "success"
            
      rescue:
        - name: Deployment failed - initiating rollback
          ansible.builtin.debug:
            msg: "Deployment failed! Rolling back to backup..."
            
        - name: Restore from backup
          ansible.builtin.unarchive:
            src: "{{ backup_file }}"
            dest: "{{ deploy_base }}"
            remote_src: yes
          when: backup_file is defined
          
        - name: Mark deployment as failed
          ansible.builtin.set_fact:
            deployment_status: "failed"
            
      always:
        - name: Log deployment result
          ansible.builtin.lineinfile:
            path: "{{ deploy_base }}/logs/deployments.log"
            line: "{{ ansible_date_time.iso8601 }} | {{ new_version }} | {{ deployment_status | default('unknown') }}"
            create: yes
            
    # Phase 4: Report
    - name: "Phase 4: Final Report"
      block:
        - name: Show deployment summary
          ansible.builtin.debug:
            msg: |
              ================================
              DEPLOYMENT SUMMARY
              ================================
              Application: {{ app_name }}
              Target Version: {{ new_version }}
              Status: {{ deployment_status | default('unknown') | upper }}
              Backup: {{ backup_file | default('N/A') }}
              Time: {{ ansible_date_time.iso8601 }}
              ================================
              
        - name: Fail play if deployment failed
          ansible.builtin.fail:
            msg: "Deployment failed and was rolled back"
          when: deployment_status | default('failed') == 'failed'
```

**How to use:**
```bash
# Run successful deployment
ansible-playbook exercise5.yml

# Test failure handling
ansible-playbook exercise5.yml -e "simulate_failure=true"
```

---

### Exercise Solutions Tips

1. **Test incrementally** - Run playbooks with `--check` first
2. **Use verbose mode** - Add `-v`, `-vv`, or `-vvv` for debugging
3. **Start simple** - Get basic functionality working before adding error handling
4. **Test failure paths** - Deliberately cause failures to verify rescue blocks work
5. **Review logs** - Check output files to verify behavior

---

## Additional Resources

### Official Documentation

- [Ansible Documentation](https://docs.ansible.com/)
- [Error Handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)
- [Blocks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
- [Module Index](https://docs.ansible.com/ansible/latest/collections/index_module.html)

### Community Resources

- [Ansible Galaxy](https://galaxy.ansible.com/) - Find roles and collections
- [Ansible GitHub](https://github.com/ansible/ansible) - Source code and issues
- [r/ansible](https://www.reddit.com/r/ansible/) - Reddit community

### Practice Environments

- Use Vagrant or Docker to create test environments
- AWS Free Tier for cloud practice
- LocalStack for AWS service emulation

Keep practicing to become proficient in Ansible automation!
