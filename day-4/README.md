# Day 4: Ansible Roles

## What are Ansible Roles?

**Roles** are a way to organize and package Ansible automation content for reuse. They provide a standardized directory structure that makes it easy to share, reuse, and manage complex playbooks.

Think of roles as **reusable building blocks** — instead of writing the same tasks repeatedly across playbooks, you package them into a role once and use it everywhere.

---

## Why Use Roles?

| Benefit | Description |
|---------|-------------|
| **Reusability** | Write once, use in multiple playbooks and projects |
| **Organization** | Clean separation of tasks, variables, files, and templates |
| **Maintainability** | Easier to update and debug isolated components |
| **Sharing** | Share with team or community via Ansible Galaxy |
| **Testing** | Test roles independently |
| **Abstraction** | Hide complexity behind a simple interface |

---

## Role Directory Structure

A role follows a specific directory structure:

```
roles/
└── webserver/
    ├── README.md           # Documentation
    ├── defaults/
    │   └── main.yml        # Default variables (lowest precedence)
    ├── vars/
    │   └── main.yml        # Role variables (higher precedence)
    ├── tasks/
    │   └── main.yml        # Main task list
    ├── handlers/
    │   └── main.yml        # Handler definitions
    ├── templates/
    │   └── nginx.conf.j2   # Jinja2 templates
    ├── files/
    │   └── index.html      # Static files
    ├── meta/
    │   └── main.yml        # Role metadata and dependencies
    └── tests/
        ├── inventory
        └── test.yml        # Test playbook
```

### Directory Purposes

| Directory | Purpose |
|-----------|---------|
| `defaults/` | Default variables (can be overridden) |
| `vars/` | Role variables (harder to override) |
| `tasks/` | Main list of tasks |
| `handlers/` | Handlers triggered by tasks |
| `templates/` | Jinja2 template files |
| `files/` | Static files to copy |
| `meta/` | Role metadata and dependencies |
| `tests/` | Test playbook and inventory |

**Note**: You only need to create directories you actually use. Ansible ignores missing directories.

---

## Creating Your First Role

### Method 1: Manual Creation

```bash
# Create role directory structure
mkdir -p roles/webserver/{tasks,handlers,templates,files,vars,defaults,meta}

# Create main.yml files
touch roles/webserver/tasks/main.yml
touch roles/webserver/handlers/main.yml
touch roles/webserver/defaults/main.yml
```

### Method 2: Using ansible-galaxy

```bash
# Create a role skeleton
ansible-galaxy role init roles/webserver

# Output:
# - Role webserver was created successfully
```

This creates the full directory structure with placeholder files.

---

## Role Example: Web Server

Let's build a complete `webserver` role step by step.

### Project Structure

```
project/
├── ansible.cfg
├── inventory
├── site.yml
└── roles/
    └── webserver/
        ├── defaults/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── templates/
        │   └── nginx.conf.j2
        └── files/
            └── index.html
```

### defaults/main.yml

Default variables that users can easily override:

```yaml
---
# Web server configuration defaults
webserver_package: nginx
webserver_service: nginx
webserver_port: 80
webserver_document_root: /var/www/html
webserver_server_name: localhost

# Feature flags
webserver_ssl_enabled: false
webserver_gzip_enabled: true

# Performance settings
webserver_worker_processes: auto
webserver_worker_connections: 1024
```

### vars/main.yml

Variables that shouldn't typically be overridden:

```yaml
---
# Internal role variables
webserver_config_path: /etc/nginx
webserver_sites_available: "{{ webserver_config_path }}/sites-available"
webserver_sites_enabled: "{{ webserver_config_path }}/sites-enabled"

# Required packages
webserver_dependencies:
  - nginx
  - openssl
  - python3-certbot-nginx
```

### tasks/main.yml

The main task file:

```yaml
---
- name: Install web server packages
  apt:
    name: "{{ webserver_dependencies }}"
    state: present
    update_cache: yes
  tags:
    - webserver
    - packages

- name: Create document root
  file:
    path: "{{ webserver_document_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'
  tags:
    - webserver
    - setup

- name: Copy default index page
  copy:
    src: index.html
    dest: "{{ webserver_document_root }}/index.html"
    owner: www-data
    group: www-data
    mode: '0644'
  tags:
    - webserver
    - content

- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: "{{ webserver_config_path }}/nginx.conf"
    owner: root
    group: root
    mode: '0644'
  notify: Reload nginx
  tags:
    - webserver
    - configuration

- name: Create site configuration
  template:
    src: site.conf.j2
    dest: "{{ webserver_sites_available }}/{{ webserver_server_name }}.conf"
  notify: Reload nginx
  tags:
    - webserver
    - configuration

- name: Enable site
  file:
    src: "{{ webserver_sites_available }}/{{ webserver_server_name }}.conf"
    dest: "{{ webserver_sites_enabled }}/{{ webserver_server_name }}.conf"
    state: link
  notify: Reload nginx
  tags:
    - webserver
    - configuration

- name: Start and enable nginx
  service:
    name: "{{ webserver_service }}"
    state: started
    enabled: yes
  tags:
    - webserver
    - service
```

### handlers/main.yml

Handlers for the role:

```yaml
---
- name: Restart nginx
  service:
    name: "{{ webserver_service }}"
    state: restarted

- name: Reload nginx
  service:
    name: "{{ webserver_service }}"
    state: reloaded

- name: Validate nginx config
  command: nginx -t
  changed_when: false
```

### templates/nginx.conf.j2

Jinja2 template for nginx configuration:

```jinja2
# Managed by Ansible - Do not edit manually
user www-data;
worker_processes {{ webserver_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ webserver_worker_connections }};
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

{% if webserver_gzip_enabled %}
    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript;
{% endif %}

    include {{ webserver_sites_enabled }}/*;
}
```

### files/index.html

Static file to deploy:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Welcome</title>
</head>
<body>
    <h1>Server is running!</h1>
    <p>Deployed with Ansible</p>
</body>
</html>
```

---

## Using Roles in Playbooks

### Basic Role Usage

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  
  roles:
    - webserver
```

### Roles with Variables

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  
  roles:
    - role: webserver
      vars:
        webserver_port: 8080
        webserver_server_name: example.com
```

### Alternative Syntax

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  
  roles:
    - role: webserver
      webserver_port: 8080
      
    # Short form (without vars)
    - { role: database, db_name: myapp }
```

### Multiple Roles

```yaml
---
- name: Full Stack Setup
  hosts: all
  become: yes
  
  roles:
    - common
    - security
    - webserver
    - database
    - monitoring
```

### Conditional Roles

```yaml
---
- name: Configure Servers
  hosts: all
  become: yes
  
  roles:
    - role: webserver
      when: "'webservers' in group_names"
      
    - role: database
      when: "'dbservers' in group_names"
```

### Roles with Tags

```yaml
---
- name: Deploy Application
  hosts: all
  become: yes
  
  roles:
    - role: common
      tags: always
      
    - role: webserver
      tags:
        - web
        - nginx
        
    - role: database
      tags:
        - db
        - postgresql
```

---

## Role Dependencies

Define dependencies in `meta/main.yml`:

```yaml
---
# roles/webserver/meta/main.yml
galaxy_info:
  author: Your Name
  description: Nginx web server role
  license: MIT
  min_ansible_version: "2.9"
  
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: Debian
      versions:
        - bullseye
        - bookworm

dependencies:
  - role: common
  
  - role: firewall
    vars:
      firewall_allowed_ports:
        - 80
        - 443
        
  - role: ssl_certificates
    when: webserver_ssl_enabled
```

### Dependency Behavior

- Dependencies are executed **before** the role
- Same role won't run twice (unless `allow_duplicates: true`)
- Dependencies can have their own dependencies

```yaml
# Allow running the same role multiple times
dependencies:
  - role: add_user
    vars:
      username: alice
      
  - role: add_user
    vars:
      username: bob
    allow_duplicates: true
```

---

## Roles vs Tasks

You can mix roles with tasks in a playbook:

```yaml
---
- name: Setup Server
  hosts: webservers
  become: yes
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
      tags: always
  
  roles:
    - common
    - webserver
  
  tasks:
    - name: Additional custom configuration
      template:
        src: custom.conf.j2
        dest: /etc/app/custom.conf
  
  post_tasks:
    - name: Verify installation
      uri:
        url: http://localhost
        return_content: yes
      register: result
      
    - name: Display result
      debug:
        var: result.content
```

### Execution Order

1. `pre_tasks`
2. Handlers triggered by pre_tasks
3. `roles`
4. `tasks`
5. Handlers triggered by roles and tasks
6. `post_tasks`
7. Handlers triggered by post_tasks

---

## Role Variables

### Variable Precedence (Low to High)

1. Role `defaults/main.yml` (lowest)
2. Inventory `group_vars`
3. Inventory `host_vars`
4. Playbook `vars`
5. Role `vars/main.yml`
6. Block vars
7. Task vars
8. Extra vars `-e` (highest)

### Accessing Role Variables

```yaml
# In tasks
- name: Use variable
  debug:
    msg: "Port is {{ webserver_port }}"

# In templates
server {
    listen {{ webserver_port }};
}
```

### Private Role Variables

Prefix with underscore to indicate private variables:

```yaml
# vars/main.yml
---
_webserver_internal_var: "don't override this"
webserver_public_var: "safe to override"
```

---

## Including Tasks in Roles

### Split Tasks into Multiple Files

```
roles/webserver/tasks/
├── main.yml
├── install.yml
├── configure.yml
└── service.yml
```

**tasks/main.yml:**

```yaml
---
- name: Include installation tasks
  include_tasks: install.yml
  tags:
    - webserver
    - install

- name: Include configuration tasks
  include_tasks: configure.yml
  tags:
    - webserver
    - configure

- name: Include service tasks
  include_tasks: service.yml
  tags:
    - webserver
    - service
```

**tasks/install.yml:**

```yaml
---
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
```

### Conditional Includes

```yaml
---
# tasks/main.yml
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Include OS-specific tasks
  include_tasks: "{{ ansible_os_family }}.yml"
```

```
roles/webserver/tasks/
├── main.yml
├── Debian.yml
└── RedHat.yml
```

---

## Role Templates

### Using Templates

Templates use Jinja2 syntax and can access all role variables:

```yaml
# tasks/main.yml
- name: Deploy configuration
  template:
    src: app.conf.j2           # Looks in templates/
    dest: /etc/app/app.conf
```

### Template Example

**templates/app.conf.j2:**

```jinja2
# Application Configuration
# Generated by Ansible on {{ ansible_date_time.iso8601 }}

[server]
host = {{ app_host | default('0.0.0.0') }}
port = {{ app_port }}
workers = {{ app_workers | default(ansible_processor_cores) }}

[database]
{% if app_database is defined %}
host = {{ app_database.host }}
port = {{ app_database.port }}
name = {{ app_database.name }}
{% endif %}

[features]
{% for feature in app_features | default([]) %}
{{ feature }} = enabled
{% endfor %}

[logging]
level = {{ app_log_level | default('info') }}
{% if app_debug | default(false) %}
debug = true
{% endif %}
```

---

## Role Files

### Static Files

Place files in `files/` directory:

```yaml
# tasks/main.yml
- name: Copy static configuration
  copy:
    src: nginx.conf        # Looks in files/
    dest: /etc/nginx/nginx.conf
    
- name: Copy script
  copy:
    src: backup.sh
    dest: /usr/local/bin/backup.sh
    mode: '0755'
```

### Organizing Files

```
roles/webserver/files/
├── ssl/
│   ├── dhparam.pem
│   └── ssl-params.conf
├── scripts/
│   └── health-check.sh
└── html/
    └── index.html
```

```yaml
- name: Copy SSL parameters
  copy:
    src: ssl/ssl-params.conf
    dest: /etc/nginx/snippets/ssl-params.conf
```

---

## Testing Roles

### Test Playbook

**roles/webserver/tests/test.yml:**

```yaml
---
- name: Test webserver role
  hosts: localhost
  connection: local
  become: yes
  
  vars:
    webserver_port: 8080
    webserver_server_name: test.local
  
  roles:
    - role: webserver
  
  post_tasks:
    - name: Check nginx is running
      command: systemctl is-active nginx
      register: nginx_status
      changed_when: false
      
    - name: Verify nginx is active
      assert:
        that:
          - nginx_status.stdout == "active"
        fail_msg: "Nginx is not running"
        success_msg: "Nginx is running correctly"
```

### Test Inventory

**roles/webserver/tests/inventory:**

```ini
[testservers]
localhost ansible_connection=local
```

### Running Tests

```bash
cd roles/webserver/tests
ansible-playbook -i inventory test.yml
```

---

## Ansible Galaxy

Ansible Galaxy is a hub for sharing roles.

### Installing Roles from Galaxy

```bash
# Install a role
ansible-galaxy role install geerlingguy.nginx

# Install with specific version
ansible-galaxy role install geerlingguy.nginx,4.1.0

# Install to custom path
ansible-galaxy role install geerlingguy.nginx -p ./roles/

# Install from requirements file
ansible-galaxy role install -r requirements.yml
```

### Requirements File

**requirements.yml:**

```yaml
---
roles:
  # From Galaxy
  - name: geerlingguy.nginx
    version: "4.1.0"
    
  - name: geerlingguy.postgresql
    version: "3.0.0"
    
  # From GitHub
  - name: my_custom_role
    src: https://github.com/user/ansible-role-custom
    version: main
    
  # From Git with SSH
  - name: private_role
    src: git@github.com:company/private-role.git
    scm: git
    version: v1.0.0

collections:
  - name: community.general
    version: ">=5.0.0"
```

```bash
# Install all requirements
ansible-galaxy install -r requirements.yml
```

### Publishing to Galaxy

1. Create a GitHub repository for your role
2. Follow naming convention: `ansible-role-<rolename>`
3. Add proper `meta/main.yml`
4. Import to Galaxy via the website

---

## Role Best Practices

### 1. Use Meaningful Names

```
# Good
roles/
├── nginx_webserver/
├── postgresql_database/
└── prometheus_monitoring/

# Bad
roles/
├── role1/
├── web/
└── stuff/
```

### 2. Prefix Variables with Role Name

```yaml
# defaults/main.yml

# Good - prevents conflicts
nginx_port: 80
nginx_worker_processes: auto
nginx_ssl_enabled: false

# Bad - might conflict with other roles
port: 80
workers: auto
ssl: false
```

### 3. Document Your Role

**README.md:**

```markdown
# Nginx Role

Installs and configures Nginx web server.

## Requirements

- Ubuntu 20.04+ or Debian 11+
- Ansible 2.9+

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_port` | `80` | HTTP port |
| `nginx_ssl_enabled` | `false` | Enable HTTPS |

## Dependencies

- `common`

## Example Playbook

```yaml
- hosts: webservers
  roles:
    - role: nginx
      nginx_port: 8080
```

## License

MIT
```

### 4. Use Defaults for Customization

```yaml
# defaults/main.yml - Users can override
nginx_port: 80
nginx_ssl_enabled: false

# vars/main.yml - Internal, don't override
_nginx_config_path: /etc/nginx
_nginx_user: www-data
```

### 5. Handle Multiple OS Families

```yaml
# tasks/main.yml
- name: Include OS-specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution }}.yml"
    - "{{ ansible_os_family }}.yml"
    - "default.yml"
```

```
roles/nginx/vars/
├── Debian.yml
├── RedHat.yml
├── Ubuntu-22.yml
└── default.yml
```

### 6. Make Roles Idempotent

```yaml
# Good - Idempotent
- name: Ensure nginx is installed
  apt:
    name: nginx
    state: present

# Bad - Not idempotent
- name: Install nginx
  command: apt-get install nginx
```

### 7. Use Tags Consistently

```yaml
# tasks/main.yml
- name: Install packages
  apt:
    name: nginx
    state: present
  tags:
    - nginx
    - packages
    - install

- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  tags:
    - nginx
    - configuration
```

---

## Complete Project Example

### Directory Structure

```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── production
│   └── staging
├── group_vars/
│   ├── all.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── host_vars/
│   └── web01.yml
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   └── defaults/
│   │       └── main.yml
│   ├── webserver/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   ├── files/
│   │   ├── defaults/
│   │   └── meta/
│   └── database/
│       ├── tasks/
│       ├── handlers/
│       ├── templates/
│       └── defaults/
├── site.yml
├── webservers.yml
└── dbservers.yml
```

### ansible.cfg

```ini
[defaults]
inventory = ./inventory/production
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```

### site.yml (Main Playbook)

```yaml
---
# Master playbook - runs all roles
- import_playbook: webservers.yml
- import_playbook: dbservers.yml
```

### webservers.yml

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  
  roles:
    - common
    - webserver
```

### group_vars/webservers.yml

```yaml
---
# Variables for all web servers
webserver_port: 80
webserver_ssl_enabled: true
webserver_server_name: "{{ inventory_hostname }}"

webserver_sites:
  - name: main
    document_root: /var/www/main
  - name: api
    document_root: /var/www/api
```

### Running the Project

```bash
# Run everything
ansible-playbook site.yml

# Run only webservers
ansible-playbook webservers.yml

# Run with specific tags
ansible-playbook site.yml --tags "configuration"

# Run on staging
ansible-playbook -i inventory/staging site.yml

# Dry run
ansible-playbook site.yml --check --diff
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Role** | Reusable package of Ansible content |
| **defaults/** | Default variables (easily overridden) |
| **vars/** | Role variables (harder to override) |
| **tasks/** | Main task definitions |
| **handlers/** | Event-triggered tasks |
| **templates/** | Jinja2 template files |
| **files/** | Static files to copy |
| **meta/** | Role metadata and dependencies |
| **Galaxy** | Community hub for sharing roles |

### Key Commands

```bash
# Create role skeleton
ansible-galaxy role init roles/myrole

# Install role from Galaxy
ansible-galaxy role install geerlingguy.nginx

# Install from requirements
ansible-galaxy install -r requirements.yml

# List installed roles
ansible-galaxy role list
```

Roles are essential for creating maintainable, reusable Ansible automation. Master them to build scalable infrastructure as code!
