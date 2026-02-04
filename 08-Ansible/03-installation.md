# Ansible Installation

## Prerequisites

Before installing Ansible, ensure your control node meets these requirements:

| Requirement | Details |
|-------------|---------|
| **Operating System** | Linux, macOS, or Windows (WSL) |
| **Python** | Python 3.8 or higher |
| **SSH Client** | For connecting to managed nodes |
| **Network Access** | To managed nodes |

### Check Python Version

```bash
# Check Python version
python3 --version

# If not installed (Ubuntu/Debian)
sudo apt update && sudo apt install python3 python3-pip -y

# If not installed (RHEL/CentOS)
sudo dnf install python3 python3-pip -y
```

---

## Installation Methods

### Method 1: pip (Recommended)

The most reliable way to get the latest version.

```bash
# Install using pip
pip3 install ansible

# Or install in user space (no sudo required)
pip3 install --user ansible

# Install specific version
pip3 install ansible==2.15.0

# Install with additional dependencies
pip3 install ansible[azure]  # Azure modules
pip3 install ansible[aws]    # AWS modules
```

### Method 2: apt (Ubuntu/Debian)

```bash
# Add Ansible PPA for latest version
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository --yes --update ppa:ansible/ansible

# Install Ansible
sudo apt install ansible -y
```

### Method 3: dnf/yum (RHEL/CentOS/Fedora)

```bash
# RHEL 8+ / CentOS Stream / Fedora
sudo dnf install ansible -y

# RHEL 7 / CentOS 7
sudo yum install epel-release -y
sudo yum install ansible -y
```

### Method 4: brew (macOS)

```bash
# Install using Homebrew
brew install ansible
```

### Method 5: Windows (WSL)

Ansible doesn't run natively on Windows. Use WSL (Windows Subsystem for Linux).

```powershell
# In PowerShell (Admin)
wsl --install

# After WSL is set up, open Ubuntu terminal
```

```bash
# In WSL Ubuntu
sudo apt update
sudo apt install ansible -y
```

---

## Installation Verification

### Check Installation

```bash
# Check Ansible version
ansible --version

# Expected output:
# ansible [core 2.15.0]
#   config file = /etc/ansible/ansible.cfg
#   configured module search path = ['/home/user/.ansible/plugins/modules']
#   ansible python module location = /usr/lib/python3/dist-packages/ansible
#   ansible collection location = /home/user/.ansible/collections
#   executable location = /usr/bin/ansible
#   python version = 3.10.12
```

### Available Commands

```bash
# List all ansible commands
ansible           # Run ad-hoc commands
ansible-playbook  # Run playbooks
ansible-galaxy    # Manage roles and collections
ansible-vault     # Encrypt/decrypt files
ansible-doc       # View module documentation
ansible-config    # View/dump configuration
ansible-inventory # View inventory
ansible-pull      # Pull playbooks from VCS
```

---

## Post-Installation Setup

### 1. Create SSH Key Pair

```bash
# Generate SSH key pair (if not exists)
ssh-keygen -t ed25519 -C "ansible"

# Or RSA (for older systems)
ssh-keygen -t rsa -b 4096 -C "ansible"
```

### 2. Copy SSH Key to Managed Nodes

```bash
# Copy SSH key to managed nodes
ssh-copy-id user@managed-node-ip

# Or manually
cat ~/.ssh/id_ed25519.pub | ssh user@managed-node "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 3. Test SSH Connection

```bash
# Test SSH without password
ssh user@managed-node-ip

# Should connect without password prompt
```

### 4. Create Inventory File

```bash
# Create project directory
mkdir -p ~/ansible-project
cd ~/ansible-project

# Create inventory file
cat > inventory << 'EOF'
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

[databases]
db1 ansible_host=192.168.1.20

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
EOF
```

### 5. Test Connectivity

```bash
# Test connection to all hosts
ansible all -i inventory -m ping

# Expected output:
# web1 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

---

## Configuration File (ansible.cfg)

### Create Configuration File

```bash
# Create ansible.cfg in project directory
cat > ansible.cfg << 'EOF'
[defaults]
inventory = ./inventory
remote_user = ubuntu
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
deprecation_warnings = False
interpreter_python = auto_silent
forks = 10
timeout = 30
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
EOF
```

### Configuration Priority

```
┌─────────────────────────────────────────┐
│     ANSIBLE.CFG SEARCH ORDER            │
│         (First found wins)              │
├─────────────────────────────────────────┤
│                                         │
│  1. ANSIBLE_CONFIG (env variable)       │
│           │                             │
│           ▼                             │
│  2. ./ansible.cfg (current directory)   │
│           │                             │
│           ▼                             │
│  3. ~/.ansible.cfg (home directory)     │
│           │                             │
│           ▼                             │
│  4. /etc/ansible/ansible.cfg            │
│                                         │
└─────────────────────────────────────────┘
```

### Check Active Configuration

```bash
# Show which config file is being used
ansible --version

# Dump current configuration
ansible-config dump

# Show only changed settings
ansible-config dump --only-changed

# View specific setting
ansible-config dump | grep -i forks
```

---

## Setting Up Managed Nodes

### Linux Managed Nodes

```bash
# On managed node - Install Python (if needed)
sudo apt update && sudo apt install python3 -y  # Debian/Ubuntu
sudo dnf install python3 -y                      # RHEL/Fedora

# Ensure SSH is running
sudo systemctl status sshd
sudo systemctl enable --now sshd

# Create ansible user (optional but recommended)
sudo useradd -m -s /bin/bash ansible
sudo passwd ansible

# Add to sudoers (passwordless sudo)
echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible
```

### Windows Managed Nodes

```powershell
# On Windows managed node - Run PowerShell as Admin

# Enable WinRM
winrm quickconfig -q

# Set up WinRM for Ansible
# Download and run the ConfigureRemotingForAnsible.ps1 script
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"
(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
powershell.exe -ExecutionPolicy ByPass -File $file

# Verify WinRM listener
winrm enumerate winrm/config/Listener
```

### Windows Inventory Entry

```ini
[windows]
win1 ansible_host=192.168.1.30

[windows:vars]
ansible_user=Administrator
ansible_password=SecurePassword123
ansible_connection=winrm
ansible_winrm_server_cert_validation=ignore
ansible_winrm_transport=ntlm
```

---

## Virtual Environment Setup

### Using Python venv

```bash
# Create virtual environment
python3 -m venv ~/ansible-venv

# Activate virtual environment
source ~/ansible-venv/bin/activate

# Install Ansible in venv
pip install ansible ansible-lint

# Verify installation
ansible --version

# Deactivate when done
deactivate
```

### Using pipx (Isolated Installation)

```bash
# Install pipx
pip install --user pipx
pipx ensurepath

# Install Ansible with pipx
pipx install ansible-core
pipx inject ansible-core ansible-lint

# Ansible is now isolated from system Python
ansible --version
```

---

## Installing Collections

Ansible Collections are the modern way to distribute content.

```bash
# Install a collection from Ansible Galaxy
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install azure.azcollection
ansible-galaxy collection install community.general

# Install from requirements file
cat > requirements.yml << 'EOF'
collections:
  - name: amazon.aws
    version: ">=5.0.0"
  - name: community.general
  - name: ansible.posix
EOF

ansible-galaxy collection install -r requirements.yml

# List installed collections
ansible-galaxy collection list
```

---

## Troubleshooting Installation

### Common Issues

| Issue | Solution |
|-------|----------|
| `ansible: command not found` | Add pip bin to PATH: `export PATH=$PATH:~/.local/bin` |
| `Permission denied` | Use `pip install --user` or virtual environment |
| `Python version error` | Install Python 3.8+ and use `pip3` |
| `SSH connection refused` | Ensure SSH server running on managed node |
| `Host key verification failed` | Set `host_key_checking = False` in ansible.cfg |

### Debug Commands

```bash
# Check Python path
which python3
python3 -m site --user-base

# Check pip packages
pip3 list | grep ansible

# Test SSH verbose
ssh -vvv user@managed-node

# Test Ansible verbose
ansible all -m ping -vvvv
```

---

## Quick Start Test

```bash
# Create a test project
mkdir ~/ansible-quickstart && cd ~/ansible-quickstart

# Create inventory (using localhost)
echo "localhost ansible_connection=local" > inventory

# Create simple playbook
cat > test.yml << 'EOF'
---
- name: Test Playbook
  hosts: localhost
  gather_facts: yes
  tasks:
    - name: Display OS info
      debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Create test file
      file:
        path: /tmp/ansible-test.txt
        state: touch
        mode: '0644'

    - name: Verify file created
      stat:
        path: /tmp/ansible-test.txt
      register: result

    - name: Show result
      debug:
        var: result.stat.exists
EOF

# Run the playbook
ansible-playbook -i inventory test.yml
```

---

## Summary

| Step | Command |
|------|---------|
| **Install (pip)** | `pip3 install ansible` |
| **Install (apt)** | `sudo apt install ansible` |
| **Verify** | `ansible --version` |
| **Generate SSH key** | `ssh-keygen -t ed25519` |
| **Copy SSH key** | `ssh-copy-id user@host` |
| **Test connection** | `ansible all -m ping` |
| **Install collection** | `ansible-galaxy collection install name` |

Ansible is now installed and ready for use. The next step is to create an inventory and start writing playbooks.
