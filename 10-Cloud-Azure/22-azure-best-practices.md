# Azure Best Practices for DevOps

## Architecture Best Practices

### Well-Architected Framework Pillars

```
┌─────────────────────────────────────────────────────────────┐
│          AZURE WELL-ARCHITECTED FRAMEWORK                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Reliability │  │  Security   │  │    Cost     │        │
│  │             │  │             │  │ Optimization │        │
│  │ • HA/DR     │  │ • Zero Trust│  │ • Right-size │        │
│  │ • Resilience│  │ • Defense   │  │ • Reserved  │        │
│  │ • Recovery  │  │   in depth  │  │ • Automation│        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │ Operational │  │ Performance │                          │
│  │ Excellence  │  │ Efficiency  │                          │
│  │             │  │             │                          │
│  │ • DevOps    │  │ • Scaling   │                          │
│  │ • Monitoring│  │ • Caching   │                          │
│  │ • IaC       │  │ • CDN       │                          │
│  └─────────────┘  └─────────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Naming Conventions

### Resource Naming Pattern

```
<resource-type>-<app-name>-<environment>-<region>-<instance>

Examples:
- rg-myapp-prod-eastus
- app-myapi-prod-eastus
- sql-myapp-prod-eastus
- kv-myapp-prod-eastus
- st-myapp-prod-eastus (storage has length limits)
- aks-myapp-prod-eastus
- acr-myapp-prod (ACR is global, no region)
```

### Resource Type Prefixes

| Resource | Prefix | Example |
|----------|--------|---------|
| Resource Group | `rg-` | rg-myapp-prod |
| App Service | `app-` | app-myapi-prod |
| App Service Plan | `asp-` | asp-myapp-prod |
| Function App | `func-` | func-myapp-prod |
| Storage Account | `st` | stmyappprod |
| SQL Server | `sql-` | sql-myapp-prod |
| SQL Database | `sqldb-` | sqldb-myapp |
| Key Vault | `kv-` | kv-myapp-prod |
| Container Registry | `acr` | acrmyapp |
| AKS Cluster | `aks-` | aks-myapp-prod |
| Virtual Network | `vnet-` | vnet-myapp-prod |
| Subnet | `snet-` | snet-web |
| NSG | `nsg-` | nsg-web |
| Log Analytics | `log-` | log-myapp-prod |
| Application Insights | `ai-` | ai-myapp-prod |

---

## Tagging Strategy

### Required Tags

```bash
# Apply consistent tags
az group update \
  --name rg-myapp-prod \
  --tags \
    Environment=Production \
    Application=MyApp \
    CostCenter=IT-123 \
    Owner=team@company.com \
    ManagedBy=Terraform \
    CreatedDate=$(date +%Y-%m-%d)
```

### Tag Definitions

| Tag | Values | Purpose |
|-----|--------|---------|
| `Environment` | dev, staging, prod | Environment identification |
| `Application` | App name | Cost allocation |
| `CostCenter` | Department code | Billing |
| `Owner` | Email/team | Contact |
| `ManagedBy` | Terraform, Bicep, Manual | IaC tracking |
| `CreatedDate` | YYYY-MM-DD | Lifecycle management |
| `ExpirationDate` | YYYY-MM-DD | Cleanup automation |

---

## Infrastructure as Code

### Terraform Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
├── modules/
│   ├── app-service/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── sql-database/
│   └── key-vault/
└── shared/
    └── naming.tf
```

### Bicep Structure

```
bicep/
├── main.bicep
├── modules/
│   ├── appService.bicep
│   ├── sqlDatabase.bicep
│   └── keyVault.bicep
└── parameters/
    ├── dev.bicepparam
    ├── staging.bicepparam
    └── prod.bicepparam
```

### IaC Best Practices

```hcl
# ✓ Use remote state with locking
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "myapp/prod/terraform.tfstate"
  }
}

# ✓ Pin provider versions
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

# ✓ Use modules for reusability
module "app_service" {
  source = "../modules/app-service"

  name        = "app-myapi-prod"
  environment = "prod"
}
```

---

## CI/CD Best Practices

### Pipeline Structure

```yaml
# Multi-stage pipeline
stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - script: dotnet build
          - script: dotnet test
          - publish: artifact

  - stage: DeployDev
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: development

  - stage: DeployStaging
    dependsOn: DeployDev
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: staging

  - stage: DeployProd
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: production  # Requires approval
```

### CI/CD Checklist

```
Build Stage:
✓ Restore dependencies
✓ Build application
✓ Run unit tests
✓ Run integration tests
✓ Security scanning (SAST, dependencies)
✓ Code coverage
✓ Publish artifacts

Deploy Stage:
✓ Use deployment slots for zero-downtime
✓ Run health checks after deployment
✓ Implement rollback strategy
✓ Use environment-specific configurations
✓ Require approvals for production
```

---

## Security Best Practices

### Identity & Access

```bash
# ✓ Use Managed Identity
az webapp identity assign --name app-myapi-prod --resource-group rg-myapp-prod

# ✓ Use OIDC for CI/CD (no secrets)
# Configure federated credentials in Azure AD

# ✓ Apply least privilege RBAC
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope $KEYVAULT_ID
```

### Secrets Management

```bash
# ✓ Use Key Vault for all secrets
az keyvault secret set --vault-name kv-myapp --name DbPassword --value "..."

# ✓ Use Key Vault references in App Service
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --settings "DbPassword=@Microsoft.KeyVault(SecretUri=https://kv-myapp.vault.azure.net/secrets/DbPassword/)"

# ✗ Never commit secrets to source control
# ✗ Never hardcode secrets in application code
```

### Network Security

```bash
# ✓ Use private endpoints for databases
# ✓ Use VNet integration for App Service
# ✓ Implement NSG rules
# ✓ Enable WAF for public-facing apps
# ✓ Use TLS 1.2 minimum
```

---

## Monitoring Best Practices

### Enable Comprehensive Monitoring

```bash
# Application Insights for APM
az monitor app-insights component create \
  --app ai-myapp-prod \
  --location eastus \
  --resource-group rg-myapp-prod

# Log Analytics for centralized logs
az monitor log-analytics workspace create \
  --workspace-name log-myapp-prod \
  --resource-group rg-myapp-prod

# Enable diagnostics for all resources
az monitor diagnostic-settings create \
  --name "send-to-log-analytics" \
  --resource $RESOURCE_ID \
  --workspace $WORKSPACE_ID \
  --logs '[{"category": "AllLogs", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

### Alerting Strategy

```bash
# Create alerts for critical metrics
az monitor metrics alert create \
  --name "High Error Rate" \
  --resource-group rg-myapp-prod \
  --scopes $APP_SERVICE_ID \
  --condition "count requests/failed > 10" \
  --action $ACTION_GROUP_ID

az monitor metrics alert create \
  --name "High Response Time" \
  --resource-group rg-myapp-prod \
  --scopes $APP_SERVICE_ID \
  --condition "avg HttpResponseTime > 5" \
  --action $ACTION_GROUP_ID
```

---

## Cost Optimization

### Right-Sizing Guidelines

| Environment | App Service | SQL Database | AKS Nodes |
|-------------|-------------|--------------|-----------|
| Development | B1 | Basic | 1 node (Standard_B2s) |
| Staging | S1 | S1 | 2 nodes (Standard_D2s_v3) |
| Production | P1v3+ | S3+ | 3+ nodes (Standard_D4s_v3) |

### Cost-Saving Strategies

```bash
# ✓ Use auto-scaling
az monitor autoscale create \
  --name autoscale-myapp \
  --resource-group rg-myapp-prod \
  --min-count 2 --max-count 10

# ✓ Use reserved instances for stable workloads
# ✓ Use spot instances for AKS non-critical workloads
# ✓ Enable auto-shutdown for dev/test VMs
# ✓ Use storage lifecycle policies
# ✓ Delete unused resources
```

---

## Disaster Recovery

### Backup Strategy

```bash
# App Service backup
az webapp config backup update \
  --resource-group rg-myapp-prod \
  --webapp-name app-myapi-prod \
  --backup-name daily-backup \
  --container-url "https://stbackup.blob.core.windows.net/backups" \
  --frequency 1d

# SQL Database (automatic)
# - Point-in-time restore: 7-35 days
# - Long-term retention: up to 10 years

# Configure geo-redundancy
az sql db update \
  --name mydb \
  --server sql-myapp-prod \
  --resource-group rg-myapp-prod \
  --backup-storage-redundancy Geo
```

### High Availability Pattern

```
┌─────────────────────────────────────────────────────────────┐
│              HIGH AVAILABILITY ARCHITECTURE                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Region: East US (Primary)        Region: West US (DR)      │
│  ┌─────────────────────┐         ┌─────────────────────┐   │
│  │                     │         │                     │   │
│  │  ┌──────────────┐  │         │  ┌──────────────┐  │   │
│  │  │ App Service  │  │         │  │ App Service  │  │   │
│  │  │ (Production) │  │         │  │ (Standby)    │  │   │
│  │  └──────────────┘  │         │  └──────────────┘  │   │
│  │         │          │         │         │          │   │
│  │  ┌──────────────┐  │ Geo-Rep │  ┌──────────────┐  │   │
│  │  │  SQL Server  │──┼─────────┼─▶│  SQL Server  │  │   │
│  │  │  (Primary)   │  │         │  │ (Secondary)  │  │   │
│  │  └──────────────┘  │         │  └──────────────┘  │   │
│  │                     │         │                     │   │
│  └─────────────────────┘         └─────────────────────┘   │
│              │                             │                │
│              └──────────┬──────────────────┘                │
│                         │                                   │
│                  ┌──────────────┐                          │
│                  │Traffic Manager│                          │
│                  │  or Front Door │                         │
│                  └──────────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Environment Parity

### Consistent Environments

```yaml
# Use same IaC for all environments
# Only change parameters

# dev.tfvars
environment         = "dev"
app_service_sku     = "B1"
sql_sku            = "Basic"
replica_count      = 1

# prod.tfvars
environment         = "prod"
app_service_sku     = "P1v3"
sql_sku            = "S3"
replica_count      = 3
```

---

## Documentation

### Required Documentation

```
docs/
├── architecture/
│   ├── overview.md
│   ├── diagrams/
│   └── decisions/
│       └── ADR-001-database-choice.md
├── operations/
│   ├── runbook.md
│   ├── incident-response.md
│   └── disaster-recovery.md
├── deployment/
│   ├── pipeline.md
│   └── rollback.md
└── security/
    └── compliance.md
```

---

## Summary Checklist

### Project Setup
- [ ] Define naming conventions
- [ ] Create tagging strategy
- [ ] Set up IaC repository
- [ ] Configure remote state

### CI/CD
- [ ] Multi-stage pipelines
- [ ] Security scanning in CI
- [ ] Deployment slots
- [ ] Health checks
- [ ] Approval gates

### Security
- [ ] Managed Identity enabled
- [ ] OIDC for CI/CD
- [ ] Secrets in Key Vault
- [ ] Private endpoints
- [ ] Network security (NSG/WAF)

### Monitoring
- [ ] Application Insights
- [ ] Log Analytics
- [ ] Alerts configured
- [ ] Dashboards created

### Cost
- [ ] Budgets set
- [ ] Tags for allocation
- [ ] Auto-scaling enabled
- [ ] Right-sized resources

### DR/HA
- [ ] Backups configured
- [ ] Geo-redundancy enabled
- [ ] Recovery tested
- [ ] Runbook documented

---

## Quick Reference Commands

```bash
# Resource Group
az group create -n rg-myapp-prod -l eastus --tags Environment=prod

# App Service
az webapp create -n app-myapi-prod -g rg-myapp-prod -p asp-myapp-prod --runtime "DOTNET|8.0"
az webapp identity assign -n app-myapi-prod -g rg-myapp-prod

# Key Vault
az keyvault create -n kv-myapp-prod -g rg-myapp-prod --enable-rbac-authorization
az keyvault secret set --vault-name kv-myapp-prod -n SecretName --value "value"

# SQL
az sql server create -n sql-myapp-prod -g rg-myapp-prod -l eastus -u admin -p Password
az sql db create -n mydb -s sql-myapp-prod -g rg-myapp-prod --service-objective S1

# Monitoring
az monitor app-insights component create --app ai-myapp-prod -g rg-myapp-prod --application-type web
az monitor log-analytics workspace create --workspace-name log-myapp-prod -g rg-myapp-prod
```
