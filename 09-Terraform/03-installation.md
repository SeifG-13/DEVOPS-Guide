# Terraform Installation

## Download Options

Terraform is distributed as a single binary. Choose your installation method:

| Method | Platforms | Best For |
|--------|-----------|----------|
| Package Manager | Linux, macOS | Easiest installation |
| Manual Download | All | Specific versions |
| tfenv | All | Version management |
| Docker | All | Isolated environments |

---

## Linux Installation

### Ubuntu/Debian (APT)

```bash
# Install dependencies
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

# Verify key
gpg --no-default-keyring \
  --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
  --fingerprint

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install
sudo apt-get update
sudo apt-get install terraform
```

### RHEL/CentOS/Fedora (YUM/DNF)

```bash
# Add HashiCorp repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install Terraform
sudo yum -y install terraform

# Or with DNF (Fedora)
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
sudo dnf -y install terraform
```

### Manual Binary Installation (Any Linux)

```bash
# Download latest version (check https://www.terraform.io/downloads)
TERRAFORM_VERSION="1.7.0"
wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip

# Unzip
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip

# Move to PATH
sudo mv terraform /usr/local/bin/

# Verify
terraform version

# Cleanup
rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip
```

---

## Windows Installation

### Chocolatey (Recommended)

```powershell
# Install Chocolatey (if not installed)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install Terraform
choco install terraform

# Upgrade Terraform
choco upgrade terraform
```

### Winget

```powershell
# Install using Windows Package Manager
winget install Hashicorp.Terraform
```

### Manual Installation

1. Download from [terraform.io/downloads](https://www.terraform.io/downloads)
2. Extract the ZIP file
3. Move `terraform.exe` to a directory in your PATH
4. Or add the directory to your PATH:

```powershell
# Add to PATH (PowerShell - User level)
$env:Path += ";C:\terraform"
[Environment]::SetEnvironmentVariable("Path", $env:Path, [EnvironmentVariableTarget]::User)
```

---

## macOS Installation

### Homebrew (Recommended)

```bash
# Install Homebrew (if not installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Terraform
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Upgrade Terraform
brew upgrade hashicorp/tap/terraform
```

### Manual Installation

```bash
# Download (Intel Mac)
curl -LO https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_darwin_amd64.zip

# Download (Apple Silicon)
curl -LO https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_darwin_arm64.zip

# Unzip and install
unzip terraform_*.zip
sudo mv terraform /usr/local/bin/
rm terraform_*.zip
```

---

## Version Management

### tfenv (Recommended)

tfenv allows managing multiple Terraform versions.

```bash
# Install tfenv (Linux/macOS)
git clone https://github.com/tfutils/tfenv.git ~/.tfenv

# Add to PATH (bash)
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Add to PATH (zsh)
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# List available versions
tfenv list-remote

# Install specific version
tfenv install 1.7.0
tfenv install 1.6.0
tfenv install latest

# Use specific version
tfenv use 1.7.0

# Set default version
tfenv use 1.7.0
echo "1.7.0" > ~/.tfenv/version

# List installed versions
tfenv list

# Uninstall version
tfenv uninstall 1.6.0
```

### .terraform-version File

```bash
# Project-specific version
echo "1.7.0" > .terraform-version

# tfenv automatically uses this version in the directory
cd my-project
terraform version  # Uses 1.7.0
```

### tfswitch (Alternative)

```bash
# Install tfswitch
brew install warrensbox/tap/tfswitch

# Or download binary
curl -L https://raw.githubusercontent.com/warrensbox/terraform-switcher/release/install.sh | bash

# Interactive version selection
tfswitch

# Install specific version
tfswitch 1.7.0
```

---

## Docker Installation

### Using Official Image

```bash
# Pull official image
docker pull hashicorp/terraform:latest

# Run Terraform
docker run --rm -it \
  -v $(pwd):/workspace \
  -w /workspace \
  hashicorp/terraform:latest \
  init

# Create alias for convenience
alias terraform='docker run --rm -it -v $(pwd):/workspace -w /workspace hashicorp/terraform:latest'

# Now use normally
terraform version
terraform plan
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3'
services:
  terraform:
    image: hashicorp/terraform:1.7.0
    volumes:
      - .:/workspace
      - ~/.aws:/root/.aws:ro  # AWS credentials
    working_dir: /workspace
    entrypoint: ["terraform"]
```

```bash
# Run commands
docker-compose run --rm terraform init
docker-compose run --rm terraform plan
```

---

## Verification

### Check Installation

```bash
# Check version
terraform version

# Expected output:
# Terraform v1.7.0
# on linux_amd64

# Check available commands
terraform -help

# Detailed help for command
terraform plan -help
```

### Test Configuration

```bash
# Create test directory
mkdir terraform-test && cd terraform-test

# Create simple configuration
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.0.0"
}

output "hello" {
  value = "Terraform is working!"
}
EOF

# Initialize
terraform init

# Apply
terraform apply -auto-approve

# Should output: hello = "Terraform is working!"

# Cleanup
cd .. && rm -rf terraform-test
```

---

## IDE Setup

### VS Code

```bash
# Install HashiCorp Terraform extension
# Extension ID: hashicorp.terraform

# Features:
# - Syntax highlighting
# - IntelliSense
# - Code formatting
# - Validation
```

### VS Code Settings

```json
// settings.json
{
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true,
    "editor.formatOnSaveMode": "file"
  },
  "[terraform-vars]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true,
    "editor.formatOnSaveMode": "file"
  },
  "terraform.languageServer.enable": true,
  "terraform.validation.enable": true
}
```

### JetBrains IDEs

- Install "HashiCorp Terraform / HCL language support" plugin
- Settings → Plugins → Search "Terraform"

### Vim/Neovim

```bash
# Install vim-terraform plugin
# Using vim-plug:
# Plug 'hashivim/vim-terraform'

# Add to .vimrc
let g:terraform_fmt_on_save=1
let g:terraform_align=1
```

---

## Shell Completion

### Bash

```bash
# Generate completion script
terraform -install-autocomplete

# Or manually
echo 'complete -C /usr/local/bin/terraform terraform' >> ~/.bashrc
source ~/.bashrc
```

### Zsh

```bash
# Generate completion script
terraform -install-autocomplete

# Or manually add to .zshrc
autoload -U +X bashcompinit && bashcompinit
complete -o nospace -C /usr/local/bin/terraform terraform
```

### Fish

```bash
# Install completion
terraform -install-autocomplete
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| `command not found` | Add terraform to PATH |
| Permission denied | Use `sudo` or fix permissions |
| Wrong version | Check PATH order, use tfenv |
| SSL certificate error | Update CA certificates |

### PATH Issues

```bash
# Check if terraform is in PATH
which terraform
echo $PATH

# Common paths
# Linux: /usr/local/bin, /usr/bin
# macOS: /usr/local/bin, /opt/homebrew/bin
# Windows: C:\terraform, C:\HashiCorp\Terraform
```

### Verify Binary

```bash
# Check binary details
file $(which terraform)
# Should show: ELF 64-bit executable (Linux)
# or: Mach-O 64-bit executable (macOS)

# Check SHA256 (verify download integrity)
sha256sum terraform_1.7.0_linux_amd64.zip
# Compare with published checksums
```

---

## Environment Setup

### AWS Credentials

```bash
# Option 1: Environment variables
export AWS_ACCESS_KEY_ID="your_access_key"
export AWS_SECRET_ACCESS_KEY="your_secret_key"
export AWS_REGION="us-east-1"

# Option 2: AWS CLI configuration
aws configure

# Option 3: ~/.aws/credentials file
cat ~/.aws/credentials
# [default]
# aws_access_key_id = your_access_key
# aws_secret_access_key = your_secret_key
```

### Azure Credentials

```bash
# Login with Azure CLI
az login

# Or use Service Principal
export ARM_CLIENT_ID="your_client_id"
export ARM_CLIENT_SECRET="your_client_secret"
export ARM_SUBSCRIPTION_ID="your_subscription_id"
export ARM_TENANT_ID="your_tenant_id"
```

### GCP Credentials

```bash
# Login with gcloud
gcloud auth application-default login

# Or use Service Account
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
```

---

## Summary

| Platform | Recommended Method |
|----------|-------------------|
| **Ubuntu/Debian** | APT repository |
| **RHEL/CentOS** | YUM repository |
| **macOS** | Homebrew |
| **Windows** | Chocolatey |
| **Any** | tfenv (version management) |
| **CI/CD** | Docker image |

### Post-Installation Checklist

- [ ] Verify installation: `terraform version`
- [ ] Install IDE extension
- [ ] Enable shell completion
- [ ] Set up cloud credentials
- [ ] Install tfenv for version management
- [ ] Test with simple configuration
