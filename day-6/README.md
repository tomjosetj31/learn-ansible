# Day 6: Ansible Collections

## What are Ansible Collections?

**Collections** are the modern way to package and distribute Ansible content. They bundle together related modules, plugins, roles, and playbooks into a single distributable unit.

Think of collections as **namespaced packages** — they organize Ansible content under a consistent namespace like `community.general` or `amazon.aws`.

---

## Why Collections?

Before collections, Ansible shipped with all modules in a single package. This caused:
- Slow release cycles
- Bloated installations
- Version conflicts

Collections solve these problems:

| Benefit | Description |
|---------|-------------|
| **Modular** | Install only what you need |
| **Versioned** | Independent versioning per collection |
| **Namespaced** | No naming conflicts between content |
| **Faster Updates** | Collections update independently of Ansible |
| **Community Driven** | Anyone can create and share collections |

---

## Collections vs Roles

| Feature | Collections | Roles |
|---------|-------------|-------|
| **Scope** | Modules, plugins, roles, playbooks | Tasks, handlers, files, templates |
| **Namespace** | `namespace.collection` | Role name only |
| **Distribution** | Galaxy, Automation Hub, Git | Galaxy, Git |
| **Contains** | Multiple roles and plugins | Single role |
| **Use Case** | Platform/vendor content | Specific configuration |

---

## Collection Structure

```
namespace/
└── collection_name/
    ├── galaxy.yml           # Collection metadata
    ├── README.md            # Documentation
    ├── CHANGELOG.rst        # Version history
    ├── docs/                # Additional documentation
    ├── plugins/
    │   ├── modules/         # Custom modules
    │   ├── inventory/       # Inventory plugins
    │   ├── lookup/          # Lookup plugins
    │   ├── filter/          # Filter plugins
    │   ├── callback/        # Callback plugins
    │   └── connection/      # Connection plugins
    ├── roles/               # Bundled roles
    │   ├── role1/
    │   └── role2/
    ├── playbooks/           # Bundled playbooks
    └── tests/               # Integration tests
```

---

## Finding Collections

### Ansible Galaxy

Browse collections at [galaxy.ansible.com](https://galaxy.ansible.com)

### Popular Collections

| Collection | Description |
|------------|-------------|
| `ansible.builtin` | Core modules (included with Ansible) |
| `community.general` | General-purpose community modules |
| `community.mysql` | MySQL database modules |
| `community.postgresql` | PostgreSQL database modules |
| `community.docker` | Docker container modules |
| `amazon.aws` | Amazon Web Services modules |
| `azure.azcollection` | Microsoft Azure modules |
| `google.cloud` | Google Cloud Platform modules |
| `kubernetes.core` | Kubernetes modules |
| `ansible.posix` | POSIX system modules |
| `ansible.windows` | Windows modules |

### Search for Collections

```bash
# Search Galaxy for collections
ansible-galaxy collection search docker

# Search with specific namespace
ansible-galaxy collection search community.docker

# Get collection info
ansible-galaxy collection info community.docker
```

---

## Installing Collections

### From Ansible Galaxy

```bash
# Install a collection
ansible-galaxy collection install community.general

# Install specific version
ansible-galaxy collection install community.general:5.0.0

# Install to custom path
ansible-galaxy collection install community.docker -p ./collections/

# Force reinstall
ansible-galaxy collection install community.general --force

# Upgrade collection
ansible-galaxy collection install community.general --upgrade
```

### From Requirements File

**requirements.yml:**

```yaml
---
collections:
  # From Galaxy
  - name: community.general
    version: ">=6.0.0"
    
  - name: community.docker
    version: "3.4.0"
    
  - name: amazon.aws
    version: "5.0.0"
    
  # From Git repository
  - name: my_namespace.my_collection
    source: https://github.com/user/my-collection.git
    type: git
    version: main
    
  # From tarball URL
  - name: my_namespace.my_collection
    source: https://example.com/collections/my_collection-1.0.0.tar.gz
    type: url
    
  # From local file
  - name: my_namespace.my_collection
    source: ./local_collections/my_collection-1.0.0.tar.gz
    type: file
```

```bash
# Install from requirements file
ansible-galaxy collection install -r requirements.yml

# Install with verbose output
ansible-galaxy collection install -r requirements.yml -v
```

### From Automation Hub (Red Hat)

```bash
# Configure in ansible.cfg
[galaxy]
server_list = automation_hub, galaxy

[galaxy_server.automation_hub]
url=https://console.redhat.com/api/automation-hub/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=your_token_here

[galaxy_server.galaxy]
url=https://galaxy.ansible.com/

# Then install
ansible-galaxy collection install redhat.rhel_system_roles
```

---

## Using Collections

### Fully Qualified Collection Names (FQCN)

Always use FQCN for clarity and consistency:

```yaml
---
- name: Example Playbook
  hosts: all
  
  tasks:
    # Using FQCN (recommended)
    - name: Create a Docker network
      community.docker.docker_network:
        name: my_network
        
    - name: Install package with apt
      ansible.builtin.apt:
        name: nginx
        state: present
        
    - name: Copy file
      ansible.builtin.copy:
        src: file.txt
        dest: /tmp/file.txt
```

### Short Names (with collections keyword)

```yaml
---
- name: Example Playbook
  hosts: all
  collections:
    - community.docker
    - community.general
  
  tasks:
    # Can use short names now
    - name: Create Docker network
      docker_network:
        name: my_network
        
    - name: Create user
      user:
        name: john
        state: present
```

**Note**: FQCN is always recommended for clarity and to avoid ambiguity.

---

## Collection Modules

### Using Modules from Collections

```yaml
---
- name: Docker Management
  hosts: docker_hosts
  become: yes
  
  tasks:
    - name: Install Docker SDK for Python
      ansible.builtin.pip:
        name: docker
        state: present
        
    - name: Pull Docker image
      community.docker.docker_image:
        name: nginx
        tag: latest
        source: pull
        
    - name: Create container
      community.docker.docker_container:
        name: web
        image: nginx:latest
        ports:
          - "8080:80"
        state: started
        
    - name: Create Docker network
      community.docker.docker_network:
        name: app_network
        driver: bridge
```

### AWS Collection Example

```yaml
---
- name: AWS Infrastructure
  hosts: localhost
  connection: local
  
  tasks:
    - name: Create VPC
      amazon.aws.ec2_vpc_net:
        name: my-vpc
        cidr_block: 10.0.0.0/16
        region: us-east-1
        state: present
      register: vpc
      
    - name: Create subnet
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: 10.0.1.0/24
        az: us-east-1a
        state: present
        
    - name: Launch EC2 instance
      amazon.aws.ec2_instance:
        name: web-server
        instance_type: t3.micro
        image_id: ami-12345678
        vpc_subnet_id: "{{ subnet.subnet.id }}"
        state: running
```

### Kubernetes Collection Example

```yaml
---
- name: Deploy to Kubernetes
  hosts: localhost
  connection: local
  
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: my-app
            
    - name: Deploy application
      kubernetes.core.k8s:
        state: present
        src: deployment.yml
        namespace: my-app
        
    - name: Wait for deployment
      kubernetes.core.k8s_info:
        kind: Deployment
        name: my-app
        namespace: my-app
      register: deployment
      until: deployment.resources[0].status.readyReplicas == deployment.resources[0].status.replicas
      retries: 10
      delay: 30
```

---

## Collection Plugins

Collections can include various plugin types:

### Lookup Plugins

```yaml
---
- name: Use lookup plugins
  hosts: localhost
  
  tasks:
    - name: Get password from file
      debug:
        msg: "{{ lookup('community.general.passwordstore', 'secret/password') }}"
        
    - name: Read from HashiCorp Vault
      debug:
        msg: "{{ lookup('community.hashi_vault.vault_read', 'secret/data/myapp') }}"
```

### Filter Plugins

```yaml
---
- name: Use filter plugins
  hosts: localhost
  vars:
    my_list:
      - { name: "alice", age: 30 }
      - { name: "bob", age: 25 }
      - { name: "charlie", age: 35 }
  
  tasks:
    - name: Sort by age
      debug:
        msg: "{{ my_list | community.general.sort_by('age') }}"
        
    - name: Convert to JSON
      debug:
        msg: "{{ my_list | ansible.builtin.to_nice_json }}"
```

### Inventory Plugins

```yaml
# inventory/aws_ec2.yml
---
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.region
    prefix: region
hostnames:
  - tag:Name
  - private-ip-address
```

```bash
# Use dynamic inventory
ansible-playbook -i inventory/aws_ec2.yml playbook.yml
```

### Callback Plugins

```ini
# ansible.cfg
[defaults]
# Use callback from collection
stdout_callback = community.general.yaml
callbacks_enabled = community.general.log_plays
```

---

## Collection Roles

Collections can include roles that you can use in playbooks:

```yaml
---
- name: Use collection role
  hosts: webservers
  
  roles:
    # FQCN for collection role
    - role: geerlingguy.docker.docker
      vars:
        docker_edition: ce
        
    # Another collection role
    - role: community.mysql.mysql
      vars:
        mysql_root_password: secret
```

### Listing Roles in a Collection

```bash
# View collection contents
ansible-doc -l -t role community.mysql
```

---

## Managing Collections

### List Installed Collections

```bash
# List all installed collections
ansible-galaxy collection list

# List with path
ansible-galaxy collection list -p ./collections

# Output example:
# Collection        Version
# ----------------- -------
# amazon.aws        5.0.0
# community.docker  3.4.0
# community.general 6.0.0
```

### Verify Collection

```bash
# Verify collection integrity
ansible-galaxy collection verify community.general
```

### Remove Collection

```bash
# Collections must be removed manually
rm -rf ~/.ansible/collections/ansible_collections/community/docker/
```

### Collection Paths

Default search paths for collections:

```bash
# View collection paths
ansible-config dump | grep COLLECTIONS_PATHS

# Default paths:
# - ./collections
# - ~/.ansible/collections
# - /usr/share/ansible/collections
```

Configure in `ansible.cfg`:

```ini
[defaults]
collections_paths = ./collections:~/.ansible/collections:/usr/share/ansible/collections
```

---

## Creating Collections

### Initialize Collection

```bash
# Create collection skeleton
ansible-galaxy collection init my_namespace.my_collection

# Structure created:
# my_namespace/
# └── my_collection/
#     ├── docs/
#     ├── galaxy.yml
#     ├── plugins/
#     │   └── README.md
#     ├── README.md
#     └── roles/
```

### galaxy.yml

```yaml
---
namespace: my_namespace
name: my_collection
version: 1.0.0
readme: README.md
authors:
  - Your Name <your.email@example.com>
description: My custom Ansible collection
license:
  - MIT
license_file: LICENSE
tags:
  - infrastructure
  - automation
  - devops
dependencies:
  community.general: ">=5.0.0"
  ansible.posix: ">=1.0.0"
repository: https://github.com/yourname/my-collection
documentation: https://github.com/yourname/my-collection/blob/main/README.md
homepage: https://github.com/yourname/my-collection
issues: https://github.com/yourname/my-collection/issues
build_ignore:
  - .gitignore
  - .github
  - tests/output
```

### Adding a Custom Module

**plugins/modules/my_module.py:**

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: my_module
short_description: My custom module
description:
  - This module does something useful
version_added: "1.0.0"
author:
  - Your Name (@yourname)
options:
  name:
    description:
      - Name of the resource
    required: true
    type: str
  state:
    description:
      - Desired state
    choices: ['present', 'absent']
    default: present
    type: str
'''

EXAMPLES = r'''
- name: Create resource
  my_namespace.my_collection.my_module:
    name: my_resource
    state: present
'''

RETURN = r'''
message:
  description: Result message
  returned: always
  type: str
'''

from ansible.module_utils.basic import AnsibleModule


def main():
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(type='str', required=True),
            state=dict(type='str', default='present', choices=['present', 'absent']),
        ),
        supports_check_mode=True,
    )
    
    name = module.params['name']
    state = module.params['state']
    
    result = dict(
        changed=False,
        message=f"Resource {name} is {state}"
    )
    
    if module.check_mode:
        module.exit_json(**result)
    
    # Your logic here
    result['changed'] = True
    
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

### Adding a Role to Collection

```bash
# Create role inside collection
cd my_namespace/my_collection/roles
ansible-galaxy role init my_role
```

### Build Collection

```bash
# Build collection archive
ansible-galaxy collection build my_namespace/my_collection

# Output: my_namespace-my_collection-1.0.0.tar.gz
```

### Publish Collection

```bash
# Publish to Galaxy
ansible-galaxy collection publish my_namespace-my_collection-1.0.0.tar.gz --api-key=your_api_key

# Publish to private Automation Hub
ansible-galaxy collection publish my_namespace-my_collection-1.0.0.tar.gz \
  --server=https://hub.example.com \
  --api-key=your_api_key
```

---

## Collection Dependencies

### Specifying Dependencies

In `galaxy.yml`:

```yaml
dependencies:
  community.general: ">=5.0.0,<7.0.0"
  ansible.posix: ">=1.3.0"
  amazon.aws: ">=4.0.0"
```

### In Playbook Requirements

**requirements.yml:**

```yaml
---
collections:
  - name: community.docker
    version: ">=3.0.0"
    
  - name: amazon.aws
    version: ">=5.0.0"
    
  - name: community.general

roles:
  - name: geerlingguy.docker
    version: "6.0.0"
```

---

## Ansible.cfg for Collections

```ini
[defaults]
# Collection paths
collections_paths = ./collections:~/.ansible/collections

# Use FQCN for clearer output
display_args_to_stdout = True

[galaxy]
# Galaxy server configuration
server_list = galaxy

[galaxy_server.galaxy]
url = https://galaxy.ansible.com/
```

---

## Complete Example Project

### Project Structure

```
my-project/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── production
│   └── group_vars/
│       └── all.yml
├── collections/
│   └── ansible_collections/
│       └── (installed collections)
├── playbooks/
│   ├── site.yml
│   ├── deploy.yml
│   └── infrastructure.yml
└── roles/
    └── app/
```

### ansible.cfg

```ini
[defaults]
inventory = ./inventory/production
roles_path = ./roles
collections_paths = ./collections
host_key_checking = False

[galaxy]
server_list = galaxy

[galaxy_server.galaxy]
url = https://galaxy.ansible.com/
```

### requirements.yml

```yaml
---
collections:
  - name: community.general
    version: ">=6.0.0"
  - name: community.docker
    version: ">=3.0.0"
  - name: amazon.aws
    version: ">=5.0.0"
  - name: kubernetes.core
    version: ">=2.0.0"

roles:
  - name: geerlingguy.docker
```

### Setup Script

```bash
#!/bin/bash
# setup.sh - Initialize project

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install Ansible
pip install ansible ansible-lint

# Install collections and roles
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-galaxy role install -r requirements.yml -p ./roles

echo "Setup complete!"
```

### playbooks/site.yml

```yaml
---
- name: Configure Infrastructure
  hosts: all
  become: yes
  
  pre_tasks:
    - name: Update package cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

  tasks:
    - name: Install common packages
      ansible.builtin.package:
        name:
          - vim
          - curl
          - wget
          - git
        state: present

- name: Configure Docker Hosts
  hosts: docker
  become: yes
  
  tasks:
    - name: Install Docker
      ansible.builtin.include_role:
        name: geerlingguy.docker
        
    - name: Create application network
      community.docker.docker_network:
        name: app_network
        state: present
        
    - name: Deploy application container
      community.docker.docker_container:
        name: myapp
        image: myapp:latest
        networks:
          - name: app_network
        ports:
          - "8080:80"
        state: started
        restart_policy: always

- name: Configure Kubernetes
  hosts: localhost
  connection: local
  
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: production
            
    - name: Apply deployment
      kubernetes.core.k8s:
        state: present
        src: "{{ playbook_dir }}/../manifests/deployment.yml"
```

---

## Best Practices

### 1. Always Use FQCN

```yaml
# Good - Clear and unambiguous
- name: Copy file
  ansible.builtin.copy:
    src: file.txt
    dest: /tmp/file.txt

# Avoid - Ambiguous
- name: Copy file
  copy:
    src: file.txt
    dest: /tmp/file.txt
```

### 2. Pin Collection Versions

```yaml
# requirements.yml
collections:
  # Good - Pinned version
  - name: community.docker
    version: "3.4.0"
    
  # Acceptable - Version range
  - name: community.general
    version: ">=6.0.0,<7.0.0"
    
  # Risky - No version (gets latest)
  - name: amazon.aws
```

### 3. Document Collection Requirements

```markdown
# README.md

## Requirements

This project requires the following Ansible collections:

| Collection | Version | Purpose |
|------------|---------|---------|
| community.docker | >=3.0.0 | Docker management |
| amazon.aws | >=5.0.0 | AWS infrastructure |
| kubernetes.core | >=2.0.0 | K8s deployments |

Install with: `ansible-galaxy collection install -r requirements.yml`
```

### 4. Use Collection Offline

```bash
# Download collections for offline use
ansible-galaxy collection download -r requirements.yml -p ./offline_collections/

# Install from downloaded files
ansible-galaxy collection install -r requirements.yml -p ./collections --offline
```

### 5. Test Before Upgrading

```bash
# Check what would be upgraded
ansible-galaxy collection install -r requirements.yml --upgrade --dry-run

# Test in staging first
ansible-playbook site.yml --check
```

---

## Troubleshooting

### Module Not Found

```bash
# Error: couldn't resolve module/action 'community.docker.docker_container'

# Solution: Install the collection
ansible-galaxy collection install community.docker
```

### Version Conflicts

```bash
# Check installed versions
ansible-galaxy collection list

# Force reinstall specific version
ansible-galaxy collection install community.general:6.0.0 --force
```

### Collection Path Issues

```bash
# Verify collection paths
ansible-config dump | grep COLLECTIONS

# Check if collection is found
ansible-doc -l | grep docker

# List modules in a collection
ansible-doc -l community.docker
```

### View Module Documentation

```bash
# View module docs
ansible-doc community.docker.docker_container

# View all modules in collection
ansible-doc -l community.docker
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Collection** | Namespaced package of Ansible content |
| **FQCN** | Fully Qualified Collection Name (e.g., `community.docker.docker_container`) |
| **galaxy.yml** | Collection metadata file |
| **plugins/** | Custom modules, filters, lookups, etc. |
| **roles/** | Bundled roles within collection |
| **requirements.yml** | Dependency specification file |

### Key Commands

```bash
# Search collections
ansible-galaxy collection search docker

# Install collection
ansible-galaxy collection install community.docker

# Install from requirements
ansible-galaxy collection install -r requirements.yml

# List installed collections
ansible-galaxy collection list

# Get collection info
ansible-galaxy collection info community.docker

# Build collection
ansible-galaxy collection build

# Publish collection
ansible-galaxy collection publish my-collection-1.0.0.tar.gz
```

Collections are the future of Ansible content distribution. They provide better organization, versioning, and namespacing for sharing automation content!

## Ansible Vault

With **Ansible Vault**, you can keep sensitive data such as passwords or API keys in encrypted files, instead of plain text YAML. Vault helps you safely share playbooks without exposing secrets.

### Why Vault?

- Protect secrets in code repositories
- Decrypt files only when needed (at runtime)
- Use encrypted variables in playbooks and roles

### Common Vault Commands

```bash
# Create a new encrypted file
ansible-vault create secrets.yml

# Edit an existing encrypted file
ansible-vault edit secrets.yml

# Encrypt an existing file
ansible-vault encrypt vars.yml

# Decrypt a file
ansible-vault decrypt secrets.yml

# View the contents without editing
ansible-vault view secrets.yml
```

### Using Vault in Playbooks

You can include vault-encrypted variable files just like any other. When running a playbook referencing them, use:

```bash
ansible-playbook playbook.yml --ask-vault-pass
```
Or with a vault password file:
```bash
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass.txt
```

### Encrypting only certain variables

You can encrypt just the values, not the whole file:

```yaml
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          356632653762...
```

### Pro Tips

- Vault supports multiple passwords/files for different environments via `--vault-id`.
- Don’t commit your secrets in plaintext.
- Vault files are YAML, just encrypted.

**Vault** enables secure automation—protect your secrets without sacrificing collaboration!

---

## Ansible Variables

Variables in Ansible allow you to manage dynamic values across your playbooks, roles, and inventories. They make your automation flexible, reusable, and maintainable.

---

### Variable Basics

#### Defining Variables

```yaml
---
- name: Variable Examples
  hosts: all
  vars:
    # String
    app_name: myapp
    
    # Number
    app_port: 8080
    
    # Boolean
    debug_mode: true
    
    # List
    packages:
      - nginx
      - git
      - curl
    
    # Dictionary
    database:
      host: localhost
      port: 5432
      name: mydb
      user: admin
```

#### Using Variables

Variables are referenced using Jinja2 syntax `{{ variable_name }}`:

```yaml
tasks:
  - name: Print app name
    ansible.builtin.debug:
      msg: "Application: {{ app_name }} running on port {{ app_port }}"
      
  - name: Install packages
    ansible.builtin.apt:
      name: "{{ packages }}"
      state: present
      
  - name: Connect to database
    ansible.builtin.debug:
      msg: "Connecting to {{ database.host }}:{{ database.port }}"
```

---

### Where to Define Variables

#### 1. Playbook Variables (`vars`)

```yaml
---
- name: Playbook with vars
  hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
    
  tasks:
    - name: Configure port
      ansible.builtin.debug:
        msg: "Port: {{ http_port }}"
```

#### 2. Variable Files (`vars_files`)

```yaml
---
- name: Playbook with vars_files
  hosts: webservers
  vars_files:
    - vars/common.yml
    - vars/production.yml
    - "vars/{{ ansible_os_family }}.yml"
```

**vars/common.yml:**
```yaml
---
app_name: myapp
app_version: "1.2.3"
log_level: info
```

#### 3. Inventory Variables

**inventory/hosts:**
```ini
[webservers]
web01 ansible_host=192.168.1.10 http_port=80
web02 ansible_host=192.168.1.11 http_port=8080

[webservers:vars]
app_env=production
max_connections=1000

[dbservers]
db01 ansible_host=192.168.1.20

[all:vars]
ansible_user=deploy
ansible_python_interpreter=/usr/bin/python3
```

#### 4. Group Variables (`group_vars/`)

```
inventory/
├── hosts
├── group_vars/
│   ├── all.yml           # All hosts
│   ├── webservers.yml    # Webserver group
│   └── dbservers.yml     # Database group
```

**group_vars/webservers.yml:**
```yaml
---
http_port: 80
document_root: /var/www/html
ssl_enabled: true
```

#### 5. Host Variables (`host_vars/`)

```
inventory/
├── hosts
├── host_vars/
│   ├── web01.yml
│   └── web02.yml
```

**host_vars/web01.yml:**
```yaml
---
http_port: 80
server_name: www.example.com
ssl_certificate: /etc/ssl/certs/web01.crt
```

#### 6. Role Variables

```
roles/webserver/
├── defaults/
│   └── main.yml    # Default values (easily overridden)
└── vars/
    └── main.yml    # Role variables (harder to override)
```

#### 7. Command Line Variables (`-e` or `--extra-vars`)

```bash
# Single variable
ansible-playbook site.yml -e "app_version=2.0.0"

# Multiple variables
ansible-playbook site.yml -e "app_version=2.0.0 debug=true"

# JSON format
ansible-playbook site.yml -e '{"app_version": "2.0.0", "replicas": 3}'

# From file
ansible-playbook site.yml -e "@vars/override.yml"
```

---

### Variable Precedence

Ansible has a specific order for variable precedence (lowest to highest):

| Priority | Source |
|----------|--------|
| 1 | Role defaults (`defaults/main.yml`) |
| 2 | Inventory file or script group vars |
| 3 | Inventory `group_vars/all` |
| 4 | Playbook `group_vars/all` |
| 5 | Inventory `group_vars/*` |
| 6 | Playbook `group_vars/*` |
| 7 | Inventory file or script host vars |
| 8 | Inventory `host_vars/*` |
| 9 | Playbook `host_vars/*` |
| 10 | Host facts / cached `set_facts` |
| 11 | Play vars |
| 12 | Play `vars_prompt` |
| 13 | Play `vars_files` |
| 14 | Role vars (`vars/main.yml`) |
| 15 | Block vars |
| 16 | Task vars |
| 17 | `include_vars` |
| 18 | `set_facts` / registered vars |
| 19 | Role parameters |
| 20 | `include` parameters |
| 21 | Extra vars (`-e`) — **always wins** |

**Remember**: Extra vars (`-e`) always have the highest precedence!

---

### Special Variables

Ansible provides many built-in variables:

#### Magic Variables

```yaml
tasks:
  - name: Show magic variables
    ansible.builtin.debug:
      msg: |
        Hostname: {{ inventory_hostname }}
        Short hostname: {{ inventory_hostname_short }}
        Groups: {{ group_names }}
        All hosts: {{ groups['all'] }}
        Hostvars: {{ hostvars[inventory_hostname] }}
        Play hosts: {{ ansible_play_hosts }}
        Playbook dir: {{ playbook_dir }}
        Role path: {{ role_path }}
```

#### Connection Variables

```yaml
# Common connection variables
ansible_host: 192.168.1.10
ansible_port: 22
ansible_user: deploy
ansible_password: secret
ansible_ssh_private_key_file: ~/.ssh/id_rsa
ansible_become: yes
ansible_become_user: root
ansible_become_method: sudo
ansible_python_interpreter: /usr/bin/python3
```

#### Facts (Gathered from Hosts)

```yaml
tasks:
  - name: Show system facts
    ansible.builtin.debug:
      msg: |
        OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
        Architecture: {{ ansible_architecture }}
        Kernel: {{ ansible_kernel }}
        Memory: {{ ansible_memtotal_mb }} MB
        CPUs: {{ ansible_processor_vcpus }}
        IPv4: {{ ansible_default_ipv4.address }}
        Hostname: {{ ansible_hostname }}
        FQDN: {{ ansible_fqdn }}
```

---

### Registering Variables

Capture task output into variables:

```yaml
tasks:
  - name: Get disk usage
    ansible.builtin.command: df -h /
    register: disk_info
    
  - name: Display disk info
    ansible.builtin.debug:
      var: disk_info
      
  - name: Show just stdout
    ansible.builtin.debug:
      msg: "{{ disk_info.stdout }}"
      
  - name: Check return code
    ansible.builtin.debug:
      msg: "Command succeeded"
    when: disk_info.rc == 0
```

#### Registered Variable Structure

```yaml
# Typical registered variable contains:
disk_info:
  changed: false
  cmd: ["df", "-h", "/"]
  rc: 0                      # Return code
  stdout: "..."              # Standard output
  stdout_lines: [...]        # Output as list
  stderr: ""                 # Standard error
  stderr_lines: []           # Error as list
  start: "2024-01-01..."     # Start time
  end: "2024-01-01..."       # End time
  delta: "0:00:00.123"       # Duration
```

---

### Setting Facts

Create variables dynamically during play execution:

```yaml
tasks:
  - name: Set a fact
    ansible.builtin.set_fact:
      my_custom_var: "Hello World"
      calculated_value: "{{ ansible_memtotal_mb * 0.8 | int }}"
      
  - name: Use the fact
    ansible.builtin.debug:
      msg: "Value: {{ my_custom_var }}, Memory limit: {{ calculated_value }} MB"
      
  - name: Set fact from command
    ansible.builtin.shell: hostname -f
    register: hostname_result
    
  - name: Store hostname as fact
    ansible.builtin.set_fact:
      server_fqdn: "{{ hostname_result.stdout }}"
      cacheable: yes  # Persist across plays
```

---

### Variable Prompts

Ask for input when running playbooks:

```yaml
---
- name: Deployment with prompts
  hosts: webservers
  vars_prompt:
    - name: release_version
      prompt: "Enter release version"
      default: "1.0.0"
      private: no
      
    - name: db_password
      prompt: "Enter database password"
      private: yes
      encrypt: sha512_crypt
      confirm: yes
      
  tasks:
    - name: Deploy version
      ansible.builtin.debug:
        msg: "Deploying version {{ release_version }}"
```

---

### Variable Filters

Transform variables using Jinja2 filters:

```yaml
vars:
  my_string: "hello world"
  my_list: [3, 1, 4, 1, 5, 9]
  my_dict:
    name: John
    age: 30

tasks:
  # String filters
  - ansible.builtin.debug:
      msg: |
        Upper: {{ my_string | upper }}
        Title: {{ my_string | title }}
        Replace: {{ my_string | replace('world', 'ansible') }}
        Length: {{ my_string | length }}
        
  # List filters
  - ansible.builtin.debug:
      msg: |
        Sorted: {{ my_list | sort }}
        Unique: {{ my_list | unique }}
        First: {{ my_list | first }}
        Last: {{ my_list | last }}
        Max: {{ my_list | max }}
        Min: {{ my_list | min }}
        Sum: {{ my_list | sum }}
        
  # Default values
  - ansible.builtin.debug:
      msg: "{{ undefined_var | default('fallback_value') }}"
      
  # Type conversions
  - ansible.builtin.debug:
      msg: |
        To int: {{ "42" | int }}
        To float: {{ "3.14" | float }}
        To bool: {{ "yes" | bool }}
        To JSON: {{ my_dict | to_json }}
        To YAML: {{ my_dict | to_yaml }}
        
  # Path manipulation
  - ansible.builtin.debug:
      msg: |
        Basename: {{ "/path/to/file.txt" | basename }}
        Dirname: {{ "/path/to/file.txt" | dirname }}
        Extension: {{ "/path/to/file.txt" | splitext | last }}
        
  # Regex
  - ansible.builtin.debug:
      msg: "{{ my_string | regex_replace('world', 'universe') }}"
```

---

### Combining Variables

```yaml
vars:
  base_packages:
    - vim
    - git
    
  extra_packages:
    - docker
    - kubectl
    
  all_packages: "{{ base_packages + extra_packages }}"
  
  default_config:
    timeout: 30
    retries: 3
    
  custom_config:
    timeout: 60
    
  merged_config: "{{ default_config | combine(custom_config) }}"
  # Result: { timeout: 60, retries: 3 }
```

---

### Dictionary Access

```yaml
vars:
  users:
    alice:
      role: admin
      email: alice@example.com
    bob:
      role: developer
      email: bob@example.com

tasks:
  # Dot notation
  - ansible.builtin.debug:
      msg: "Alice role: {{ users.alice.role }}"
      
  # Bracket notation (safer for special characters)
  - ansible.builtin.debug:
      msg: "Bob email: {{ users['bob']['email'] }}"
      
  # Dynamic key access
  - ansible.builtin.debug:
      msg: "{{ users[username].role }}"
    vars:
      username: alice
      
  # Loop over dictionary
  - ansible.builtin.debug:
      msg: "{{ item.key }}: {{ item.value.email }}"
    loop: "{{ users | dict2items }}"
```

---

### Conditional Variables

```yaml
vars:
  # Ternary operator
  env_name: "{{ 'production' if is_production else 'staging' }}"
  
  # Default with condition
  http_port: "{{ custom_port | default(443 if ssl_enabled else 80) }}"
  
tasks:
  # Define variable based on OS
  - name: Set package manager
    ansible.builtin.set_fact:
      pkg_manager: "{{ 'apt' if ansible_os_family == 'Debian' else 'yum' }}"
      
  # Use omit to skip parameters
  - name: Create user
    ansible.builtin.user:
      name: "{{ username }}"
      groups: "{{ user_groups | default(omit) }}"
      shell: "{{ user_shell | default('/bin/bash') }}"
```

---

### Variable Files Organization

Recommended structure:

```
project/
├── inventory/
│   ├── production/
│   │   ├── hosts
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── vars.yml
│   │   │   │   └── vault.yml    # Encrypted secrets
│   │   │   ├── webservers.yml
│   │   │   └── dbservers.yml
│   │   └── host_vars/
│   │       ├── web01.yml
│   │       └── db01.yml
│   └── staging/
│       ├── hosts
│       └── group_vars/
│           └── all.yml
├── vars/
│   ├── common.yml
│   ├── Debian.yml
│   └── RedHat.yml
└── playbook.yml
```

---

### Debugging Variables

```yaml
tasks:
  # Print single variable
  - ansible.builtin.debug:
      var: my_variable
      
  # Print with message
  - ansible.builtin.debug:
      msg: "The value is: {{ my_variable }}"
      
  # Print all variables (verbose)
  - ansible.builtin.debug:
      var: hostvars[inventory_hostname]
      
  # Check variable type
  - ansible.builtin.debug:
      msg: "Type: {{ my_variable | type_debug }}"
      
  # Assert variable conditions
  - ansible.builtin.assert:
      that:
        - my_variable is defined
        - my_variable | length > 0
        - my_port | int > 0 and my_port | int < 65536
      fail_msg: "Variable validation failed"
      success_msg: "All variables are valid"
```

---

### Variable Best Practices

#### 1. Use Descriptive Names

```yaml
# Good
webserver_http_port: 80
database_connection_timeout: 30
app_max_upload_size_mb: 100

# Bad
port: 80
timeout: 30
size: 100
```

#### 2. Prefix Role Variables

```yaml
# Good - clear ownership
nginx_worker_processes: auto
nginx_client_max_body_size: 50m
postgresql_max_connections: 100

# Bad - might conflict
worker_processes: auto
max_connections: 100
```

#### 3. Use Defaults Wisely

```yaml
# Provide sensible defaults
http_port: "{{ custom_http_port | default(80) }}"
log_level: "{{ app_log_level | default('info') }}"
```

#### 4. Validate Variables

```yaml
- name: Validate required variables
  ansible.builtin.assert:
    that:
      - app_name is defined
      - app_name | length > 0
      - app_port is defined
      - app_port | int > 0
    fail_msg: "Required variables are missing or invalid"
```

#### 5. Document Variables

```yaml
---
# vars/main.yml
#
# Application Configuration
#

# Application name (required)
# Example: myapp
app_name: ""

# HTTP port for the application
# Default: 8080
# Range: 1-65535
app_port: 8080

# Enable debug mode
# Warning: Do not enable in production
app_debug: false
```

---

### Variables Summary

| Type | Location | Purpose |
|------|----------|---------|
| Play vars | `vars:` in playbook | Play-specific values |
| Var files | `vars_files:` | External variable files |
| Group vars | `group_vars/` | Per-group settings |
| Host vars | `host_vars/` | Per-host settings |
| Role defaults | `roles/x/defaults/` | Default values |
| Role vars | `roles/x/vars/` | Role-specific values |
| Extra vars | `-e` flag | Override any variable |
| Registered | `register:` | Capture task output |
| Facts | `set_fact:` | Dynamic values |
| Magic vars | Built-in | Ansible internals |

Master variables to create flexible, reusable Ansible automation!