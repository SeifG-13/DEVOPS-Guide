# Azure Key Vault

## What is Azure Key Vault?

Azure Key Vault is a cloud service for securely storing and accessing secrets, keys, and certificates.

```
┌─────────────────────────────────────────────────────────────┐
│                    AZURE KEY VAULT                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Key Vault: kv-myapp-prod                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Secrets                    Keys                    │   │
│  │  ┌─────────────────────┐   ┌─────────────────────┐ │   │
│  │  │ DatabasePassword    │   │ EncryptionKey       │ │   │
│  │  │ ApiKey              │   │ SigningKey          │ │   │
│  │  │ ConnectionStrings   │   │                     │ │   │
│  │  └─────────────────────┘   └─────────────────────┘ │   │
│  │                                                     │   │
│  │  Certificates                                       │   │
│  │  ┌─────────────────────┐                           │   │
│  │  │ SSL Certificate     │                           │   │
│  │  │ Code Signing Cert   │                           │   │
│  │  └─────────────────────┘                           │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Access: RBAC or Access Policies                           │
│  Network: Public or Private Endpoint                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Create Key Vault

### Azure CLI

```bash
# Variables
RESOURCE_GROUP="rg-myapp-prod"
KEY_VAULT="kv-myapp-prod"
LOCATION="eastus"

# Create Key Vault
az keyvault create \
  --name $KEY_VAULT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku standard \
  --enable-rbac-authorization true

# With soft delete and purge protection
az keyvault create \
  --name $KEY_VAULT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-soft-delete true \
  --soft-delete-retention-days 90 \
  --enable-purge-protection true
```

### Bicep

```bicep
param keyVaultName string
param location string = resourceGroup().location

resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
  }
}

output keyVaultUri string = keyVault.properties.vaultUri
output keyVaultId string = keyVault.id
```

---

## Managing Secrets

### Add and Retrieve Secrets

```bash
# Set a secret
az keyvault secret set \
  --vault-name $KEY_VAULT \
  --name "DatabasePassword" \
  --value "MySecureP@ssw0rd123!"

# Set secret with expiration
az keyvault secret set \
  --vault-name $KEY_VAULT \
  --name "ApiKey" \
  --value "api-key-value" \
  --expires "2025-12-31T23:59:59Z"

# Get secret value
az keyvault secret show \
  --vault-name $KEY_VAULT \
  --name "DatabasePassword" \
  --query "value" -o tsv

# List all secrets
az keyvault secret list \
  --vault-name $KEY_VAULT \
  -o table

# Delete secret (soft delete)
az keyvault secret delete \
  --vault-name $KEY_VAULT \
  --name "OldSecret"

# Recover deleted secret
az keyvault secret recover \
  --vault-name $KEY_VAULT \
  --name "OldSecret"

# Permanently delete (purge)
az keyvault secret purge \
  --vault-name $KEY_VAULT \
  --name "OldSecret"
```

### Secret Versions

```bash
# List all versions of a secret
az keyvault secret list-versions \
  --vault-name $KEY_VAULT \
  --name "DatabasePassword" \
  -o table

# Get specific version
az keyvault secret show \
  --vault-name $KEY_VAULT \
  --name "DatabasePassword" \
  --version "abc123"
```

---

## Access Control

### RBAC (Recommended)

```bash
# Key Vault built-in roles:
# - Key Vault Administrator: Full access
# - Key Vault Secrets User: Read secrets only
# - Key Vault Secrets Officer: Manage secrets
# - Key Vault Certificates Officer: Manage certificates
# - Key Vault Crypto Officer: Manage keys

# Grant secrets access to user
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee user@company.com \
  --scope /subscriptions/<sub-id>/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEY_VAULT

# Grant secrets access to managed identity
PRINCIPAL_ID=$(az webapp identity show -n app-myapi-prod -g $RESOURCE_GROUP --query "principalId" -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/<sub-id>/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEY_VAULT

# Grant secrets access to service principal
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <sp-app-id> \
  --scope /subscriptions/<sub-id>/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEY_VAULT
```

### Access Policies (Legacy)

```bash
# Grant access policy to managed identity
az keyvault set-policy \
  --name $KEY_VAULT \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get list

# Grant full access to service principal
az keyvault set-policy \
  --name $KEY_VAULT \
  --spn <sp-app-id> \
  --secret-permissions get list set delete

# Grant access to user
az keyvault set-policy \
  --name $KEY_VAULT \
  --upn user@company.com \
  --secret-permissions get list set delete

# Remove access policy
az keyvault delete-policy \
  --name $KEY_VAULT \
  --object-id $PRINCIPAL_ID
```

---

## Integration with .NET 10

### Add NuGet Packages

```bash
dotnet add package Azure.Identity
dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
dotnet add package Azure.Security.KeyVault.Secrets
```

### Configure Key Vault in Program.cs

```csharp
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

// Add Key Vault configuration
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = builder.Configuration["KeyVault:Uri"];
    if (!string.IsNullOrEmpty(keyVaultUri))
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUri),
            new DefaultAzureCredential());
    }
}

// Secrets are now available via IConfiguration
// Secret named "DatabasePassword" → Configuration["DatabasePassword"]
// Secret named "ConnectionStrings--DefaultConnection" → Configuration.GetConnectionString("DefaultConnection")
```

### appsettings.json

```json
{
  "KeyVault": {
    "Uri": "https://kv-myapp-prod.vault.azure.net/"
  }
}
```

### Direct Access to Key Vault

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

public class SecretsService
{
    private readonly SecretClient _client;

    public SecretsService(IConfiguration config)
    {
        var vaultUri = new Uri(config["KeyVault:Uri"]!);
        _client = new SecretClient(vaultUri, new DefaultAzureCredential());
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        KeyVaultSecret secret = await _client.GetSecretAsync(secretName);
        return secret.Value;
    }

    public async Task SetSecretAsync(string secretName, string value)
    {
        await _client.SetSecretAsync(secretName, value);
    }
}
```

---

## Key Vault References in App Service

### Configure App Settings with Key Vault References

```bash
# Set app setting that references Key Vault
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group $RESOURCE_GROUP \
  --settings \
    "DatabasePassword=@Microsoft.KeyVault(SecretUri=https://kv-myapp-prod.vault.azure.net/secrets/DatabasePassword/)" \
    "ApiKey=@Microsoft.KeyVault(VaultName=kv-myapp-prod;SecretName=ApiKey)"

# Connection string with Key Vault reference
az webapp config connection-string set \
  --name app-myapi-prod \
  --resource-group $RESOURCE_GROUP \
  --connection-string-type SQLAzure \
  --settings \
    "DefaultConnection=@Microsoft.KeyVault(SecretUri=https://kv-myapp-prod.vault.azure.net/secrets/ConnectionStrings--DefaultConnection/)"
```

### Reference Formats

```bash
# Full URI (with or without version)
@Microsoft.KeyVault(SecretUri=https://kv-name.vault.azure.net/secrets/SecretName/)
@Microsoft.KeyVault(SecretUri=https://kv-name.vault.azure.net/secrets/SecretName/abc123)

# Short format
@Microsoft.KeyVault(VaultName=kv-name;SecretName=SecretName)
@Microsoft.KeyVault(VaultName=kv-name;SecretName=SecretName;SecretVersion=abc123)
```

---

## CI/CD Integration

### Azure Pipelines

```yaml
# azure-pipelines.yml
variables:
  - group: my-keyvault-secrets  # Variable group linked to Key Vault

steps:
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: 'Azure-ServiceConnection'
      keyVaultName: 'kv-myapp-prod'
      secretsFilter: 'DatabasePassword,ApiKey'
      runAsPreJob: true

  - script: |
      echo "Using secret in script"
      # Secret is available as $(DatabasePassword)
    env:
      DB_PASSWORD: $(DatabasePassword)
```

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get Key Vault Secrets
        uses: Azure/get-keyvault-secrets@v1
        with:
          keyvault: 'kv-myapp-prod'
          secrets: 'DatabasePassword,ApiKey'
        id: keyvault

      - name: Use Secrets
        run: |
          echo "Secret is available"
        env:
          DB_PASSWORD: ${{ steps.keyvault.outputs.DatabasePassword }}
```

---

## Managing Keys

### Create and Use Cryptographic Keys

```bash
# Create RSA key
az keyvault key create \
  --vault-name $KEY_VAULT \
  --name "EncryptionKey" \
  --kty RSA \
  --size 2048

# Create EC key
az keyvault key create \
  --vault-name $KEY_VAULT \
  --name "SigningKey" \
  --kty EC \
  --curve P-256

# List keys
az keyvault key list --vault-name $KEY_VAULT -o table

# Get key details
az keyvault key show --vault-name $KEY_VAULT --name "EncryptionKey"

# Rotate key (create new version)
az keyvault key rotate --vault-name $KEY_VAULT --name "EncryptionKey"
```

---

## Managing Certificates

### Import and Create Certificates

```bash
# Import PFX certificate
az keyvault certificate import \
  --vault-name $KEY_VAULT \
  --name "MyCertificate" \
  --file mycert.pfx \
  --password "certpassword"

# Create self-signed certificate
az keyvault certificate create \
  --vault-name $KEY_VAULT \
  --name "SelfSignedCert" \
  --policy "$(az keyvault certificate get-default-policy)"

# List certificates
az keyvault certificate list --vault-name $KEY_VAULT -o table

# Download certificate
az keyvault certificate download \
  --vault-name $KEY_VAULT \
  --name "MyCertificate" \
  --file downloaded-cert.pem
```

---

## Private Endpoint

### Secure Key Vault with Private Link

```bash
# Create private endpoint
az network private-endpoint create \
  --name pe-kv-myapp \
  --resource-group $RESOURCE_GROUP \
  --vnet-name vnet-myapp-prod \
  --subnet snet-private-endpoints \
  --private-connection-resource-id $(az keyvault show --name $KEY_VAULT --query "id" -o tsv) \
  --group-id vault \
  --connection-name kv-connection

# Disable public access
az keyvault update \
  --name $KEY_VAULT \
  --public-network-access Disabled

# Create private DNS zone
az network private-dns zone create \
  --name privatelink.vaultcore.azure.net \
  --resource-group $RESOURCE_GROUP

# Link to VNet
az network private-dns link vnet create \
  --name kv-dns-link \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.vaultcore.azure.net \
  --virtual-network vnet-myapp-prod \
  --registration-enabled false

# Create DNS zone group
az network private-endpoint dns-zone-group create \
  --name kv-dns-group \
  --resource-group $RESOURCE_GROUP \
  --endpoint-name pe-kv-myapp \
  --private-dns-zone privatelink.vaultcore.azure.net \
  --zone-name default
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│              KEY VAULT BEST PRACTICES                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Use RBAC Authorization                                  │
│     • More granular control                                │
│     • Aligns with Azure RBAC                               │
│                                                             │
│  2. Enable Soft Delete & Purge Protection                   │
│     • Prevent accidental deletion                          │
│     • Required for production                              │
│                                                             │
│  3. Use Managed Identity                                    │
│     • No secrets in code or config                         │
│     • Auto-rotated credentials                             │
│                                                             │
│  4. Use Key Vault References                                │
│     • Don't copy secrets to app settings                   │
│     • Automatic secret rotation                            │
│                                                             │
│  5. Enable Private Endpoint                                 │
│     • No public internet exposure                          │
│     • Access only from VNet                                │
│                                                             │
│  6. Rotate Secrets Regularly                                │
│     • Use versioning                                       │
│     • Automate rotation                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

| Feature | Command/Usage |
|---------|---------------|
| Create vault | `az keyvault create` |
| Add secret | `az keyvault secret set` |
| Get secret | `az keyvault secret show` |
| Grant access | `az role assignment create` or `az keyvault set-policy` |
| App Service ref | `@Microsoft.KeyVault(SecretUri=...)` |
| .NET integration | `builder.Configuration.AddAzureKeyVault()` |

### Quick Commands

```bash
# Create Key Vault
az keyvault create -n kv-name -g rg --enable-rbac-authorization

# Secrets
az keyvault secret set --vault-name kv-name -n SecretName --value "secret"
az keyvault secret show --vault-name kv-name -n SecretName --query "value" -o tsv

# Grant access
az role assignment create --role "Key Vault Secrets User" --assignee <id> --scope <kv-resource-id>

# List
az keyvault secret list --vault-name kv-name -o table
```
