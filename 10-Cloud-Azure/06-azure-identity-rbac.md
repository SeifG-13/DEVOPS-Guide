# Azure Identity and RBAC

## Azure Active Directory (Azure AD / Entra ID)

Azure AD is Microsoft's cloud-based identity and access management service.

```
┌─────────────────────────────────────────────────────────────┐
│                    AZURE AD / ENTRA ID                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Azure AD Tenant                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Identities:                                        │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐              │   │
│  │  │  Users  │ │ Groups  │ │  Apps   │              │   │
│  │  └─────────┘ └─────────┘ └─────────┘              │   │
│  │                                                     │   │
│  │  Security:                                          │   │
│  │  ┌───────────────┐ ┌───────────────┐              │   │
│  │  │ Conditional   │ │     MFA       │              │   │
│  │  │   Access      │ │               │              │   │
│  │  └───────────────┘ └───────────────┘              │   │
│  │                                                     │   │
│  │  Enterprise:                                        │   │
│  │  ┌───────────────┐ ┌───────────────┐              │   │
│  │  │ Service       │ │   Managed     │              │   │
│  │  │ Principals    │ │  Identities   │              │   │
│  │  └───────────────┘ └───────────────┘              │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Identity Types

### Comparison

| Identity Type | Use Case | Password/Secret |
|---------------|----------|-----------------|
| **User** | Human access | Password + MFA |
| **Service Principal** | CI/CD pipelines, automation | Client secret or certificate |
| **Managed Identity** | Azure resources | No secrets (Azure managed) |
| **Group** | Collective access management | N/A |

---

## Service Principals

A Service Principal is an identity for applications/services to access Azure resources.

```
┌─────────────────────────────────────────────────────────────┐
│                  SERVICE PRINCIPAL                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  App Registration (Azure AD)                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Application ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx │   │
│  │  Directory ID:   xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx │   │
│  │                                                     │   │
│  │  Credentials:                                       │   │
│  │  ├── Client Secret (password)                      │   │
│  │  └── Certificate (more secure)                     │   │
│  └─────────────────────────────────────────────────────┘   │
│         │                                                   │
│         ▼                                                   │
│  Service Principal (Enterprise App)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Object ID: yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy    │   │
│  │                                                     │   │
│  │  RBAC Assignments:                                  │   │
│  │  ├── Contributor on rg-myapp-dev                   │   │
│  │  └── Reader on subscription                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create Service Principal for CI/CD

```bash
# Create service principal with Contributor role on resource group
az ad sp create-for-rbac \
  --name "sp-myapp-cicd" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/rg-myapp-dev

# Output (SAVE THESE VALUES!):
# {
#   "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",        # Client ID
#   "displayName": "sp-myapp-cicd",
#   "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",         # Client Secret
#   "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"        # Tenant ID
# }

# Create with custom role and multiple scopes
az ad sp create-for-rbac \
  --name "sp-myapp-prod" \
  --role "Website Contributor" \
  --scopes /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod \
           /subscriptions/<sub-id>/resourceGroups/rg-shared

# Reset credentials
az ad sp credential reset --id <app-id>

# List service principals
az ad sp list --display-name "sp-myapp" -o table

# Delete service principal
az ad sp delete --id <app-id>
```

### Using Service Principal in CI/CD

```yaml
# Azure Pipelines - azure-pipelines.yml
variables:
  - name: AZURE_CLIENT_ID
    value: $(SP_CLIENT_ID)  # From pipeline variables/secrets
  - name: AZURE_CLIENT_SECRET
    value: $(SP_CLIENT_SECRET)
  - name: AZURE_TENANT_ID
    value: $(SP_TENANT_ID)

steps:
  - task: AzureCLI@2
    inputs:
      azureSubscription: 'my-service-connection'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az webapp deployment source config-zip \
          --resource-group rg-myapp-dev \
          --name app-myapp-dev \
          --src app.zip
```

```yaml
# GitHub Actions
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
          # AZURE_CREDENTIALS = {"clientId":"...","clientSecret":"...","subscriptionId":"...","tenantId":"..."}
```

---

## Managed Identities

Managed Identities provide an automatically managed identity for Azure resources.

```
┌─────────────────────────────────────────────────────────────┐
│                  MANAGED IDENTITY                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  System-Assigned                  User-Assigned             │
│  ┌─────────────────────┐         ┌─────────────────────┐   │
│  │ 1 identity per      │         │ Standalone identity │   │
│  │   resource          │         │                     │   │
│  │                     │         │ Can be assigned to  │   │
│  │ Lifecycle tied to   │         │ multiple resources  │   │
│  │   resource          │         │                     │   │
│  │                     │         │ Independent         │   │
│  │ Delete resource =   │         │ lifecycle           │   │
│  │   delete identity   │         │                     │   │
│  └─────────────────────┘         └─────────────────────┘   │
│                                                             │
│  Benefits:                                                  │
│  • No secrets to manage                                    │
│  • Auto-rotated credentials                                │
│  • Azure manages everything                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Enable Managed Identity

```bash
# Enable system-assigned managed identity on App Service
az webapp identity assign \
  --name app-myapp-prod \
  --resource-group rg-myapp-prod

# Output:
# {
#   "principalId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#   "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#   "type": "SystemAssigned"
# }

# Create user-assigned managed identity
az identity create \
  --name id-myapp-shared \
  --resource-group rg-myapp-shared

# Assign user-assigned identity to App Service
az webapp identity assign \
  --name app-myapp-prod \
  --resource-group rg-myapp-prod \
  --identities /subscriptions/<sub-id>/resourceGroups/rg-myapp-shared/providers/Microsoft.ManagedIdentity/userAssignedIdentities/id-myapp-shared
```

### Use Managed Identity to Access Key Vault

```bash
# Get App Service's managed identity principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name app-myapp-prod \
  --resource-group rg-myapp-prod \
  --query "principalId" -o tsv)

# Grant access to Key Vault
az keyvault set-policy \
  --name kv-myapp-prod \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get list

# Or use RBAC (recommended)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.KeyVault/vaults/kv-myapp-prod
```

### Access Key Vault in .NET 10

```csharp
// Program.cs
using Azure.Identity;
using Azure.Extensions.AspNetCore.Configuration.Secrets;

var builder = WebApplication.CreateBuilder(args);

// Use Managed Identity to access Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri("https://kv-myapp-prod.vault.azure.net/"),
    new DefaultAzureCredential());

// Secrets are now available via Configuration
var dbConnection = builder.Configuration["DatabaseConnectionString"];
```

```xml
<!-- Add NuGet packages -->
<PackageReference Include="Azure.Identity" Version="1.10.4" />
<PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" Version="1.3.0" />
```

---

## Role-Based Access Control (RBAC)

RBAC manages who has access to Azure resources and what they can do with them.

```
┌─────────────────────────────────────────────────────────────┐
│                        RBAC                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Role Assignment = Security Principal + Role + Scope        │
│                                                             │
│  ┌──────────────────┐                                      │
│  │Security Principal│  WHO?                                │
│  │ - User           │  ─────▶ alice@company.com            │
│  │ - Group          │         sp-myapp-cicd                │
│  │ - Service Princ. │         App Service (MI)             │
│  │ - Managed ID     │                                      │
│  └──────────────────┘                                      │
│           +                                                 │
│  ┌──────────────────┐                                      │
│  │      Role        │  WHAT?                               │
│  │ - Owner          │  ─────▶ Contributor                  │
│  │ - Contributor    │         (full access except RBAC)    │
│  │ - Reader         │                                      │
│  │ - Custom         │                                      │
│  └──────────────────┘                                      │
│           +                                                 │
│  ┌──────────────────┐                                      │
│  │      Scope       │  WHERE?                              │
│  │ - Management Grp │  ─────▶ /subscriptions/xxx/          │
│  │ - Subscription   │         resourceGroups/rg-myapp      │
│  │ - Resource Group │                                      │
│  │ - Resource       │                                      │
│  └──────────────────┘                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Built-in Roles

| Role | Permissions |
|------|-------------|
| **Owner** | Full access + manage RBAC |
| **Contributor** | Full access, no RBAC management |
| **Reader** | View only |
| **User Access Administrator** | Manage user access only |
| **Website Contributor** | Manage websites, not App Service plans |
| **SQL DB Contributor** | Manage SQL databases |
| **Key Vault Administrator** | Manage Key Vault |
| **Key Vault Secrets User** | Read secrets only |

### Assign Roles

```bash
# Assign role to user
az role assignment create \
  --role "Contributor" \
  --assignee alice@company.com \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-dev

# Assign role to service principal
az role assignment create \
  --role "Website Contributor" \
  --assignee <service-principal-app-id> \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod

# Assign role to managed identity
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <managed-identity-principal-id> \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.KeyVault/vaults/kv-myapp-prod

# List role assignments
az role assignment list \
  --resource-group rg-myapp-dev \
  -o table

# Remove role assignment
az role assignment delete \
  --role "Contributor" \
  --assignee alice@company.com \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-dev
```

### Bicep RBAC Assignment

```bicep
param principalId string
param roleDefinitionId string = 'b24988ac-6180-42a0-ab88-20f7382dd24c' // Contributor

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, principalId, roleDefinitionId)
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', roleDefinitionId)
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}
```

---

## Workload Identity Federation (OIDC)

Passwordless authentication for GitHub Actions and Azure DevOps.

```
┌─────────────────────────────────────────────────────────────┐
│             WORKLOAD IDENTITY FEDERATION                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Traditional (Secret-based)        OIDC (Federated)         │
│  ┌─────────────────────────┐      ┌────────────────────┐   │
│  │ GitHub Actions          │      │ GitHub Actions     │   │
│  │   │                     │      │   │                │   │
│  │   │ Client Secret       │      │   │ OIDC Token     │   │
│  │   │ (stored in secrets) │      │   │ (auto-issued)  │   │
│  │   ▼                     │      │   ▼                │   │
│  │ Azure AD                │      │ Azure AD           │   │
│  │   │                     │      │ (Trust GitHub)     │   │
│  │   │ Access Token        │      │   │                │   │
│  │   ▼                     │      │   │ Access Token   │   │
│  │ Azure Resources         │      │   ▼                │   │
│  └─────────────────────────┘      │ Azure Resources    │   │
│                                   └────────────────────┘   │
│  ✗ Secret rotation needed          ✓ No secrets!           │
│  ✗ Risk of secret exposure         ✓ More secure           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Configure OIDC for GitHub Actions

```bash
# Create app registration
az ad app create --display-name "gh-myapp-deploy"

# Get app ID
APP_ID=$(az ad app list --display-name "gh-myapp-deploy" --query "[0].appId" -o tsv)

# Create service principal
az ad sp create --id $APP_ID

# Add federated credential for GitHub
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "description": "GitHub Actions main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Assign role to service principal
SP_ID=$(az ad sp show --id $APP_ID --query "id" -o tsv)
az role assignment create \
  --role Contributor \
  --assignee $SP_ID \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod
```

### GitHub Actions with OIDC

```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: app-myapp-prod
          package: ./publish
```

---

## Best Practices

### Identity Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│              IDENTITY BEST PRACTICES                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Use Managed Identity when possible                      │
│     • No secrets to manage                                 │
│     • Auto-rotation                                        │
│                                                             │
│  2. Use OIDC/Workload Federation for CI/CD                 │
│     • GitHub Actions: Federated credentials                │
│     • Azure DevOps: Workload identity                      │
│                                                             │
│  3. Least Privilege Principle                               │
│     • Assign minimum required permissions                   │
│     • Use specific roles (not Owner/Contributor)           │
│                                                             │
│  4. Use Groups for Access                                   │
│     • Assign roles to groups, not individuals              │
│     • Easier to manage                                     │
│                                                             │
│  5. Scope Appropriately                                     │
│     • Prefer resource/RG scope over subscription           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Azure AD** | Identity and access management |
| **Service Principal** | App identity for automation |
| **Managed Identity** | Passwordless Azure resource identity |
| **RBAC** | Role-based access control |
| **OIDC Federation** | Passwordless CI/CD authentication |

### Quick Commands

```bash
# Service Principal
az ad sp create-for-rbac --name sp-name --role Contributor --scopes /subscriptions/<id>/resourceGroups/rg

# Managed Identity
az webapp identity assign --name app-name --resource-group rg

# Role Assignment
az role assignment create --role "Role Name" --assignee <principal-id> --scope <scope>

# List Roles
az role definition list --output table
az role assignment list --resource-group rg -o table
```
