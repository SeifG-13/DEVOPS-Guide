# Ansible Vault

## What is Ansible Vault?

Ansible Vault is a feature that allows you to encrypt sensitive data such as passwords, keys, and other secrets within Ansible files.

```
┌─────────────────────────────────────────────────────────────┐
│                    ANSIBLE VAULT                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Plaintext                    Encrypted                    │
│   ┌─────────────────┐         ┌─────────────────┐          │
│   │ db_password:    │         │ $ANSIBLE_VAULT; │          │
│   │   "secret123"   │ ──────► │ 1.1;AES256     │          │
│   │ api_key:        │ encrypt │ 38626532373233 │          │
│   │   "abc123xyz"   │         │ 64613432363438 │          │
│   └─────────────────┘         │ ...            │          │
│                               └─────────────────┘          │
│                                      │                      │
│                                      │ decrypt              │
│                                      ▼                      │
│                               ┌─────────────────┐          │
│                               │ Ansible reads   │          │
│                               │ plaintext at    │          │
│                               │ runtime         │          │
│                               └─────────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Creating Encrypted Files

### Create New Encrypted File

```bash
# Create new encrypted file
ansible-vault create secrets.yml

# Editor opens, enter your secrets:
# db_password: supersecret
# api_key: abc123xyz

# Save and exit - file is now encrypted
```

### Create with Specific Editor

```bash
# Use specific editor
EDITOR=nano ansible-vault create secrets.yml

# Or set in environment
export EDITOR=vim
ansible-vault create secrets.yml
```

### View Encrypted File

```bash
# View contents without editing
ansible-vault view secrets.yml

# Enter vault password when prompted
```

---

## Encrypting Existing Files

### Encrypt a File

```bash
# Encrypt existing plaintext file
ansible-vault encrypt vars/secrets.yml

# Encrypt multiple files
ansible-vault encrypt vars/secrets.yml vars/api_keys.yml

# Encrypt with output to different file
ansible-vault encrypt plaintext.yml --output encrypted.yml
```

### Decrypt a File

```bash
# Decrypt file (permanent)
ansible-vault decrypt secrets.yml

# Decrypt to different file
ansible-vault decrypt secrets.yml --output plaintext.yml
```

---

## Editing Encrypted Files

### Edit Command

```bash
# Edit encrypted file
ansible-vault edit secrets.yml

# File is decrypted, opened in editor
# After save, file is re-encrypted
```

### Rekey (Change Password)

```bash
# Change the vault password
ansible-vault rekey secrets.yml

# Rekey multiple files
ansible-vault rekey secrets.yml vars/api_keys.yml

# Rekey with password files
ansible-vault rekey secrets.yml --old-vault-password-file=old_pass.txt --new-vault-password-file=new_pass.txt
```

---

## Password Management

### Interactive Password

```bash
# Default: prompt for password
ansible-vault create secrets.yml
# Vault password: (enter password)

ansible-playbook playbook.yml --ask-vault-pass
# Vault password: (enter password)
```

### Password File

```bash
# Create password file
echo "MyVaultPassword123" > .vault_pass
chmod 600 .vault_pass

# Use password file
ansible-vault create secrets.yml --vault-password-file .vault_pass

# Run playbook with password file
ansible-playbook playbook.yml --vault-password-file .vault_pass
```

### Environment Variable

```bash
# Set password file via environment
export ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass

# Now commands don't need --vault-password-file
ansible-vault edit secrets.yml
ansible-playbook playbook.yml
```

### In ansible.cfg

```ini
# ansible.cfg
[defaults]
vault_password_file = .vault_pass
```

### Password Script

```bash
#!/bin/bash
# vault_pass.sh - Get password from secure source

# From environment variable
echo "$VAULT_PASSWORD"

# From secret manager (example)
# aws secretsmanager get-secret-value --secret-id ansible-vault --query SecretString --output text

# From 1Password (example)
# op read "op://vault/ansible/password"
```

```bash
# Make script executable
chmod +x vault_pass.sh

# Use script
ansible-playbook playbook.yml --vault-password-file ./vault_pass.sh
```

---

## Encrypting Single Variables

### encrypt_string Command

```bash
# Encrypt a string
ansible-vault encrypt_string 'supersecret' --name 'db_password'

# Output:
# db_password: !vault |
#           $ANSIBLE_VAULT;1.1;AES256
#           61626364656667...

# Encrypt with password file
ansible-vault encrypt_string 'mypassword' --name 'api_key' --vault-password-file .vault_pass

# Encrypt from stdin (no echo)
ansible-vault encrypt_string --name 'password' --stdin-name 'secret_password'
# Type the secret, then Ctrl+D
```

### Using Encrypted Strings

```yaml
# vars/main.yml
---
# Regular variables
app_name: myapp
app_port: 8080

# Encrypted variable (inline)
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          61626364656667686970716b6c6d6e6f70717273747576777879
          7a3132333435363738393031323334353637383930313233343536

# Another encrypted variable
api_key: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          31323334353637383930313233343536373839303132333435
```

---

## Multiple Vault IDs

Use different passwords for different secrets.

### Creating with Vault IDs

```bash
# Create with vault ID
ansible-vault create --vault-id dev@prompt secrets_dev.yml
ansible-vault create --vault-id prod@prompt secrets_prod.yml

# With password files
ansible-vault create --vault-id dev@.vault_pass_dev secrets_dev.yml
ansible-vault create --vault-id prod@.vault_pass_prod secrets_prod.yml

# Encrypt string with vault ID
ansible-vault encrypt_string --vault-id prod@.vault_pass_prod 'secret' --name 'db_password'
```

### Using Multiple Vault IDs

```bash
# Run playbook with multiple vault IDs
ansible-playbook playbook.yml \
  --vault-id dev@.vault_pass_dev \
  --vault-id prod@.vault_pass_prod

# Or with prompts
ansible-playbook playbook.yml \
  --vault-id dev@prompt \
  --vault-id prod@prompt
```

### Vault ID in Files

```yaml
# When encrypted with vault ID, file contains:
$ANSIBLE_VAULT;1.2;AES256;prod
61626364656667686970716b6c6d6e6f70...
```

---

## Using Vault in Playbooks

### Encrypted Variable Files

```yaml
# playbook.yml
---
- name: Deploy Application
  hosts: webservers
  become: yes

  vars_files:
    - vars/common.yml      # Plaintext
    - vars/secrets.yml     # Encrypted

  tasks:
    - name: Configure database
      template:
        src: db.conf.j2
        dest: /etc/app/db.conf
      vars:
        password: "{{ db_password }}"  # From encrypted file
```

```bash
# Run playbook
ansible-playbook playbook.yml --ask-vault-pass
```

### Encrypted Inventory Variables

```yaml
# inventory/group_vars/production/vault.yml (encrypted)
---
vault_db_password: supersecret
vault_api_key: abc123xyz

# inventory/group_vars/production/vars.yml (plaintext)
---
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
```

### Encrypted Host Variables

```yaml
# inventory/host_vars/db1.example.com/vault.yml (encrypted)
---
vault_mysql_root_password: rootsecret

# inventory/host_vars/db1.example.com/vars.yml
---
mysql_root_password: "{{ vault_mysql_root_password }}"
```

---

## Best Practices

### Naming Convention

```
project/
├── inventory/
│   └── group_vars/
│       └── production/
│           ├── vars.yml        # Plaintext, references vault_*
│           └── vault.yml       # Encrypted, vault_* variables
├── vars/
│   ├── main.yml               # Plaintext variables
│   └── vault.yml              # Encrypted secrets
└── playbook.yml
```

```yaml
# vars/vault.yml (encrypted)
---
vault_db_password: supersecret
vault_api_key: abc123xyz
vault_ssl_private_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvgIBADANBgkqh...
  -----END PRIVATE KEY-----

# vars/main.yml (plaintext)
---
app_name: myapp
db_host: db.example.com
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
ssl_private_key: "{{ vault_ssl_private_key }}"
```

### Git Integration

```bash
# .gitignore
.vault_pass
*.vault_pass
vault_password*

# But DO commit encrypted files
# They are safe to commit
```

```bash
# Pre-commit hook to prevent plaintext secrets
#!/bin/bash
# .git/hooks/pre-commit

# Check for unencrypted vault files
for file in $(git diff --cached --name-only | grep -E 'vault\.yml$|secrets\.yml$'); do
  if ! head -1 "$file" | grep -q '^\$ANSIBLE_VAULT'; then
    echo "ERROR: $file appears to be unencrypted!"
    exit 1
  fi
done
```

### Separate Environments

```
project/
├── .vault_pass_dev
├── .vault_pass_staging
├── .vault_pass_prod
├── inventory/
│   ├── development/
│   │   └── group_vars/
│   │       └── all/
│   │           └── vault.yml  # Encrypted with dev password
│   ├── staging/
│   │   └── group_vars/
│   │       └── all/
│   │           └── vault.yml  # Encrypted with staging password
│   └── production/
│       └── group_vars/
│           └── all/
│               └── vault.yml  # Encrypted with prod password
```

### CI/CD Integration

```yaml
# GitHub Actions example
name: Deploy
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Create vault password file
        run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > .vault_pass
        shell: bash

      - name: Run Ansible
        run: |
          ansible-playbook playbook.yml \
            --vault-password-file .vault_pass \
            -i inventory/production

      - name: Cleanup
        if: always()
        run: rm -f .vault_pass
```

---

## Practical Examples

### Database Credentials

```yaml
# vars/vault.yml (encrypted)
---
vault_db_credentials:
  production:
    host: prod-db.example.com
    user: app_prod
    password: prod_secret_123
  staging:
    host: staging-db.example.com
    user: app_staging
    password: staging_secret_456

# playbook.yml
---
- name: Configure Application
  hosts: webservers
  vars_files:
    - vars/vault.yml

  tasks:
    - name: Deploy database config
      template:
        src: database.yml.j2
        dest: /etc/app/database.yml
        mode: '0600'
      vars:
        db_config: "{{ vault_db_credentials[environment] }}"
```

### SSL Certificates

```yaml
# vars/vault.yml (encrypted)
---
vault_ssl_certificate: |
  -----BEGIN CERTIFICATE-----
  MIIDXTCCAkWgAwIBAgIJAJC1...
  -----END CERTIFICATE-----

vault_ssl_private_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvgIBADANBgkqhkiG9w0B...
  -----END PRIVATE KEY-----

# tasks/ssl.yml
---
- name: Deploy SSL certificate
  copy:
    content: "{{ vault_ssl_certificate }}"
    dest: /etc/ssl/certs/app.crt
    mode: '0644'

- name: Deploy SSL private key
  copy:
    content: "{{ vault_ssl_private_key }}"
    dest: /etc/ssl/private/app.key
    mode: '0600'
```

### API Keys and Tokens

```yaml
# vars/vault.yml (encrypted)
---
vault_api_keys:
  github: ghp_xxxxxxxxxxxxxxxxxxxx
  aws_access_key: AKIAIOSFODNN7EXAMPLE
  aws_secret_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  datadog_api_key: abc123def456
  slack_webhook: https://hooks.slack.com/services/T00/B00/xxx

# playbook.yml
---
- name: Configure monitoring
  hosts: all
  vars_files:
    - vars/vault.yml

  tasks:
    - name: Configure Datadog agent
      template:
        src: datadog.yaml.j2
        dest: /etc/datadog-agent/datadog.yaml
        mode: '0640'
```

---

## Vault Commands Summary

| Command | Description |
|---------|-------------|
| `ansible-vault create` | Create new encrypted file |
| `ansible-vault encrypt` | Encrypt existing file |
| `ansible-vault decrypt` | Decrypt file |
| `ansible-vault edit` | Edit encrypted file |
| `ansible-vault view` | View encrypted file |
| `ansible-vault rekey` | Change vault password |
| `ansible-vault encrypt_string` | Encrypt single string |

### Common Options

| Option | Description |
|--------|-------------|
| `--ask-vault-pass` | Prompt for password |
| `--vault-password-file` | Use password file |
| `--vault-id` | Use specific vault ID |
| `--output` | Write to different file |

---

## Summary

### Security Best Practices

1. **Never commit** vault passwords to version control
2. **Use different passwords** for different environments
3. **Prefix vault variables** with `vault_` for clarity
4. **Store passwords** in secure locations (password managers, secret managers)
5. **Rotate passwords** regularly
6. **Use CI/CD secrets** for automation
7. **Limit access** to production vault passwords

### When to Use Vault

- Database passwords and credentials
- API keys and tokens
- SSL/TLS certificates and private keys
- SSH private keys
- Any sensitive configuration data

### When NOT to Use Vault

- Large binary files (use encrypted storage instead)
- Frequently changing data (management overhead)
- Non-sensitive configuration
