# Azure CLI Essentials

## CLI Overview

The Azure CLI (`az`) is a cross-platform command-line tool for managing Azure resources.

```
┌─────────────────────────────────────────────────────────────┐
│                    AZURE CLI STRUCTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  az <group> <subgroup> <command> <arguments>               │
│                                                             │
│  Examples:                                                  │
│  az vm create --name myVM --resource-group myRG            │
│  az webapp list --resource-group myRG                      │
│  az storage account show --name mystorageacct              │
│                                                             │
│  Common Groups:                                             │
│  • az vm        - Virtual machines                         │
│  • az webapp    - App Service                              │
│  • az sql       - Azure SQL                                │
│  • az acr       - Container Registry                       │
│  • az aks       - Kubernetes Service                       │
│  • az keyvault  - Key Vault                                │
│  • az storage   - Storage accounts                         │
│  • az network   - Networking                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Getting Help

```bash
# General help
az --help

# Help for a group
az vm --help

# Help for a specific command
az vm create --help

# Find commands
az find "create web app"

# Interactive mode (with auto-complete)
az interactive
```

---

## Output Formats

### Available Formats

```bash
# JSON (default)
az group list

# Table (human readable)
az group list --output table
az group list -o table

# TSV (for scripting)
az group list -o tsv

# YAML
az group list -o yaml

# JSON with specific fields (JMESPath query)
az group list --query "[].{Name:name, Location:location}" -o table
```

### Query Examples (JMESPath)

```bash
# Get specific property
az vm show -g myRG -n myVM --query "hardwareProfile.vmSize" -o tsv

# Filter results
az vm list --query "[?powerState=='VM running'].name" -o table

# Multiple properties
az vm list --query "[].{Name:name, Size:hardwareProfile.vmSize, State:powerState}" -o table

# First item
az vm list --query "[0]"

# Count items
az vm list --query "length(@)"
```

---

## Resource Management

### Resource Groups

```bash
# Create resource group
az group create \
  --name rg-myapp-dev \
  --location eastus \
  --tags env=dev project=myapp

# List resource groups
az group list -o table

# Show resource group details
az group show --name rg-myapp-dev

# Delete resource group
az group delete --name rg-myapp-dev --yes --no-wait

# List all resources in a group
az resource list --resource-group rg-myapp-dev -o table

# Export ARM template from resource group
az group export --name rg-myapp-dev > template.json
```

### Tags

```bash
# Add tags to resource group
az group update \
  --name rg-myapp-dev \
  --tags env=dev project=myapp owner=team@company.com

# Add tag to a resource
az resource tag \
  --tags env=dev \
  --ids /subscriptions/.../resourceGroups/rg-myapp-dev/providers/Microsoft.Web/sites/mywebapp

# List resources by tag
az resource list --tag env=dev -o table

# Remove a tag
az group update --name rg-myapp-dev --remove tags.owner
```

---

## Common Service Commands

### App Service (Web Apps)

```bash
# List App Service plans
az appservice plan list -o table

# Create App Service plan
az appservice plan create \
  --name asp-myapp-dev \
  --resource-group rg-myapp-dev \
  --sku B1 \
  --is-linux

# Create web app
az webapp create \
  --name app-myapp-dev \
  --resource-group rg-myapp-dev \
  --plan asp-myapp-dev \
  --runtime "DOTNET|8.0"

# List web apps
az webapp list -o table

# Show web app details
az webapp show --name app-myapp-dev --resource-group rg-myapp-dev

# Get web app URL
az webapp show -n app-myapp-dev -g rg-myapp-dev --query "defaultHostName" -o tsv

# Start/Stop/Restart
az webapp start --name app-myapp-dev --resource-group rg-myapp-dev
az webapp stop --name app-myapp-dev --resource-group rg-myapp-dev
az webapp restart --name app-myapp-dev --resource-group rg-myapp-dev

# View logs
az webapp log tail --name app-myapp-dev --resource-group rg-myapp-dev

# Deploy from ZIP
az webapp deployment source config-zip \
  --resource-group rg-myapp-dev \
  --name app-myapp-dev \
  --src app.zip
```

### Azure SQL

```bash
# Create SQL server
az sql server create \
  --name sql-myapp-dev \
  --resource-group rg-myapp-dev \
  --location eastus \
  --admin-user sqladmin \
  --admin-password 'SecureP@ssw0rd!'

# Create database
az sql db create \
  --resource-group rg-myapp-dev \
  --server sql-myapp-dev \
  --name myappdb \
  --service-objective S0

# Configure firewall (allow Azure services)
az sql server firewall-rule create \
  --resource-group rg-myapp-dev \
  --server sql-myapp-dev \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Get connection string
az sql db show-connection-string \
  --server sql-myapp-dev \
  --name myappdb \
  --client ado.net
```

### Storage Account

```bash
# Create storage account
az storage account create \
  --name stmyappdev \
  --resource-group rg-myapp-dev \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

# Get connection string
az storage account show-connection-string \
  --name stmyappdev \
  --resource-group rg-myapp-dev \
  --query "connectionString" -o tsv

# Create blob container
az storage container create \
  --name uploads \
  --account-name stmyappdev

# Upload blob
az storage blob upload \
  --account-name stmyappdev \
  --container-name uploads \
  --name myfile.txt \
  --file ./myfile.txt

# List blobs
az storage blob list \
  --account-name stmyappdev \
  --container-name uploads \
  -o table
```

### Key Vault

```bash
# Create Key Vault
az keyvault create \
  --name kv-myapp-dev \
  --resource-group rg-myapp-dev \
  --location eastus

# Add secret
az keyvault secret set \
  --vault-name kv-myapp-dev \
  --name "DatabasePassword" \
  --value "SuperSecretPassword123!"

# Get secret
az keyvault secret show \
  --vault-name kv-myapp-dev \
  --name "DatabasePassword" \
  --query "value" -o tsv

# List secrets
az keyvault secret list --vault-name kv-myapp-dev -o table
```

### Container Registry

```bash
# Create ACR
az acr create \
  --name acrmyappdev \
  --resource-group rg-myapp-dev \
  --sku Basic

# Login to ACR
az acr login --name acrmyappdev

# Build and push image
az acr build \
  --registry acrmyappdev \
  --image myapp:v1 \
  .

# List images
az acr repository list --name acrmyappdev -o table

# Show tags for an image
az acr repository show-tags --name acrmyappdev --repository myapp -o table
```

---

## Scripting with Azure CLI

### Bash Script Example

```bash
#!/bin/bash

# Variables
RESOURCE_GROUP="rg-myapp-dev"
LOCATION="eastus"
APP_NAME="app-myapp-dev"
PLAN_NAME="asp-myapp-dev"

# Create resource group
echo "Creating resource group..."
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --tags env=dev project=myapp

# Create App Service plan
echo "Creating App Service plan..."
az appservice plan create \
  --name $PLAN_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku B1 \
  --is-linux

# Create web app
echo "Creating web app..."
az webapp create \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --plan $PLAN_NAME \
  --runtime "DOTNET|8.0"

# Get the URL
URL=$(az webapp show -n $APP_NAME -g $RESOURCE_GROUP --query "defaultHostName" -o tsv)
echo "Web app created at: https://$URL"
```

### PowerShell Script Example

```powershell
# Variables
$resourceGroup = "rg-myapp-dev"
$location = "eastus"
$appName = "app-myapp-dev"
$planName = "asp-myapp-dev"

# Create resource group
Write-Host "Creating resource group..."
az group create `
  --name $resourceGroup `
  --location $location `
  --tags env=dev project=myapp

# Create App Service plan
Write-Host "Creating App Service plan..."
az appservice plan create `
  --name $planName `
  --resource-group $resourceGroup `
  --sku B1 `
  --is-linux

# Create web app
Write-Host "Creating web app..."
az webapp create `
  --name $appName `
  --resource-group $resourceGroup `
  --plan $planName `
  --runtime "DOTNET|8.0"

# Get the URL
$url = az webapp show -n $appName -g $resourceGroup --query "defaultHostName" -o tsv
Write-Host "Web app created at: https://$url"
```

### Using Variables and Conditionals

```bash
#!/bin/bash

# Check if resource group exists
if az group exists --name rg-myapp-dev | grep -q true; then
    echo "Resource group exists"
else
    echo "Creating resource group..."
    az group create --name rg-myapp-dev --location eastus
fi

# Loop through environments
for ENV in dev staging prod; do
    echo "Creating resources for $ENV..."
    az group create --name "rg-myapp-$ENV" --location eastus
done

# Use output from one command in another
VAULT_ID=$(az keyvault show --name kv-myapp-dev --query "id" -o tsv)
echo "Key Vault ID: $VAULT_ID"
```

---

## Service Principal for CI/CD

### Create Service Principal

```bash
# Create service principal with contributor role
az ad sp create-for-rbac \
  --name "sp-myapp-cicd" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/rg-myapp-dev

# Output:
# {
#   "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#   "displayName": "sp-myapp-cicd",
#   "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
#   "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# }

# Create with specific roles
az ad sp create-for-rbac \
  --name "sp-myapp-cicd" \
  --role "Website Contributor" \
  --scopes /subscriptions/<sub-id>/resourceGroups/rg-myapp-dev
```

### Login with Service Principal

```bash
# Set environment variables (for CI/CD)
export AZURE_CLIENT_ID="<appId>"
export AZURE_CLIENT_SECRET="<password>"
export AZURE_TENANT_ID="<tenant>"

# Login
az login --service-principal \
  --username $AZURE_CLIENT_ID \
  --password $AZURE_CLIENT_SECRET \
  --tenant $AZURE_TENANT_ID
```

---

## Configuration and Defaults

### Set Defaults

```bash
# Set default subscription
az account set --subscription "My Subscription"

# Set default location and resource group
az configure --defaults location=eastus group=rg-myapp-dev

# View defaults
az configure --list-defaults

# Clear defaults
az configure --defaults location='' group=''
```

### Configuration File

```bash
# Location: ~/.azure/config

# View config
cat ~/.azure/config

# Example config:
[defaults]
location = eastus
group = rg-myapp-dev

[core]
output = table
```

---

## Useful Commands for DevOps

### Quick Reference

```bash
# Account & Subscription
az login                              # Login
az account list -o table              # List subscriptions
az account set -s "Name"              # Switch subscription
az account show                       # Current subscription

# Resource Groups
az group create -n rg-name -l eastus  # Create
az group list -o table                # List
az group delete -n rg-name --yes      # Delete

# App Service
az webapp list -o table               # List web apps
az webapp create ...                  # Create web app
az webapp log tail -n name -g rg      # Stream logs
az webapp deployment source config-zip ... # Deploy ZIP

# Container Registry
az acr login --name acrname           # Login to ACR
az acr build --registry acr --image img:tag . # Build & push

# Key Vault
az keyvault secret list --vault-name kv # List secrets
az keyvault secret show --vault-name kv --name secret # Get secret

# Monitoring
az monitor metrics list --resource <id> # List metrics
az monitor activity-log list            # Activity log
```

### Cleanup Script

```bash
#!/bin/bash
# Cleanup all dev resources

echo "Deleting development resources..."

# Delete resource groups starting with 'rg-dev-'
for RG in $(az group list --query "[?starts_with(name, 'rg-dev-')].name" -o tsv); do
    echo "Deleting $RG..."
    az group delete --name $RG --yes --no-wait
done

echo "Cleanup initiated. Resources will be deleted in background."
```

---

## Summary

| Command | Description |
|---------|-------------|
| `az login` | Authenticate to Azure |
| `az group create` | Create resource group |
| `az webapp create` | Create App Service |
| `az sql server create` | Create SQL Server |
| `az keyvault create` | Create Key Vault |
| `az acr create` | Create Container Registry |
| `az configure --defaults` | Set CLI defaults |
| `--query` | Filter output with JMESPath |
| `-o table` | Table output format |
