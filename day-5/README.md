# Day 5: Ansible Galaxy & Ansible Lint

## Ansible Galaxy

**Ansible Galaxy** is the official hub for finding, sharing, and downloading Ansible roles and collections. It's like a package manager for Ansible content.

Website: [galaxy.ansible.com](https://galaxy.ansible.com)

---

## Why Use Galaxy?

| Benefit | Description |
|---------|-------------|
| **Save Time** | Use pre-built roles instead of writing from scratch |
| **Best Practices** | Community-maintained content follows standards |
| **Versioning** | Pin specific versions for reproducibility |
| **Quality** | Popular roles are battle-tested |
| **Documentation** | Well-documented usage and variables |

---

## Searching for Roles

### Command Line

```bash
# Search for roles
ansible-galaxy search nginx

# Search with filters
ansible-galaxy search nginx --author geerlingguy

# Get information about a role
ansible-galaxy info geerlingguy.nginx
```

### Web Interface

Browse [galaxy.ansible.com](https://galaxy.ansible.com) to:
- Search by keyword, platform, or category
- View download counts and ratings
- Read documentation and source code
- Check compatibility and dependencies

---

## Installing Roles from Galaxy

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

---

## Ansible Lint

**Ansible Lint** is a command-line tool that checks playbooks, roles, and collections for best practices and potential errors.

### Installing Ansible Lint

```bash
# Install via pip
pip install ansible-lint

# Or with pipx (isolated environment)
pipx install ansible-lint

# Verify installation
ansible-lint --version
```

### Basic Usage

```bash
# Lint a playbook
ansible-lint playbook.yml

# Lint a role
ansible-lint roles/webserver/

# Lint with specific rules
ansible-lint -r /path/to/rules playbook.yml

# List all rules
ansible-lint -L

# Lint with specific tags
ansible-lint -t yaml playbook.yml
```

### Common Lint Rules

| Rule | Description |
|------|-------------|
| `yaml` | YAML syntax and formatting |
| `name` | Task naming conventions |
| `command-instead-of-module` | Use modules instead of commands |
| `no-changed-when` | Set `changed_when` for commands |
| `risky-file-permissions` | Avoid world-writable files |
| `package-latest` | Avoid `state: latest` in production |
| `no-jinja-when` | Don't use Jinja2 in `when` |

### Example Output

```bash
$ ansible-lint playbook.yml

WARNING  Listing 5 violation(s) that are fatal
playbook.yml:5 Task/Handler: Install packages
name[missing]: All tasks should be named.

playbook.yml:10 Task/Handler: Run setup script
no-changed-when: Commands should not change things if nothing needs doing.

playbook.yml:15 Task/Handler: Copy config
risky-file-permissions: File permissions unset or incorrect.

roles/webserver/tasks/main.yml:3 Task/Handler: Install nginx
package-latest: Package installs should not use latest.

roles/webserver/tasks/main.yml:12 Task/Handler: shell task
command-instead-of-shell: Use shell only when shell functionality is required.
```

### Fixing Common Issues

```yaml
# BAD - Missing name
- apt:
    name: nginx
    state: present

# GOOD - Has descriptive name
- name: Install nginx web server
  ansible.builtin.apt:
    name: nginx
    state: present

# BAD - Using command instead of module
- name: Create directory
  command: mkdir -p /opt/myapp

# GOOD - Using file module
- name: Create application directory
  ansible.builtin.file:
    path: /opt/myapp
    state: directory
    mode: '0755'

# BAD - No changed_when
- name: Check status
  command: /opt/app/status.sh
  register: result

# GOOD - With changed_when
- name: Check application status
  ansible.builtin.command: /opt/app/status.sh
  register: result
  changed_when: false

# BAD - Package latest
- name: Install packages
  apt:
    name: nginx
    state: latest

# GOOD - Package present (pin version if needed)
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
```

### Configuration File

Create `.ansible-lint` in your project root:

```yaml
# .ansible-lint configuration
---
# Exclude paths from linting
exclude_paths:
  - .cache/
  - .git/
  - molecule/
  - tests/

# Skip specific rules
skip_list:
  - yaml[line-length]  # Allow long lines
  - name[casing]       # Allow any case in names

# Warn only (don't fail) for these rules
warn_list:
  - experimental

# Enable specific tags
enable_list:
  - no-same-owner

# Offline mode (don't download collections)
offline: false

# Set custom rules directory
# rulesdir:
#   - ./custom_rules/

# Specify profile (min, basic, moderate, safety, shared, production)
profile: moderate
```

### Profiles

Ansible Lint has predefined profiles with different strictness levels:

| Profile | Description |
|---------|-------------|
| `min` | Minimal rules, syntax only |
| `basic` | Basic best practices |
| `moderate` | Recommended for most projects |
| `safety` | Security-focused rules |
| `shared` | For shared/public content |
| `production` | Strictest, production-ready |

```bash
# Use a specific profile
ansible-lint -p production playbook.yml
```

### Integrating with CI/CD

**GitHub Actions:**
```yaml
# .github/workflows/lint.yml
name: Ansible Lint

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run ansible-lint
        uses: ansible/ansible-lint-action@v6
```

**Pre-commit Hook:**
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/ansible/ansible-lint
    rev: v6.17.0
    hooks:
      - id: ansible-lint
        files: \.(yaml|yml)$
```

### Best Practices for Clean Lint

1. **Always name tasks** - Every task should have a descriptive name
2. **Use FQCN** - Use fully qualified collection names (e.g., `ansible.builtin.apt`)
3. **Set changed_when** - For command/shell tasks that don't change state
4. **Use modules** - Prefer modules over raw commands
5. **Explicit permissions** - Always set file modes explicitly
6. **Avoid `latest`** - Use `present` or pin versions
7. **Quote strings** - Especially paths and values with special characters

---

## Summary

| Tool | Purpose |
|------|---------|
| **Galaxy** | Find and share roles/collections |
| **ansible-lint** | Check for best practices and errors |

### Key Commands

```bash
# Galaxy
ansible-galaxy search nginx
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy collection install community.docker
ansible-galaxy install -r requirements.yml

# Lint
ansible-lint playbook.yml
ansible-lint -L                    # List rules
ansible-lint -p production .       # Use production profile
ansible-lint --fix playbook.yml   # Auto-fix some issues
```

Use Galaxy to leverage community content and ansible-lint to ensure quality!
