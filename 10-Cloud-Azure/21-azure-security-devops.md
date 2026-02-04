# Azure Security for DevOps

## Security Overview

Implementing security throughout the DevOps lifecycle (DevSecOps) on Azure.

```
┌─────────────────────────────────────────────────────────────┐
│                    DEVSECOPS ON AZURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │  Plan   │─▶│  Code   │─▶│  Build  │─▶│  Test   │       │
│  │         │  │         │  │         │  │         │       │
│  │ Threat  │  │ SAST    │  │ Dep     │  │ DAST    │       │
│  │ Model   │  │ Secrets │  │ Scan    │  │ Pen Test│       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│       │                                       │             │
│       │            ┌─────────────────┐        │             │
│       └───────────▶│ Security Center │◀───────┘             │
│                    │   (Defender)    │                      │
│                    └─────────────────┘                      │
│                            │                                │
│  ┌─────────┐  ┌─────────┐  ▼  ┌─────────┐  ┌─────────┐   │
│  │ Deploy  │─▶│ Operate │─▶│ Monitor │─▶│ Respond │     │
│  │         │  │         │  │         │  │         │       │
│  │ IaC     │  │ RBAC    │  │ Logs    │  │ Alerts  │       │
│  │ Secure  │  │ Network │  │ SIEM    │  │ Incident│       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Microsoft Defender for Cloud

### Enable Defender

```bash
# Enable Defender for Cloud (enhanced security)
az security pricing create \
  --name VirtualMachines \
  --tier Standard

az security pricing create \
  --name AppServices \
  --tier Standard

az security pricing create \
  --name SqlServers \
  --tier Standard

az security pricing create \
  --name StorageAccounts \
  --tier Standard

az security pricing create \
  --name Containers \
  --tier Standard

az security pricing create \
  --name KeyVaults \
  --tier Standard

# Check security score
az security secure-score-controls list -o table
```

### Security Recommendations

```bash
# List security recommendations
az security assessment list \
  --query "[?status.code=='Unhealthy'].{Name:displayName, Severity:metadata.severity}" \
  -o table

# Get detailed recommendation
az security assessment show \
  --name <assessment-id> \
  --assessed-resource-id <resource-id>
```

---

## Secure CI/CD Pipeline

### GitHub Actions Security

```yaml
# .github/workflows/secure-pipeline.yml
name: Secure CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write  # For CodeQL
  id-token: write         # For OIDC

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Secret scanning
      - name: TruffleHog Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

      # SAST with CodeQL
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp

      - name: Build
        run: dotnet build

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # .NET dependency vulnerabilities
      - name: .NET Security Scan
        run: |
          dotnet list package --vulnerable --include-transitive

      # Snyk vulnerability scan
      - name: Snyk Security Scan
        uses: snyk/actions/dotnet@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:scan .

      # Trivy container scan
      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:scan'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  deploy:
    needs: [security-scan, dependency-scan, container-scan]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      # OIDC authentication (no secrets!)
      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy
        run: |
          az webapp deployment source config-zip \
            --name app-myapi-prod \
            --resource-group rg-myapp-prod \
            --src app.zip
```

### Azure Pipelines Security

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: SecurityScans
    jobs:
      - job: CredentialScan
        steps:
          - task: CredScan@3
            displayName: 'Credential Scanner'

          - task: PostAnalysis@2
            displayName: 'Post Analysis'
            inputs:
              CredScan: true

      - job: DependencyScan
        steps:
          - script: dotnet list package --vulnerable
            displayName: 'Check vulnerable packages'

      - job: ContainerScan
        steps:
          - task: ContainerStructureTest@0
            displayName: 'Container structure test'

  - stage: Build
    dependsOn: SecurityScans
    jobs:
      - job: Build
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'build'
```

---

## Secure Application Configuration

### Never Hardcode Secrets

```csharp
// ❌ BAD - Never do this
var connectionString = "Server=sql.database.windows.net;Password=MySecret123!";

// ✅ GOOD - Use configuration
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

// ✅ BETTER - Use Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri("https://kv-myapp.vault.azure.net/"),
    new DefaultAzureCredential());

// ✅ BEST - Use Managed Identity + Key Vault References
// In App Service settings:
// ConnectionStrings__Default = @Microsoft.KeyVault(SecretUri=https://kv-myapp.vault.azure.net/secrets/DbConnection/)
```

### Secure appsettings

```json
// appsettings.json - No secrets!
{
  "KeyVault": {
    "Uri": "https://kv-myapp-prod.vault.azure.net/"
  },
  "ApplicationInsights": {
    "ConnectionString": ""  // Set via environment variable
  }
}
```

```bash
# .gitignore
appsettings.Development.json
appsettings.*.local.json
*.pfx
*.key
.env
```

---

## Network Security

### Private Endpoints

```bash
# Create private endpoint for SQL
az network private-endpoint create \
  --name pe-sql-myapp \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-myapp \
  --subnet snet-private \
  --private-connection-resource-id $(az sql server show -n sql-myapp -g rg-myapp-prod --query id -o tsv) \
  --group-id sqlServer \
  --connection-name sql-connection

# Disable public access
az sql server update \
  --name sql-myapp \
  --resource-group rg-myapp-prod \
  --public-network-access Disabled
```

### Network Security Groups

```bash
# Create NSG with secure rules
az network nsg create \
  --name nsg-web \
  --resource-group rg-myapp-prod

# Allow HTTPS only
az network nsg rule create \
  --nsg-name nsg-web \
  --resource-group rg-myapp-prod \
  --name AllowHTTPS \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443

# Deny all other inbound
az network nsg rule create \
  --nsg-name nsg-web \
  --resource-group rg-myapp-prod \
  --name DenyAllInbound \
  --priority 4096 \
  --direction Inbound \
  --access Deny \
  --protocol '*' \
  --destination-port-ranges '*'
```

### Web Application Firewall (WAF)

```bash
# Create Application Gateway with WAF
az network application-gateway waf-policy create \
  --name waf-policy-myapp \
  --resource-group rg-myapp-prod \
  --type OWASP \
  --version 3.2

# Enable WAF rules
az network application-gateway waf-policy managed-rule rule-set add \
  --policy-name waf-policy-myapp \
  --resource-group rg-myapp-prod \
  --type OWASP \
  --version 3.2
```

---

## Identity Security

### Managed Identity Best Practices

```bash
# Enable managed identity
az webapp identity assign \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod

# Grant minimal permissions
PRINCIPAL_ID=$(az webapp identity show -n app-myapi-prod -g rg-myapp-prod --query principalId -o tsv)

# Key Vault - secrets only, no management
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.KeyVault/vaults/kv-myapp-prod

# SQL - specific database only
# Use SQL RBAC instead of server admin
```

### RBAC Best Practices

```bash
# Create custom role with minimal permissions
az role definition create --role-definition '{
  "Name": "App Deployer",
  "Description": "Can deploy to App Service only",
  "Actions": [
    "Microsoft.Web/sites/publish/Action",
    "Microsoft.Web/sites/slots/publish/Action",
    "Microsoft.Web/sites/restart/Action"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/<sub-id>/resourceGroups/rg-myapp-prod"]
}'

# Assign custom role
az role assignment create \
  --role "App Deployer" \
  --assignee <sp-app-id> \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod
```

---

## Container Security

### Secure Dockerfile

```dockerfile
# Use specific version, not :latest
FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Don't expose unnecessary ports
EXPOSE 8080

# Use HTTPS
ENV ASPNETCORE_URLS=https://+:8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1
```

### ACR Security

```bash
# Enable content trust
az acr config content-trust update \
  --registry acrmyapp \
  --status enabled

# Enable vulnerability scanning
az security pricing create \
  --name Containers \
  --tier Standard

# Restrict network access
az acr update \
  --name acrmyapp \
  --public-network-enabled false
```

---

## Compliance and Governance

### Azure Policy

```bash
# Assign built-in security policies
az policy assignment create \
  --name "Require HTTPS for App Service" \
  --policy "a4af4a39-4135-47fb-b175-47fbdf85311d" \
  --scope /subscriptions/<sub-id>

az policy assignment create \
  --name "Require TLS 1.2" \
  --policy "f9d614cd-f1fd-4e61-bcec-0de4e1b5b86c" \
  --scope /subscriptions/<sub-id>

az policy assignment create \
  --name "Require Key Vault for secrets" \
  --policy "1a5b4dca-0b6f-4cf5-907c-56316bc1bf3d" \
  --scope /subscriptions/<sub-id>
```

### Security Benchmark

```bash
# Apply Azure Security Benchmark initiative
az policy assignment create \
  --name "Azure Security Benchmark" \
  --policy-set-definition "1f3afdf9-d0c9-4c3d-847f-89da613e70a8" \
  --scope /subscriptions/<sub-id>

# Check compliance
az policy state summarize \
  --resource-group rg-myapp-prod
```

---

## Secrets Management

### Key Vault Best Practices

```bash
# Enable soft delete and purge protection
az keyvault update \
  --name kv-myapp-prod \
  --enable-soft-delete true \
  --enable-purge-protection true

# Enable audit logging
az monitor diagnostic-settings create \
  --name kv-diagnostics \
  --resource $(az keyvault show -n kv-myapp-prod --query id -o tsv) \
  --workspace $(az monitor log-analytics workspace show -n log-myapp -g rg-myapp-prod --query id -o tsv) \
  --logs '[{"category": "AuditEvent", "enabled": true}]'

# Rotate secrets regularly
az keyvault secret set \
  --vault-name kv-myapp-prod \
  --name ApiKey \
  --value "new-rotated-value" \
  --expires $(date -d "+90 days" -u +"%Y-%m-%dT%H:%M:%SZ")
```

---

## Summary

| Security Layer | Tools/Services |
|----------------|----------------|
| **Code** | CodeQL, SAST, secret scanning |
| **Dependencies** | Snyk, Dependabot, dotnet audit |
| **Containers** | Trivy, ACR scanning, Defender |
| **Infrastructure** | Defender for Cloud, Azure Policy |
| **Identity** | Managed Identity, RBAC, OIDC |
| **Network** | NSG, Private Endpoints, WAF |
| **Data** | Key Vault, encryption at rest |
| **Monitoring** | Security Center, Log Analytics |

### Security Checklist

```
✓ Enable Microsoft Defender for Cloud
✓ Use Managed Identity (no passwords)
✓ Use OIDC for CI/CD (no secrets in pipelines)
✓ Store secrets in Key Vault
✓ Enable private endpoints for databases
✓ Implement NSG rules (deny by default)
✓ Enable WAF for web applications
✓ Run security scans in CI/CD pipeline
✓ Apply Azure Security Benchmark policies
✓ Enable audit logging
```
