# Azure Cost Management

## Overview

Azure Cost Management helps you monitor, allocate, and optimize your cloud spending.

```
┌─────────────────────────────────────────────────────────────┐
│              AZURE COST MANAGEMENT                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    ANALYZE                          │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐        │   │
│  │  │   Cost    │ │  Cost by  │ │  Cost by  │        │   │
│  │  │ Analysis  │ │  Service  │ │  Resource │        │   │
│  │  └───────────┘ └───────────┘ └───────────┘        │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    BUDGET                           │   │
│  │  Set spending limits → Get alerts → Take action     │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   OPTIMIZE                          │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐        │   │
│  │  │ Reserved  │ │  Right-   │ │  Advisor  │        │   │
│  │  │ Instances │ │   Size    │ │   Tips    │        │   │
│  │  └───────────┘ └───────────┘ └───────────┘        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Cost Analysis

### View Costs via CLI

```bash
# View current month costs
az consumption usage list \
  --start-date $(date -d "$(date +%Y-%m-01)" +%Y-%m-%d) \
  --end-date $(date +%Y-%m-%d) \
  --query "[].{Name:instanceName, Cost:pretaxCost, Currency:currency}" \
  -o table

# View costs by resource group
az consumption usage list \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --query "[].{ResourceGroup:instanceId, Cost:pretaxCost}" \
  -o table
```

### Cost Analysis in Portal

```
Azure Portal > Cost Management + Billing > Cost analysis

Views:
- Accumulated costs (daily/monthly trend)
- Cost by resource
- Cost by service
- Cost by resource group
- Cost by tag
```

---

## Budgets

### Create Budget

```bash
# Create monthly budget
az consumption budget create \
  --budget-name "Monthly-Prod-Budget" \
  --amount 1000 \
  --category Cost \
  --time-grain Monthly \
  --start-date 2024-01-01 \
  --end-date 2024-12-31 \
  --resource-group rg-myapp-prod

# Create budget with alerts
az consumption budget create \
  --budget-name "Monthly-Budget" \
  --amount 1000 \
  --category Cost \
  --time-grain Monthly \
  --start-date 2024-01-01 \
  --end-date 2024-12-31 \
  --notifications '{
    "Actual_GreaterThan_80_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["devops@company.com"],
      "contactRoles": ["Owner"]
    },
    "Forecasted_GreaterThan_100_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "contactEmails": ["devops@company.com"],
      "thresholdType": "Forecasted"
    }
  }'
```

### Budget Bicep

```bicep
resource budget 'Microsoft.Consumption/budgets@2023-05-01' = {
  name: 'monthly-budget'
  properties: {
    category: 'Cost'
    amount: 1000
    timeGrain: 'Monthly'
    timePeriod: {
      startDate: '2024-01-01'
      endDate: '2024-12-31'
    }
    notifications: {
      actual80: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 80
        contactEmails: ['devops@company.com']
        thresholdType: 'Actual'
      }
      forecasted100: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        contactEmails: ['devops@company.com']
        thresholdType: 'Forecasted'
      }
    }
  }
}
```

---

## Cost Optimization Strategies

### 1. Right-Size Resources

```bash
# Check Azure Advisor recommendations
az advisor recommendation list \
  --category Cost \
  -o table

# Resize VM
az vm resize \
  --name myVM \
  --resource-group rg-myapp-prod \
  --size Standard_D2s_v3

# Resize App Service Plan
az appservice plan update \
  --name asp-myapp-prod \
  --resource-group rg-myapp-prod \
  --sku B1  # Downgrade from P1v3
```

### 2. Reserved Instances

```
Cost savings: Up to 72% vs pay-as-you-go

Best for:
- Predictable workloads
- VMs running 24/7
- Databases with consistent usage

Purchase via:
Azure Portal > Reservations > Add
```

### 3. Azure Hybrid Benefit

```bash
# Apply Azure Hybrid Benefit to VM (if you have Windows Server licenses)
az vm update \
  --name myVM \
  --resource-group rg-myapp-prod \
  --license-type Windows_Server

# Apply to SQL Database
az sql db update \
  --name mydb \
  --server sql-myapp-prod \
  --resource-group rg-myapp-prod \
  --license-type BasePrice
```

### 4. Spot VMs for AKS

```bash
# Add spot node pool to AKS (up to 90% savings)
az aks nodepool add \
  --name spotpool \
  --cluster-name aks-myapp-prod \
  --resource-group rg-myapp-prod \
  --priority Spot \
  --spot-max-price -1 \
  --eviction-policy Delete \
  --node-count 3
```

### 5. Auto-Shutdown for Dev/Test

```bash
# Enable auto-shutdown for VMs
az vm auto-shutdown \
  --name myVM \
  --resource-group rg-myapp-dev \
  --time 1900

# Scale down App Service at night (via Azure Automation)
```

### 6. Storage Optimization

```bash
# Change storage tier (Hot → Cool)
az storage account update \
  --name stmyapp \
  --resource-group rg-myapp-prod \
  --access-tier Cool

# Enable lifecycle management
az storage account management-policy create \
  --account-name stmyapp \
  --resource-group rg-myapp-prod \
  --policy @lifecycle-policy.json
```

```json
// lifecycle-policy.json
{
  "rules": [
    {
      "name": "move-to-cool",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/", "backups/"]
        }
      }
    }
  ]
}
```

---

## Tagging for Cost Allocation

### Apply Tags

```bash
# Tag resource group
az group update \
  --name rg-myapp-prod \
  --tags Environment=Production Project=MyApp CostCenter=IT-123 Team=Backend

# Tag specific resource
az resource tag \
  --tags Environment=Production Project=MyApp \
  --ids /subscriptions/.../resourceGroups/rg-myapp-prod/providers/Microsoft.Web/sites/app-myapi-prod

# Tag all resources in resource group
az resource list --resource-group rg-myapp-prod --query "[].id" -o tsv | \
  xargs -I {} az resource tag --tags Environment=Production Project=MyApp --ids {}
```

### Enforce Tags with Policy

```bash
# Create policy to require tags
az policy definition create \
  --name "require-cost-center-tag" \
  --display-name "Require CostCenter tag" \
  --mode All \
  --rules '{
    "if": {
      "field": "tags[CostCenter]",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  }'

# Assign policy to subscription
az policy assignment create \
  --name "require-cost-center" \
  --policy "require-cost-center-tag" \
  --scope /subscriptions/<sub-id>
```

---

## Cost Alerts

### Create Cost Alert

```bash
# Create action group
az monitor action-group create \
  --name ag-cost-alerts \
  --resource-group rg-myapp-prod \
  --short-name cost \
  --action email finance finance@company.com

# Alerts are configured with budgets (see Budget section)
```

### Azure Advisor Alerts

```bash
# Enable Advisor digest emails
az advisor configuration update \
  --low-cpu-threshold 5 \
  --exclude "HighAvailability"
```

---

## Cost Monitoring Dashboard

### Key Metrics to Track

| Metric | Description |
|--------|-------------|
| **Daily Spend** | Track against budget |
| **Cost by Service** | Identify expensive services |
| **Cost by Resource** | Find cost outliers |
| **Reserved Utilization** | Ensure reservations are used |
| **Forecast** | Predicted month-end spend |

### Create Cost Dashboard

```
Azure Portal > Dashboard > New dashboard

Add tiles:
1. Cost analysis chart (accumulated costs)
2. Cost by service (pie chart)
3. Budget vs actual
4. Advisor recommendations
5. Resource group costs
```

---

## DevOps Cost Optimization

### App Service Cost Tips

```bash
# Use shared/basic plans for dev/test
az appservice plan create --name asp-dev --sku B1  # Not P1v3

# Use deployment slots wisely (included in Standard+)
# Don't create unnecessary slots

# Enable auto-scale instead of over-provisioning
az monitor autoscale create \
  --name autoscale-myapp \
  --resource-group rg-myapp-prod \
  --min-count 1 \
  --max-count 5 \
  --count 2
```

### AKS Cost Tips

```bash
# Use cluster autoscaler
az aks nodepool update \
  --name nodepool1 \
  --cluster-name aks-myapp \
  --resource-group rg-myapp \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10

# Use spot instances for non-critical workloads
# Right-size node VMs
# Use Virtual Nodes for burst scenarios
```

### Database Cost Tips

```bash
# Use serverless for variable workloads
az sql db create \
  --name mydb \
  --server sql-myapp \
  --resource-group rg-myapp \
  --compute-model Serverless \
  --auto-pause-delay 60 \
  --min-capacity 0.5

# Use Basic tier for dev/test
# Enable auto-tuning
```

---

## Summary

| Strategy | Savings |
|----------|---------|
| **Reserved Instances** | Up to 72% |
| **Spot VMs** | Up to 90% |
| **Hybrid Benefit** | Up to 40% |
| **Right-sizing** | 20-40% |
| **Auto-shutdown** | Variable |
| **Storage tiering** | 50%+ |

### Cost Management Checklist

```
✓ Set up budgets with alerts
✓ Apply tags for cost allocation
✓ Review Advisor recommendations weekly
✓ Use reserved instances for stable workloads
✓ Right-size resources based on utilization
✓ Enable auto-scaling
✓ Use appropriate service tiers per environment
✓ Clean up unused resources
```

### Quick Commands

```bash
# View Advisor cost recommendations
az advisor recommendation list --category Cost -o table

# Create budget
az consumption budget create --budget-name name --amount 1000 --time-grain Monthly

# Apply tags
az group update -n rg --tags CostCenter=123

# Check spending
az consumption usage list --start-date 2024-01-01 --end-date 2024-01-31
```
