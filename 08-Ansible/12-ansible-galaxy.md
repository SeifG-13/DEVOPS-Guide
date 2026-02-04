# Ansible Galaxy

## What is Ansible Galaxy?

Ansible Galaxy is a hub for finding, sharing, and downloading Ansible content including roles and collections.

```
┌─────────────────────────────────────────────────────────────┐
│                    ANSIBLE GALAXY                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   galaxy.ansible.com                 │   │
│  │                                                      │   │
│  │   ┌──────────────┐      ┌──────────────┐           │   │
│  │   │    Roles     │      │  Collections │           │   │
│  │   │              │      │              │           │   │
│  │   │ geerlingguy  │      │ amazon.aws   │           │   │
│  │   │ ansible.     │      │ azure.       │           │   │
│  │   │ community    │      │ community.   │           │   │
│  │   └──────────────┘      └──────────────┘           │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           │ ansible-galaxy                  │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Local Installation                      │   │
│  │                                                      │   │
│  │   ~/.ansible/roles/                                  │   │
│  │   ~/.ansible/collections/                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Galaxy Website

Visit [galaxy.ansible.com](https://galaxy.ansible.com) to:
- Search for roles and collections
- View documentation and usage
- Check quality scores and downloads
- Find compatible platforms

---

## Working with Roles

### Searching Roles

```bash
# Search on command line
ansible-galaxy search nginx
ansible-galaxy search nginx --author geerlingguy
ansible-galaxy search nginx --platforms Ubuntu

# Search with filters
ansible-galaxy search docker --galaxy-tags container
```

### Installing Roles

```bash
# Install from Galaxy
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install geerlingguy.docker

# Install specific version
ansible-galaxy install geerlingguy.nginx,3.1.0

# Install to specific path
ansible-galaxy install geerlingguy.nginx -p ./roles

# Install with different name
ansible-galaxy install geerlingguy.nginx,3.1.0,nginx-role

# Install from Git
ansible-galaxy install git+https://github.com/user/repo.git
ansible-galaxy install git+https://github.com/user/repo.git,v1.0.0

# Install from URL
ansible-galaxy install https://github.com/user/repo/archive/main.tar.gz
```

### Role Info

```bash
# Get information about a role
ansible-galaxy info geerlingguy.nginx

# List installed roles
ansible-galaxy list

# Remove a role
ansible-galaxy remove geerlingguy.nginx
```

### Requirements File

```yaml
# requirements.yml
---
roles:
  # From Galaxy
  - name: geerlingguy.nginx
    version: "3.1.0"

  # From Galaxy with custom name
  - name: geerlingguy.docker
    version: "6.0.0"
    src: geerlingguy.docker

  # From GitHub
  - name: nginx-custom
    src: https://github.com/company/ansible-nginx
    version: v2.0.0

  # From Git with branch
  - name: internal-role
    src: git@github.com:company/internal-role.git
    version: develop
    scm: git

  # From URL (tarball)
  - name: custom-role
    src: https://example.com/roles/custom-role.tar.gz
```

```bash
# Install from requirements
ansible-galaxy install -r requirements.yml

# Install roles only (skip collections)
ansible-galaxy role install -r requirements.yml

# Force reinstall
ansible-galaxy install -r requirements.yml --force
```

---

## Working with Collections

Collections are a newer distribution format that can include:
- Roles
- Modules
- Plugins
- Playbooks
- Documentation

### Installing Collections

```bash
# Install from Galaxy
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install azure.azcollection

# Install specific version
ansible-galaxy collection install amazon.aws:5.0.0

# Install to specific path
ansible-galaxy collection install amazon.aws -p ./collections

# Install from tarball
ansible-galaxy collection install /path/to/collection.tar.gz

# Install from Git
ansible-galaxy collection install git+https://github.com/ansible-collections/amazon.aws.git
```

### Collection Requirements

```yaml
# requirements.yml
---
collections:
  # From Galaxy
  - name: amazon.aws
    version: ">=5.0.0,<6.0.0"

  - name: azure.azcollection
    version: "1.15.0"

  - name: community.general
    # Latest version

  - name: community.docker
    version: "3.4.0"

  # From Git
  - name: company.internal
    source: git@github.com:company/ansible-collection.git
    type: git
    version: main

  # From URL
  - name: custom.collection
    source: https://example.com/collections/custom-1.0.0.tar.gz
    type: url
```

```bash
# Install collections from requirements
ansible-galaxy collection install -r requirements.yml

# Install everything (roles + collections)
ansible-galaxy install -r requirements.yml
```

### Using Collections

```yaml
# Method 1: Fully Qualified Collection Name (FQCN)
---
- name: Use AWS modules
  hosts: localhost
  tasks:
    - name: Create EC2 instance
      amazon.aws.ec2_instance:
        name: web-server
        instance_type: t3.micro
        image_id: ami-12345678

# Method 2: collections keyword
---
- name: Use AWS modules
  hosts: localhost
  collections:
    - amazon.aws

  tasks:
    - name: Create EC2 instance
      ec2_instance:  # Short name works
        name: web-server
        instance_type: t3.micro

# Method 3: In ansible.cfg
# [defaults]
# collections_paths = ./collections:/usr/share/ansible/collections
```

### Listing Collections

```bash
# List installed collections
ansible-galaxy collection list

# Show collection details
ansible-doc -l amazon.aws
ansible-doc amazon.aws.ec2_instance
```

---

## Creating Roles for Galaxy

### Initialize Role

```bash
# Create role structure
ansible-galaxy init my_role

# With custom path
ansible-galaxy init --init-path=./roles my_role
```

### Role Metadata

```yaml
# meta/main.yml
---
galaxy_info:
  author: Your Name
  description: A brief description of the role
  company: Your Company (optional)

  license: MIT

  min_ansible_version: "2.10"

  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: Debian
      versions:
        - bullseye
        - bookworm
    - name: EL  # Enterprise Linux (RHEL, CentOS)
      versions:
        - "8"
        - "9"

  galaxy_tags:
    - web
    - nginx
    - proxy

  # Optional: Link to issue tracker
  issue_tracker_url: https://github.com/user/repo/issues

  # Optional: Link to documentation
  documentation: https://github.com/user/repo/wiki

dependencies: []
```

### README.md Template

```markdown
# Role Name

Brief description of the role.

## Requirements

List any prerequisites or dependencies.

## Role Variables

Available variables with defaults:

```yaml
variable_name: default_value  # Description
```

## Dependencies

List of dependent roles.

## Example Playbook

```yaml
- hosts: servers
  roles:
    - role: username.rolename
      vars:
        variable_name: value
```

## License

MIT

## Author Information

Your Name (your@email.com)
```

### Publishing to Galaxy

```bash
# Login to Galaxy (get token from galaxy.ansible.com)
ansible-galaxy login

# Or use token directly
export ANSIBLE_GALAXY_TOKEN=your_token

# Import from GitHub (role must be in a public repo)
ansible-galaxy import username repo_name

# The repo name should match: ansible-role-<name>
# Example: ansible-role-nginx -> installs as username.nginx
```

---

## Creating Collections

### Initialize Collection

```bash
# Create collection structure
ansible-galaxy collection init company.mycollection

# Result:
# company/mycollection/
# ├── docs/
# ├── galaxy.yml
# ├── plugins/
# │   └── README.md
# ├── README.md
# └── roles/
```

### Collection Structure

```
company/mycollection/
├── galaxy.yml              # Collection metadata
├── README.md               # Documentation
├── docs/                   # Additional documentation
├── plugins/                # Plugins
│   ├── modules/            # Custom modules
│   ├── inventory/          # Inventory plugins
│   ├── lookup/             # Lookup plugins
│   ├── filter/             # Filter plugins
│   └── callback/           # Callback plugins
├── roles/                  # Roles
│   └── myrole/
├── playbooks/              # Playbooks
└── tests/                  # Tests
```

### galaxy.yml

```yaml
# galaxy.yml
---
namespace: company
name: mycollection
version: 1.0.0
readme: README.md
authors:
  - Your Name <your@email.com>
description: Description of the collection
license:
  - MIT
license_file: LICENSE
tags:
  - cloud
  - automation
dependencies:
  amazon.aws: ">=5.0.0"
  community.general: "*"
repository: https://github.com/company/ansible-collection
documentation: https://github.com/company/ansible-collection/wiki
homepage: https://company.com
issues: https://github.com/company/ansible-collection/issues
build_ignore:
  - .gitignore
  - "*.tar.gz"
```

### Building Collection

```bash
# Build the collection tarball
ansible-galaxy collection build

# Output: company-mycollection-1.0.0.tar.gz

# Build from specific path
ansible-galaxy collection build /path/to/collection
```

### Publishing Collection

```bash
# Publish to Galaxy
ansible-galaxy collection publish company-mycollection-1.0.0.tar.gz

# Or with token
ansible-galaxy collection publish ./company-mycollection-1.0.0.tar.gz --token $GALAXY_TOKEN

# Publish to Automation Hub (Red Hat)
ansible-galaxy collection publish ./company-mycollection-1.0.0.tar.gz --server https://cloud.redhat.com/api/automation-hub/
```

---

## Private Galaxy / Automation Hub

### Using Private Galaxy Server

```yaml
# ansible.cfg
[galaxy]
server_list = private_galaxy, release_galaxy

[galaxy_server.private_galaxy]
url = https://galaxy.company.com/api/
token = your_private_token

[galaxy_server.release_galaxy]
url = https://galaxy.ansible.com/
```

```bash
# Install from private server
ansible-galaxy collection install company.mycollection --server private_galaxy
```

### AWX/Tower as Private Galaxy

```yaml
# ansible.cfg
[galaxy]
server_list = automation_hub

[galaxy_server.automation_hub]
url = https://awx.company.com/api/v2/
token = awx_token
```

---

## Managing Dependencies

### Complete Requirements File

```yaml
# requirements.yml
---
# Roles
roles:
  - name: geerlingguy.nginx
    version: "3.1.0"
  - name: geerlingguy.php
    version: "5.0.0"
  - name: geerlingguy.mysql
    version: "4.0.0"

# Collections
collections:
  - name: amazon.aws
    version: ">=5.0.0"
  - name: community.general
  - name: community.docker
    version: "3.4.0"
  - name: ansible.posix
```

### Version Constraints

```yaml
collections:
  # Exact version
  - name: amazon.aws
    version: "5.0.0"

  # Minimum version
  - name: amazon.aws
    version: ">=5.0.0"

  # Version range
  - name: amazon.aws
    version: ">=5.0.0,<6.0.0"

  # Latest
  - name: amazon.aws
    # No version = latest

  # Semantic versioning
  - name: amazon.aws
    version: "~=5.0"  # >=5.0.0, <6.0.0
```

### Installation Best Practices

```bash
# Create virtual environment for isolation
python -m venv ansible-venv
source ansible-venv/bin/activate
pip install ansible

# Install requirements
ansible-galaxy install -r requirements.yml

# Force reinstall (update to match requirements)
ansible-galaxy install -r requirements.yml --force

# Verify installation
ansible-galaxy list
ansible-galaxy collection list
```

---

## Popular Collections

| Collection | Description |
|------------|-------------|
| `amazon.aws` | AWS modules and plugins |
| `azure.azcollection` | Azure modules |
| `google.cloud` | Google Cloud modules |
| `community.general` | General community modules |
| `community.docker` | Docker modules |
| `community.kubernetes` | Kubernetes modules |
| `ansible.posix` | POSIX modules |
| `ansible.netcommon` | Network common modules |
| `ansible.windows` | Windows modules |

---

## Summary

| Command | Description |
|---------|-------------|
| `ansible-galaxy search` | Search for roles |
| `ansible-galaxy install` | Install roles/collections |
| `ansible-galaxy list` | List installed content |
| `ansible-galaxy remove` | Remove a role |
| `ansible-galaxy init` | Create role skeleton |
| `ansible-galaxy collection build` | Build collection tarball |
| `ansible-galaxy collection publish` | Publish to Galaxy |
| `ansible-galaxy install -r` | Install from requirements |

### Best Practices

1. Pin versions in requirements.yml for reproducibility
2. Use collections over standalone roles when possible
3. Test roles/collections before production use
4. Keep requirements.yml in version control
5. Use private Galaxy for internal content
6. Document role/collection usage in README
