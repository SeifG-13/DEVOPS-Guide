# Azure Account Setup

## Creating an Azure Account

### Free Account Options

```
┌─────────────────────────────────────────────────────────────┐
│                AZURE ACCOUNT OPTIONS                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Free Account                 Pay-As-You-Go                 │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │ • $200 credit   │         │ • No upfront    │           │
│  │ • 30 days       │         │ • Pay for usage │           │
│  │ • Free services │         │ • All services  │           │
│  │ • 12 months free│         │ • Enterprise    │           │
│  │   for some      │         │   features      │           │
│  └─────────────────┘         └─────────────────┘           │
│                                                             │
│  Student Account              Enterprise Agreement          │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │ • $100 credit   │         │ • Volume        │           │
│  │ • No credit card│         │   discounts     │           │
│  │ • Free services │         │ • Committed     │           │
│  │ • Renews yearly │         │   spend         │           │
│  └─────────────────┘         └─────────────────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Sign Up Process

```bash
# 1. Go to https://azure.microsoft.com/free
# 2. Click "Start free"
# 3. Sign in with Microsoft account (or create one)
# 4. Verify phone number
# 5. Enter credit card (for verification, not charged)
# 6. Accept terms and sign up
```

---

## Azure Portal Overview

### Portal Navigation

```
┌─────────────────────────────────────────────────────────────┐
│  Azure Portal (portal.azure.com)                            │
├─────────────────────────────────────────────────────────────┤
│ ┌────────┐ ┌──────────────────────────────────────────────┐│
│ │ ☰ Menu │ │ 🔍 Search resources, services, docs          ││
│ └────────┘ └──────────────────────────────────────────────┘│
│ ┌─────────────────────────────────────────────────────────┐│
│ │                                                         ││
│ │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   ││
│ │  │ Create  │  │Resource │  │ All     │  │ Recent  │   ││
│ │  │Resource │  │ Groups  │  │Services │  │Resources│   ││
│ │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   ││
│ │                                                         ││
│ │  Dashboard                                              ││
│ │  ┌─────────────────┐  ┌─────────────────┐             ││
│ │  │ Cost Summary    │  │ Service Health  │             ││
│ │  │ $XX.XX / month  │  │ All systems OK  │             ││
│ │  └─────────────────┘  └─────────────────┘             ││
│ │                                                         ││
│ │  ┌─────────────────┐  ┌─────────────────┐             ││
│ │  │ Recent Resources│  │ Favorite        │             ││
│ │  │ - VM-1          │  │ Services        │             ││
│ │  │ - webapp-prod   │  │ - App Services  │             ││
│ │  │ - sql-db-1      │  │ - SQL Databases │             ││
│ │  └─────────────────┘  └─────────────────┘             ││
│ │                                                         ││
│ └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Key Portal Features

| Feature | Description | Shortcut |
|---------|-------------|----------|
| **Search** | Find any resource or service | `G + /` |
| **Cloud Shell** | Built-in CLI | `>_` icon |
| **Notifications** | Deployment status | Bell icon |
| **Directory** | Switch tenants | Profile menu |
| **Cost Management** | Monitor spending | Search "Cost" |

---

## Subscriptions

### Understanding Subscriptions

```
┌─────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTIONS                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Subscription = Billing Boundary + Access Boundary          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Subscription: Production                            │   │
│  │  ├── Billing: Credit Card / Invoice                 │   │
│  │  ├── Quota: Default service limits                  │   │
│  │  ├── Access: Who can manage resources               │   │
│  │  └── Resources:                                      │   │
│  │      ├── Resource Group: rg-webapp                   │   │
│  │      ├── Resource Group: rg-database                 │   │
│  │      └── Resource Group: rg-monitoring               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Common Patterns:                                           │
│  • One subscription per environment (dev/staging/prod)     │
│  • One subscription per team/department                     │
│  • One subscription per project                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create a Subscription

```bash
# Via Portal:
# 1. Search "Subscriptions"
# 2. Click "+ Add"
# 3. Select offer type
# 4. Enter details

# Via CLI (requires permissions):
az account list --output table
```

### Subscription Types

| Type | Best For |
|------|----------|
| **Free** | Learning, testing |
| **Pay-As-You-Go** | Small projects, startups |
| **Dev/Test** | Development (discounted rates) |
| **Enterprise Agreement** | Large organizations |
| **CSP** | Through Microsoft partners |

---

## Management Groups

### Hierarchy for Enterprise

```
┌─────────────────────────────────────────────────────────────┐
│                  MANAGEMENT GROUPS                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Root Management Group (Tenant)                             │
│  │                                                          │
│  ├── MG: Contoso                                           │
│  │   │                                                      │
│  │   ├── MG: Production                                    │
│  │   │   ├── Sub: Prod-East                                │
│  │   │   └── Sub: Prod-West                                │
│  │   │                                                      │
│  │   ├── MG: Development                                   │
│  │   │   ├── Sub: Dev                                      │
│  │   │   └── Sub: QA                                       │
│  │   │                                                      │
│  │   └── MG: Sandbox                                       │
│  │       └── Sub: Sandbox                                  │
│  │                                                          │
│  └── Policies & RBAC applied at any level                  │
│      (Inherited by children)                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create Management Group

```bash
# Create management group
az account management-group create \
  --name "Production" \
  --display-name "Production Workloads"

# Add subscription to management group
az account management-group subscription add \
  --name "Production" \
  --subscription "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# List management groups
az account management-group list --output table
```

---

## Resource Groups

### What are Resource Groups?

```
┌─────────────────────────────────────────────────────────────┐
│                   RESOURCE GROUPS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Resource Group = Logical Container for Resources           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Resource Group: rg-myapp-prod-eastus               │   │
│  │  ├── Location: East US (metadata only)              │   │
│  │  ├── Tags: env=prod, app=myapp                      │   │
│  │  │                                                   │   │
│  │  └── Resources:                                      │   │
│  │      ├── App Service: app-myapp-prod                │   │
│  │      ├── Azure SQL: sql-myapp-prod                  │   │
│  │      ├── Key Vault: kv-myapp-prod                   │   │
│  │      ├── Storage: stmyappprod                       │   │
│  │      └── App Insights: ai-myapp-prod                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Note: Resources can be in different regions than RG       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Resource Group Best Practices

```bash
# Naming convention: rg-<app>-<env>-<region>
# Examples:
# - rg-webapp-prod-eastus
# - rg-api-dev-westus
# - rg-shared-prod-eastus

# Create resource group
az group create \
  --name rg-myapp-prod-eastus \
  --location eastus \
  --tags env=prod app=myapp team=backend

# List resource groups
az group list --output table

# Delete resource group (and ALL resources inside)
az group delete --name rg-myapp-dev-eastus --yes --no-wait
```

### Organizing Resources

| Pattern | Example | Use Case |
|---------|---------|----------|
| **By Application** | rg-webapp-prod | All app resources together |
| **By Environment** | rg-prod, rg-dev | Separate environments |
| **By Lifecycle** | rg-temp-testing | Easy cleanup |
| **By Type** | rg-networking | Shared resources |

---

## Installing Azure CLI

### Windows Installation

```powershell
# Option 1: MSI Installer
# Download from: https://aka.ms/installazurecliwindows

# Option 2: Winget
winget install -e --id Microsoft.AzureCLI

# Option 3: Chocolatey
choco install azure-cli

# Verify installation
az --version
```

### macOS Installation

```bash
# Using Homebrew
brew update && brew install azure-cli

# Verify
az --version
```

### Linux Installation

```bash
# Ubuntu/Debian
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# RHEL/CentOS
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo dnf install azure-cli

# Verify
az --version
```

### Docker

```bash
# Run Azure CLI in Docker
docker run -it mcr.microsoft.com/azure-cli

# With volume for persistence
docker run -it -v ~/.azure:/root/.azure mcr.microsoft.com/azure-cli
```

---

## Azure CLI Authentication

### Login Methods

```bash
# Interactive login (opens browser)
az login

# Device code (for headless/SSH)
az login --use-device-code

# Service principal (for CI/CD)
az login --service-principal \
  --username <app-id> \
  --password <password-or-cert> \
  --tenant <tenant-id>

# Managed identity (from Azure resource)
az login --identity
```

### Account Management

```bash
# List subscriptions
az account list --output table

# Show current subscription
az account show

# Set default subscription
az account set --subscription "My Subscription"
# or by ID
az account set --subscription "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# List available regions
az account list-locations --output table
```

---

## Azure PowerShell

### Installation

```powershell
# Install Azure PowerShell module
Install-Module -Name Az -Repository PSGallery -Force

# Update to latest version
Update-Module -Name Az

# Verify installation
Get-InstalledModule -Name Az
```

### Authentication

```powershell
# Interactive login
Connect-AzAccount

# With device code
Connect-AzAccount -UseDeviceAuthentication

# Service principal
$credential = Get-Credential
Connect-AzAccount -ServicePrincipal -Credential $credential -Tenant <tenant-id>
```

### Basic Commands

```powershell
# List subscriptions
Get-AzSubscription

# Set context
Set-AzContext -SubscriptionId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# List resource groups
Get-AzResourceGroup | Format-Table

# Create resource group
New-AzResourceGroup -Name "rg-test" -Location "eastus"
```

---

## Azure Cloud Shell

### Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   CLOUD SHELL                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Browser-based shell with pre-installed tools               │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  $ az --version                                      │   │
│  │  azure-cli    2.55.0                                │   │
│  │                                                      │   │
│  │  $ terraform --version                               │   │
│  │  Terraform v1.7.0                                   │   │
│  │                                                      │   │
│  │  $ kubectl version --client                          │   │
│  │  Client Version: v1.28.0                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Pre-installed: az, terraform, kubectl, docker,            │
│                 git, vim, code, .NET SDK, Node.js          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Access Cloud Shell

```bash
# Option 1: Portal
# Click >_ icon in top navigation

# Option 2: Direct URL
# https://shell.azure.com

# Option 3: VS Code
# Install Azure Account extension
# Ctrl+Shift+P > "Azure: Open Bash in Cloud Shell"
```

### Cloud Shell Storage

```bash
# First-time setup creates:
# - Storage account (in new RG)
# - File share (5GB)
# - Mounted at $HOME/clouddrive

# Files in clouddrive persist between sessions
ls ~/clouddrive

# Upload files via portal or:
# Drag and drop into Cloud Shell window
```

---

## Initial Setup Checklist

### DevOps Ready Setup

```bash
# 1. Login to Azure
az login

# 2. Set default subscription
az account set --subscription "Development"

# 3. Create resource group for DevOps resources
az group create \
  --name rg-devops-shared \
  --location eastus \
  --tags purpose=devops team=platform

# 4. Register required providers
az provider register --namespace Microsoft.Web
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.KeyVault
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.ContainerService

# 5. Check registration status
az provider list --query "[?registrationState=='Registered'].namespace" -o table

# 6. Set default location
az configure --defaults location=eastus

# 7. Set default resource group
az configure --defaults group=rg-devops-shared
```

### Configure CLI Defaults

```bash
# View current defaults
az configure --list-defaults

# Set defaults
az configure --defaults \
  location=eastus \
  group=rg-myapp-dev

# Now you can omit --location and --resource-group
az storage account create --name mystorageacct
# Instead of:
# az storage account create --name mystorageacct --resource-group rg-myapp-dev --location eastus
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Azure Account** | Access to Azure services |
| **Subscription** | Billing and access boundary |
| **Management Group** | Organize subscriptions |
| **Resource Group** | Logical container for resources |
| **Azure CLI** | Command-line management |
| **Azure PowerShell** | PowerShell-based management |
| **Cloud Shell** | Browser-based CLI |

### Quick Reference

```bash
# Essential CLI commands
az login                          # Authenticate
az account list -o table          # List subscriptions
az account set -s "Name"          # Switch subscription
az group create -n rg-name -l eastus  # Create RG
az group list -o table            # List RGs
az resource list -g rg-name -o table  # List resources in RG
```
