# Ansible Galaxy

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
