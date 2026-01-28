# Learn Ansible

A comprehensive beginner-to-intermediate guide for learning Ansible automation.

---

## 📚 Course Contents

| Day | Topic | Description |
|-----|-------|-------------|
| [Day 1](./day-1/README.md) | **Introduction to Ansible** | Installation, What is Ansible, Why Ansible, How it works, Shell vs Python vs Ansible |
| [Day 2](./day-2/README.md) | **Architecture & Configuration** | ansible.cfg, Architecture, Authentication, Inventory, Ad-hoc commands |
| [Day 3](./day-3/README.md) | **YAML & Playbooks** | YAML basics, Playbook structure, Tasks, Handlers, Variables, Conditionals, Loops |
| [Day 4](./day-4/README.md) | **Ansible Roles** | Role structure, Creating roles, Using roles, Role dependencies, Best practices |
| [Day 5](./day-5/README.md) | **Ansible Galaxy & Lint** | Finding roles, Installing from Galaxy, Requirements files, Ansible Lint, Code quality |
| [Day 6](./day-6/README.md) | **Collections, Vault & Variables** | Collections, Ansible Vault for secrets, Variable types, Precedence, Filters |
| [Day 7](./day-7/README.md) | **Jinja2 Templates & Modules** | Jinja2 templating, Filters, Common modules reference |
| [Day 8](./day-8/README.md) | **Error Handling** | ignore_errors, failed_when, Blocks, Retry/Until, Assertions, Practice exercises |

---

## 🚀 Quick Start

### Prerequisites

- Linux, macOS, or WSL (Windows Subsystem for Linux)
- Python 3.8 or higher
- SSH access to target machines (for remote management)

### Installation

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install ansible

# macOS
brew install ansible

# Using pip
pip install ansible

# Verify installation
ansible --version
```

### Your First Command

```bash
# Test connection to localhost
ansible localhost -m ping

# Run a command on localhost
ansible localhost -m command -a "whoami"
```

---

## 📁 Repository Structure

```
learn-ansible/
├── README.md              # This file
├── day-1/
│   └── README.md          # Introduction & Installation
├── day-2/
│   └── README.md          # Architecture & Configuration
├── day-3/
│   └── README.md          # YAML & Playbooks
├── day-4/
│   └── README.md          # Roles
├── day-5/
│   └── README.md          # Galaxy & Lint
├── day-6/
│   └── README.md          # Collections, Vault & Variables
├── day-7/
│   └── README.md          # Jinja2 Templates & Modules
└── day-8/
    └── README.md          # Error Handling & Exercises
```

---

## 🎯 Learning Path

### Beginner (Days 1-3)
- Understand what Ansible is and why it's used
- Learn to install and configure Ansible
- Write your first playbooks with YAML

### Intermediate (Days 4-6)
- Organize code with roles
- Use community content from Galaxy
- Manage secrets and complex variables

### Advanced (Days 7-8)
- Master Jinja2 templating
- Handle errors gracefully
- Build production-ready automation

---

## 💡 Tips for Success

1. **Practice regularly** - Create a lab environment with VMs or containers
2. **Start simple** - Begin with basic playbooks before adding complexity
3. **Read the docs** - [docs.ansible.com](https://docs.ansible.com) is your friend
4. **Use version control** - Keep your playbooks in Git
5. **Test before production** - Always use `--check` and `--diff` first

---

## 🔗 Useful Resources

- [Official Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/) - Community roles and collections
- [Ansible GitHub](https://github.com/ansible/ansible)
- [Ansible Module Index](https://docs.ansible.com/ansible/latest/collections/index_module.html)

---

## 📝 License

This learning material is free to use and share.

---

Happy Automating! 🎉
