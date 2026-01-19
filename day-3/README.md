# Day 3: YAML and YAML Basics

## What is YAML?

YAML stands for **"YAML Ain't Markup Language"** (a recursive acronym). It's a human-readable data serialization format that is commonly used for configuration files. Ansible uses YAML for its playbooks, inventory files, and variable files.

YAML files typically use the `.yml` or `.yaml` file extension.

---

## Why YAML for Ansible?

- **Human-readable**: Easy to read and write
- **Minimal syntax**: No brackets or tags like JSON or XML
- **Hierarchical**: Supports nested data structures naturally
- **Comments**: Allows inline documentation with `#`

---

## YAML Basics

### 1. Key-Value Pairs

The most basic YAML structure is a key-value pair, separated by a colon and space:

```yaml
name: webserver
port: 8080
enabled: true
```

### 2. Strings

Strings can be written in multiple ways:

```yaml
# Without quotes (most common)
name: my-application

# With single quotes (preserves literal content)
message: 'Hello World'

# With double quotes (allows escape sequences)
path: "C:\\Users\\admin"

# Multi-line strings with | (preserves newlines)
description: |
  This is a multi-line
  string that preserves
  line breaks.

# Multi-line strings with > (folds into single line)
summary: >
  This is a long string
  that will be folded
  into a single line.
```

### 3. Numbers and Booleans

```yaml
# Integers
count: 42
port: 8080

# Floats
price: 19.99
pi: 3.14159

# Booleans (multiple valid formats)
enabled: true
active: yes
running: on

disabled: false
stopped: no
inactive: off
```

### 4. Lists (Arrays)

Lists are defined using dashes:

```yaml
# Block style (most common in Ansible)
packages:
  - nginx
  - postgresql
  - redis

# Inline style
colors: [red, green, blue]

# List of numbers
ports:
  - 80
  - 443
  - 8080
```

### 5. Dictionaries (Maps)

Dictionaries contain nested key-value pairs:

```yaml
# Block style
server:
  hostname: web01
  ip: 192.168.1.10
  port: 8080

# Inline style
person: {name: John, age: 30}
```

### 6. Nested Structures

YAML supports complex nested structures:

```yaml
web_servers:
  - name: server1
    ip: 192.168.1.10
    ports:
      - 80
      - 443
    tags:
      environment: production
      role: frontend

  - name: server2
    ip: 192.168.1.11
    ports:
      - 8080
    tags:
      environment: staging
      role: backend
```

### 7. Comments

Comments start with `#`:

```yaml
# This is a comment
name: myapp  # Inline comment

# Multi-line comments require
# multiple hash symbols
```

---

## YAML Syntax Rules

### Indentation

- **Use spaces, NOT tabs** (this is critical!)
- Consistent indentation (typically 2 spaces)
- Indentation defines structure and hierarchy

```yaml
# Correct - using spaces
parent:
  child: value
  another_child: value2

# WRONG - mixing tabs and spaces will cause errors!
```

### Special Characters

Some characters have special meaning and may need quoting:

```yaml
# These need quotes
special: "value: with colon"
another: "value # with hash"
regex: '[a-z]+'
```

### Null Values

```yaml
empty_value: null
also_empty: ~
implicit_null:
```

---

## YAML in Ansible Context

Here's how YAML is used in Ansible playbooks:

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    max_clients: 200

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Start nginx service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Copy configuration file
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
...
```

### Key Ansible YAML Elements

| Element | Description |
|---------|-------------|
| `---` | Document start marker (optional but recommended) |
| `...` | Document end marker (optional) |
| `-` | List item indicator |
| `:` | Key-value separator |
| `#` | Comment |

---

## Common YAML Mistakes to Avoid

### 1. Tab Characters
```yaml
# WRONG - tabs will cause errors
parent:
	child: value  # This is a tab!

# CORRECT - use spaces
parent:
  child: value
```

### 2. Missing Space After Colon
```yaml
# WRONG
name:value

# CORRECT
name: value
```

### 3. Incorrect Indentation
```yaml
# WRONG - inconsistent indentation
parent:
  child1: value
   child2: value  # Extra space!

# CORRECT
parent:
  child1: value
  child2: value
```

### 4. Unquoted Special Characters
```yaml
# WRONG - colon in value
message: Error: Something failed

# CORRECT
message: "Error: Something failed"
```

---

## Validating YAML

You can validate YAML syntax using various tools:

```bash
# Using Python
python -c "import yaml; yaml.safe_load(open('file.yml'))"

# Using yamllint (recommended)
pip install yamllint
yamllint playbook.yml

# Using ansible-lint for Ansible-specific validation
pip install ansible-lint
ansible-lint playbook.yml
```

---

## Practice Exercise

Create a file called `practice.yml` with the following structure:

```yaml
---
# My first YAML file
application:
  name: my-web-app
  version: 1.0.0
  
  database:
    host: localhost
    port: 5432
    name: mydb
    
  features:
    - user-authentication
    - api-gateway
    - caching
    
  settings:
    debug: false
    log_level: info
    max_connections: 100
...
```

---

## Summary

- YAML is a human-readable data format used extensively in Ansible
- Use **spaces** (not tabs) for indentation
- Key-value pairs are separated by `: `
- Lists use `-` prefix
- Comments start with `#`
- Quote strings containing special characters
- The `---` and `...` markers denote document boundaries

Understanding YAML is fundamental to working with Ansible effectively!

---

# Ansible Playbooks

## What is an Ansible Playbook?

An Ansible **Playbook** is a YAML file that defines a set of tasks to be executed on remote hosts. Playbooks are the core of Ansible's configuration management and automation capabilities. They describe the desired state of your systems and the steps needed to achieve that state.

Think of a playbook as a **recipe** — it contains all the instructions needed to configure your infrastructure.

---

## Playbook Structure

A playbook consists of one or more **plays**. Each play targets a group of hosts and defines tasks to run on them.

```yaml
---
# A playbook with a single play
- name: My First Playbook
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
```

### Anatomy of a Playbook

| Component | Description |
|-----------|-------------|
| `name` | A descriptive name for the play (for documentation) |
| `hosts` | Target hosts or groups from inventory |
| `become` | Privilege escalation (run as sudo) |
| `vars` | Variables for the play |
| `tasks` | List of tasks to execute |
| `handlers` | Tasks triggered by notifications |

---

## Basic Playbook Example

```yaml
---
- name: Configure Web Server
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    document_root: /var/www/html

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Create document root
      file:
        path: "{{ document_root }}"
        state: directory
        mode: '0755'

    - name: Copy index.html
      copy:
        src: files/index.html
        dest: "{{ document_root }}/index.html"
        mode: '0644'

    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: yes
...
```

---

## Running Playbooks

### Basic Syntax

```bash
# Run a playbook
ansible-playbook playbook.yml

# Run with a specific inventory
ansible-playbook -i inventory.ini playbook.yml

# Run with verbose output
ansible-playbook playbook.yml -v      # verbose
ansible-playbook playbook.yml -vv     # more verbose
ansible-playbook playbook.yml -vvv    # debug level

# Dry run (check mode)
ansible-playbook playbook.yml --check

# Show differences in files
ansible-playbook playbook.yml --diff

# Limit to specific hosts
ansible-playbook playbook.yml --limit web01

# Start at a specific task
ansible-playbook playbook.yml --start-at-task="Install nginx"
```

---

## Playbook Keywords

### Play-Level Keywords

```yaml
---
- name: Example Play
  hosts: all                    # Target hosts (required)
  become: yes                   # Enable privilege escalation
  become_user: root             # User to become (default: root)
  become_method: sudo           # Method for escalation
  gather_facts: yes             # Collect system facts (default: yes)
  connection: ssh               # Connection type
  remote_user: ansible          # SSH user
  
  vars:                         # Play variables
    my_var: value
    
  vars_files:                   # Load variables from files
    - vars/main.yml
    
  pre_tasks:                    # Tasks to run before roles
    - name: Pre-task
      debug:
        msg: "Running pre-task"
        
  roles:                        # Roles to apply
    - webserver
    - database
    
  tasks:                        # Main tasks
    - name: Main task
      debug:
        msg: "Running main task"
        
  post_tasks:                   # Tasks to run after roles
    - name: Post-task
      debug:
        msg: "Running post-task"
        
  handlers:                     # Event-driven tasks
    - name: Restart service
      service:
        name: nginx
        state: restarted
```

---

## Tasks

Tasks are the individual units of work in a playbook. Each task calls an Ansible **module**.

### Task Structure

```yaml
tasks:
  - name: Descriptive task name
    module_name:
      parameter1: value1
      parameter2: value2
    register: result_variable
    when: condition
    become: yes
    tags:
      - mytag
```

### Common Task Parameters

| Parameter | Description |
|-----------|-------------|
| `name` | Description of what the task does |
| `register` | Store the result in a variable |
| `when` | Conditional execution |
| `become` | Privilege escalation for this task |
| `tags` | Tags for selective execution |
| `loop` | Iterate over a list |
| `notify` | Trigger handlers |
| `ignore_errors` | Continue even if task fails |
| `changed_when` | Override change detection |
| `failed_when` | Override failure detection |

---

## Variables in Playbooks

### Defining Variables

```yaml
---
- name: Variable Examples
  hosts: all
  vars:
    # Simple variables
    app_name: myapp
    app_port: 8080
    
    # List variable
    packages:
      - nginx
      - git
      - curl
    
    # Dictionary variable
    database:
      host: localhost
      port: 5432
      name: mydb
      
  tasks:
    - name: Use simple variable
      debug:
        msg: "App: {{ app_name }} on port {{ app_port }}"
    
    - name: Install packages from list
      apt:
        name: "{{ packages }}"
        state: present
    
    - name: Use dictionary variable
      debug:
        msg: "Database: {{ database.name }} on {{ database.host }}:{{ database.port }}"
```

### Variable Precedence (Low to High)

1. Role defaults
2. Inventory vars
3. Playbook vars
4. Role vars
5. Block vars
6. Task vars
7. Extra vars (`-e`)

```bash
# Pass extra variables via command line (highest precedence)
ansible-playbook playbook.yml -e "app_port=9000"
```

---

## Conditionals

Use `when` to conditionally execute tasks:

```yaml
tasks:
  # Simple condition
  - name: Install on Debian
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"

  # Multiple conditions (AND)
  - name: Install on Ubuntu 22.04
    apt:
      name: nginx
      state: present
    when:
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_version == "22.04"

  # OR condition
  - name: Install on Debian or Ubuntu
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian" or ansible_os_family == "Ubuntu"

  # Check variable existence
  - name: Run if variable is defined
    debug:
      msg: "Variable exists: {{ my_var }}"
    when: my_var is defined

  # Check previous task result
  - name: Check service status
    command: systemctl status nginx
    register: nginx_status
    ignore_errors: yes

  - name: Start nginx if not running
    service:
      name: nginx
      state: started
    when: nginx_status.rc != 0
```

---

## Loops

Iterate over lists with `loop`:

```yaml
tasks:
  # Simple loop
  - name: Install multiple packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - git
      - curl

  # Loop with index
  - name: Create users
    user:
      name: "{{ item }}"
      state: present
    loop:
      - alice
      - bob
      - charlie
    loop_control:
      index_var: user_index

  # Loop over dictionary
  - name: Create users with details
    user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
      groups: "{{ item.groups }}"
    loop:
      - { name: 'alice', uid: 1001, groups: 'admin' }
      - { name: 'bob', uid: 1002, groups: 'developers' }

  # Loop with variable
  - name: Install packages from variable
    apt:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"
    
  # Nested loops with subelements
  - name: Add SSH keys for users
    authorized_key:
      user: "{{ item.0.name }}"
      key: "{{ item.1 }}"
    loop: "{{ users | subelements('ssh_keys') }}"
```

---

## Handlers

Handlers are special tasks that only run when notified:

```yaml
---
- name: Configure Nginx
  hosts: webservers
  become: yes

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Copy nginx configuration
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

    - name: Copy site configuration
      template:
        src: site.conf.j2
        dest: /etc/nginx/sites-available/default
      notify:
        - Validate nginx config
        - Reload nginx

  handlers:
    - name: Validate nginx config
      command: nginx -t
      changed_when: false

    - name: Restart nginx
      service:
        name: nginx
        state: restarted

    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

### Handler Behavior

- Handlers run **once** at the end of the play
- Multiple notifications to the same handler trigger it only once
- Handlers run in the order they are defined, not notified
- Use `meta: flush_handlers` to run handlers immediately

```yaml
tasks:
  - name: Some task
    command: echo "hello"
    notify: My handler

  - name: Run handlers now
    meta: flush_handlers

  - name: Continue with other tasks
    debug:
      msg: "Handlers have run"
```

---

## Tags

Tags allow selective execution of tasks:

```yaml
---
- name: Tagged Playbook
  hosts: all
  
  tasks:
    - name: Install packages
      apt:
        name: nginx
        state: present
      tags:
        - packages
        - nginx

    - name: Configure nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - configuration
        - nginx

    - name: Deploy application
      copy:
        src: app/
        dest: /var/www/app/
      tags:
        - deploy
        - never  # Won't run unless explicitly tagged
```

### Running with Tags

```bash
# Run only tasks with specific tags
ansible-playbook playbook.yml --tags "packages,configuration"

# Skip tasks with specific tags
ansible-playbook playbook.yml --skip-tags "deploy"

# List all tags in a playbook
ansible-playbook playbook.yml --list-tags

# Run tasks tagged with 'never' (must be explicit)
ansible-playbook playbook.yml --tags "never,deploy"
```

### Special Tags

| Tag | Description |
|-----|-------------|
| `always` | Always runs unless explicitly skipped |
| `never` | Never runs unless explicitly tagged |

---

## Blocks

Group tasks together for error handling and conditional execution:

```yaml
tasks:
  - name: Handle web server setup
    block:
      - name: Install nginx
        apt:
          name: nginx
          state: present

      - name: Start nginx
        service:
          name: nginx
          state: started

    rescue:
      - name: Handle failure
        debug:
          msg: "Installation failed, running cleanup"

      - name: Install apache as fallback
        apt:
          name: apache2
          state: present

    always:
      - name: Always run this
        debug:
          msg: "This runs regardless of success or failure"

    when: ansible_os_family == "Debian"
    become: yes
```

---

## Multiple Plays

A playbook can contain multiple plays targeting different hosts:

```yaml
---
# Play 1: Configure web servers
- name: Configure Web Servers
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

# Play 2: Configure database servers
- name: Configure Database Servers
  hosts: dbservers
  become: yes
  
  tasks:
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present

# Play 3: Configure all servers
- name: Configure All Servers
  hosts: all
  become: yes
  
  tasks:
    - name: Update all packages
      apt:
        upgrade: dist
        update_cache: yes
...
```

---

## Including and Importing

### Include Tasks (Dynamic)

```yaml
# main.yml
---
- name: Main Playbook
  hosts: all
  
  tasks:
    - name: Include common tasks
      include_tasks: tasks/common.yml
      
    - name: Include tasks conditionally
      include_tasks: "tasks/{{ ansible_os_family }}.yml"
```

### Import Tasks (Static)

```yaml
# main.yml
---
- name: Main Playbook
  hosts: all
  
  tasks:
    - name: Import common tasks
      import_tasks: tasks/common.yml
```

### Include vs Import

| Feature | `include_*` | `import_*` |
|---------|-------------|------------|
| Processing | Dynamic (runtime) | Static (parse time) |
| Loops | Supported | Not supported |
| Conditionals | Applied to include | Applied to each task |
| Tags | Applied to include | Applied to each task |
| Use case | Conditional includes | Predictable structure |

### Import Playbooks

```yaml
# site.yml
---
- import_playbook: webservers.yml
- import_playbook: dbservers.yml
- import_playbook: monitoring.yml
```

---

## Gathering Facts

Ansible automatically collects system information (facts) at the start of each play:

```yaml
---
- name: Show Facts
  hosts: all
  gather_facts: yes  # Default is yes
  
  tasks:
    - name: Display OS info
      debug:
        msg: |
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Architecture: {{ ansible_architecture }}
          Memory: {{ ansible_memtotal_mb }} MB
          CPU Cores: {{ ansible_processor_cores }}
          
    - name: Display network info
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          FQDN: {{ ansible_fqdn }}
          Default IPv4: {{ ansible_default_ipv4.address }}
```

### Disabling Fact Gathering

```yaml
---
- name: Quick Playbook
  hosts: all
  gather_facts: no  # Skip fact gathering for speed
  
  tasks:
    - name: Quick task
      command: echo "hello"
```

---

## Error Handling

### Ignore Errors

```yaml
tasks:
  - name: This might fail
    command: /bin/false
    ignore_errors: yes
    
  - name: This will still run
    debug:
      msg: "Previous task failure was ignored"
```

### Custom Failure Conditions

```yaml
tasks:
  - name: Check disk space
    shell: df -h / | tail -1 | awk '{print $5}' | tr -d '%'
    register: disk_usage
    failed_when: disk_usage.stdout | int > 90
    
  - name: Run command and check output
    command: mycommand
    register: result
    failed_when: "'ERROR' in result.stdout"
```

### Custom Changed Conditions

```yaml
tasks:
  - name: Check if config exists
    command: cat /etc/myapp/config.yml
    register: config
    changed_when: false  # Never report as changed
    
  - name: Run idempotent script
    command: /opt/scripts/setup.sh
    register: setup
    changed_when: "'Configuration updated' in setup.stdout"
```

---

## Complete Playbook Example

Here's a comprehensive playbook that incorporates many concepts:

```yaml
---
- name: Deploy Web Application
  hosts: webservers
  become: yes
  gather_facts: yes
  
  vars:
    app_name: mywebapp
    app_port: 8080
    app_user: www-data
    packages:
      - nginx
      - git
      - python3-pip
    
  vars_files:
    - vars/secrets.yml
    
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      tags: always

  tasks:
    - name: Install required packages
      apt:
        name: "{{ packages }}"
        state: present
      tags: packages

    - name: Create application directory
      file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: "{{ app_user }}"
        mode: '0755'
      tags: setup

    - name: Clone application repository
      git:
        repo: "https://github.com/example/{{ app_name }}.git"
        dest: "/opt/{{ app_name }}"
        version: main
      become_user: "{{ app_user }}"
      notify: Restart application
      tags:
        - deploy
        - git

    - name: Install Python dependencies
      pip:
        requirements: "/opt/{{ app_name }}/requirements.txt"
        virtualenv: "/opt/{{ app_name }}/venv"
      tags: deploy

    - name: Configure nginx
      block:
        - name: Copy nginx site configuration
          template:
            src: templates/nginx-site.conf.j2
            dest: "/etc/nginx/sites-available/{{ app_name }}"
          notify: Reload nginx

        - name: Enable site
          file:
            src: "/etc/nginx/sites-available/{{ app_name }}"
            dest: "/etc/nginx/sites-enabled/{{ app_name }}"
            state: link
          notify: Reload nginx
      tags: nginx

    - name: Deploy systemd service
      template:
        src: templates/app.service.j2
        dest: "/etc/systemd/system/{{ app_name }}.service"
      notify:
        - Reload systemd
        - Restart application
      tags: service

    - name: Ensure services are running
      service:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - nginx
        - "{{ app_name }}"
      tags: service

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Restart application
      service:
        name: "{{ app_name }}"
        state: restarted

    - name: Reload nginx
      service:
        name: nginx
        state: reloaded

  post_tasks:
    - name: Verify application is responding
      uri:
        url: "http://localhost:{{ app_port }}/health"
        return_content: yes
      register: health_check
      until: health_check.status == 200
      retries: 5
      delay: 10
      tags: verify
...
```

---

## Best Practices

### 1. Use Descriptive Names

```yaml
# Good
- name: Install nginx web server
  apt:
    name: nginx
    state: present

# Bad
- apt:
    name: nginx
    state: present
```

### 2. Keep Playbooks Focused

- One playbook per application or service
- Use roles for reusable components
- Keep playbooks under 100 tasks

### 3. Use Variables

```yaml
# Good - easy to modify
vars:
  http_port: 80
  
tasks:
  - name: Configure port
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
```

### 4. Always Use `state`

```yaml
# Explicit state
- name: Ensure nginx is installed
  apt:
    name: nginx
    state: present

- name: Ensure vim is removed
  apt:
    name: vim
    state: absent
```

### 5. Use Check Mode

```bash
# Always test with --check first
ansible-playbook playbook.yml --check --diff
```

### 6. Organize Your Project

```
project/
├── ansible.cfg
├── inventory/
│   ├── production
│   └── staging
├── group_vars/
│   ├── all.yml
│   └── webservers.yml
├── host_vars/
│   └── web01.yml
├── roles/
│   └── webserver/
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── dbservers.yml
└── templates/
    └── nginx.conf.j2
```

---

## Playbook Summary

| Concept | Description |
|---------|-------------|
| **Playbook** | YAML file containing one or more plays |
| **Play** | Maps hosts to tasks |
| **Task** | Single unit of work using a module |
| **Handler** | Task triggered by notifications |
| **Variable** | Reusable value |
| **Conditional** | `when` clause for conditional execution |
| **Loop** | Iterate with `loop` |
| **Tag** | Label for selective execution |
| **Block** | Group tasks for error handling |

Playbooks are the heart of Ansible automation. Master them to effectively manage your infrastructure!
