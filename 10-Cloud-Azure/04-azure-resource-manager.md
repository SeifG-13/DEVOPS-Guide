# Azure Resource Manager (ARM)

## What is ARM?

Azure Resource Manager is the deployment and management service for Azure. It provides a consistent management layer for creating, updating, and deleting resources.

```
┌─────────────────────────────────────────────────────────────┐
│              AZURE RESOURCE MANAGER                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ Portal  │ │   CLI   │ │PowerShell│ │   SDK   │          │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
│       │           │           │           │                 │
│       └───────────┴─────┬─────┴───────────┘                 │
│                         ▼                                   │
│              ┌─────────────────────┐                       │
│              │  Azure Resource     │                       │
│              │     Manager         │                       │
│              │  ┌───────────────┐  │                       │
│              │  │ Authentication│  │                       │
│              │  │ Authorization │  │                       │
│              │  │ Request       │  │                       │
│              │  │ Processing    │  │                       │
│              │  └───────────────┘  │                       │
│              └─────────┬───────────┘                       │
│                        │                                    │
│       ┌────────┬───────┼───────┬────────┐                  │
│       ▼        ▼       ▼       ▼        ▼                  │
│   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐           │
│   │ VMs  │ │  SQL │ │ Web  │ │  KV  │ │  ACR │           │
│   └──────┘ └──────┘ └──────┘ └──────┘ └──────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## ARM Templates

### What are ARM Templates?

ARM templates are JSON files that define the infrastructure and configuration for your Azure resources (Infrastructure as Code).

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": { },
  "variables": { },
  "resources": [ ],
  "outputs": { }
}
```

### Template Structure

```
┌─────────────────────────────────────────────────────────────┐
│                  ARM TEMPLATE STRUCTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  $schema        → Template schema version                   │
│  contentVersion → Your template version                     │
│  parameters     → Input values at deployment                │
│  variables      → Reusable values within template           │
│  resources      → Azure resources to deploy                 │
│  outputs        → Values returned after deployment          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Basic ARM Template Example

### Storage Account Template

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "minLength": 3,
      "maxLength": 24,
      "metadata": {
        "description": "Name of the storage account"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for the storage account"
      }
    },
    "sku": {
      "type": "string",
      "defaultValue": "Standard_LRS",
      "allowedValues": [
        "Standard_LRS",
        "Standard_GRS",
        "Standard_ZRS",
        "Premium_LRS"
      ]
    }
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[parameters('sku')]"
      },
      "kind": "StorageV2",
      "properties": {
        "supportsHttpsTrafficOnly": true,
        "minimumTlsVersion": "TLS1_2"
      }
    }
  ],
  "outputs": {
    "storageAccountId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    },
    "primaryEndpoint": {
      "type": "string",
      "value": "[reference(parameters('storageAccountName')).primaryEndpoints.blob]"
    }
  }
}
```

### Deploy ARM Template

```bash
# Create resource group
az group create --name rg-arm-demo --location eastus

# Deploy template
az deployment group create \
  --resource-group rg-arm-demo \
  --template-file template.json \
  --parameters storageAccountName=stmydemoaccount

# Deploy with parameter file
az deployment group create \
  --resource-group rg-arm-demo \
  --template-file template.json \
  --parameters @parameters.json

# What-if (preview changes)
az deployment group what-if \
  --resource-group rg-arm-demo \
  --template-file template.json \
  --parameters storageAccountName=stmydemoaccount
```

### Parameters File

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "stmydemoaccount"
    },
    "sku": {
      "value": "Standard_GRS"
    }
  }
}
```

---

## Web App ARM Template

### App Service with .NET 10

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": {
      "type": "string",
      "metadata": {
        "description": "Name of the web app"
      }
    },
    "environment": {
      "type": "string",
      "allowedValues": ["dev", "staging", "prod"],
      "defaultValue": "dev"
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "variables": {
    "appServicePlanName": "[concat('asp-', parameters('appName'), '-', parameters('environment'))]",
    "webAppName": "[concat('app-', parameters('appName'), '-', parameters('environment'))]",
    "appInsightsName": "[concat('ai-', parameters('appName'), '-', parameters('environment'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2022-09-01",
      "name": "[variables('appServicePlanName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[if(equals(parameters('environment'), 'prod'), 'P1v3', 'B1')]"
      },
      "kind": "linux",
      "properties": {
        "reserved": true
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2022-09-01",
      "name": "[variables('webAppName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]",
        "[resourceId('Microsoft.Insights/components', variables('appInsightsName'))]"
      ],
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]",
        "siteConfig": {
          "linuxFxVersion": "DOTNETCORE|8.0",
          "alwaysOn": "[if(equals(parameters('environment'), 'prod'), true(), false())]",
          "appSettings": [
            {
              "name": "ASPNETCORE_ENVIRONMENT",
              "value": "[parameters('environment')]"
            },
            {
              "name": "APPLICATIONINSIGHTS_CONNECTION_STRING",
              "value": "[reference(resourceId('Microsoft.Insights/components', variables('appInsightsName'))).ConnectionString]"
            }
          ]
        },
        "httpsOnly": true
      }
    },
    {
      "type": "Microsoft.Insights/components",
      "apiVersion": "2020-02-02",
      "name": "[variables('appInsightsName')]",
      "location": "[parameters('location')]",
      "kind": "web",
      "properties": {
        "Application_Type": "web"
      }
    }
  ],
  "outputs": {
    "webAppUrl": {
      "type": "string",
      "value": "[concat('https://', reference(variables('webAppName')).defaultHostName)]"
    },
    "appInsightsKey": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.Insights/components', variables('appInsightsName'))).InstrumentationKey]"
    }
  }
}
```

---

## Azure Bicep

### What is Bicep?

Bicep is a domain-specific language (DSL) for deploying Azure resources. It's simpler than ARM JSON templates.

```
┌─────────────────────────────────────────────────────────────┐
│                   ARM JSON vs BICEP                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ARM JSON                          Bicep                    │
│  ┌─────────────────────────┐      ┌────────────────────┐   │
│  │ {                       │      │ param location     │   │
│  │   "parameters": {       │      │   string =         │   │
│  │     "location": {       │  ──▶ │   resourceGroup()  │   │
│  │       "type": "string", │      │     .location      │   │
│  │       "defaultValue":   │      │                    │   │
│  │         "[resourceGroup │      │ resource sa '...'= │   │
│  │           ().location]" │      │ {                  │   │
│  │     }                   │      │   name: 'mysa'     │   │
│  │   }                     │      │   location: loc    │   │
│  │ }                       │      │ }                  │   │
│  └─────────────────────────┘      └────────────────────┘   │
│                                                             │
│  • Simpler syntax          • Compiles to ARM JSON          │
│  • IntelliSense support    • No state file needed          │
│  • Modules support         • First-class Azure support     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Install Bicep

```bash
# Included with Azure CLI 2.20+
az bicep install

# Upgrade Bicep
az bicep upgrade

# Check version
az bicep version

# Decompile ARM to Bicep
az bicep decompile --file template.json
```

---

## Bicep Examples

### Storage Account (main.bicep)

```bicep
// Parameters
param storageAccountName string
param location string = resourceGroup().location

@allowed([
  'Standard_LRS'
  'Standard_GRS'
  'Standard_ZRS'
])
param sku string = 'Standard_LRS'

// Resource
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: sku
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
  }
}

// Outputs
output storageAccountId string = storageAccount.id
output primaryEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### Web App with SQL (main.bicep)

```bicep
// Parameters
param appName string
param environment string = 'dev'
param location string = resourceGroup().location
param sqlAdminPassword string

// Variables
var appServicePlanName = 'asp-${appName}-${environment}'
var webAppName = 'app-${appName}-${environment}'
var sqlServerName = 'sql-${appName}-${environment}'
var sqlDbName = 'sqldb-${appName}'

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: environment == 'prod' ? 'P1v3' : 'B1'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// Web App
resource webApp 'Microsoft.Web/sites@2022-09-01' = {
  name: webAppName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: environment == 'prod'
      connectionStrings: [
        {
          name: 'DefaultConnection'
          connectionString: 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Database=${sqlDbName};'
          type: 'SQLAzure'
        }
      ]
    }
    httpsOnly: true
  }
}

// SQL Server
resource sqlServer 'Microsoft.Sql/servers@2022-05-01-preview' = {
  name: sqlServerName
  location: location
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
    version: '12.0'
  }
}

// SQL Database
resource sqlDatabase 'Microsoft.Sql/servers/databases@2022-05-01-preview' = {
  parent: sqlServer
  name: sqlDbName
  location: location
  sku: {
    name: environment == 'prod' ? 'S1' : 'Basic'
  }
}

// Allow Azure services
resource sqlFirewall 'Microsoft.Sql/servers/firewallRules@2022-05-01-preview' = {
  parent: sqlServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

// Outputs
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
```

### Deploy Bicep

```bash
# Deploy Bicep file
az deployment group create \
  --resource-group rg-myapp-dev \
  --template-file main.bicep \
  --parameters appName=myapp environment=dev

# Deploy with parameter file
az deployment group create \
  --resource-group rg-myapp-dev \
  --template-file main.bicep \
  --parameters @parameters.json

# What-if
az deployment group what-if \
  --resource-group rg-myapp-dev \
  --template-file main.bicep \
  --parameters appName=myapp
```

---

## Bicep Modules

### Module Structure

```
project/
├── main.bicep           # Main template
├── modules/
│   ├── webapp.bicep     # Web app module
│   ├── sql.bicep        # SQL module
│   └── keyvault.bicep   # Key Vault module
└── parameters/
    ├── dev.json
    └── prod.json
```

### webapp.bicep (Module)

```bicep
param appName string
param location string
param appServicePlanId string
param appInsightsConnectionString string = ''

resource webApp 'Microsoft.Web/sites@2022-09-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlanId
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      appSettings: appInsightsConnectionString != '' ? [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsightsConnectionString
        }
      ] : []
    }
    httpsOnly: true
  }
}

output webAppId string = webApp.id
output defaultHostName string = webApp.properties.defaultHostName
```

### main.bicep (Using Modules)

```bicep
param appName string
param environment string = 'dev'
param location string = resourceGroup().location

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: 'asp-${appName}-${environment}'
  location: location
  sku: {
    name: 'B1'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// Use module
module webApp 'modules/webapp.bicep' = {
  name: 'webAppDeployment'
  params: {
    appName: 'app-${appName}-${environment}'
    location: location
    appServicePlanId: appServicePlan.id
  }
}

output webAppUrl string = 'https://${webApp.outputs.defaultHostName}'
```

---

## Resource Providers

### What are Resource Providers?

Resource providers are services that supply Azure resources (e.g., Microsoft.Web provides App Service).

```bash
# List all providers
az provider list --output table

# List registered providers
az provider list --query "[?registrationState=='Registered'].namespace" -o table

# Register a provider
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.ContainerService

# Check registration status
az provider show --namespace Microsoft.ContainerRegistry --query "registrationState"
```

### Common Providers

| Provider | Resources |
|----------|-----------|
| Microsoft.Web | App Service, Functions |
| Microsoft.Sql | Azure SQL |
| Microsoft.Storage | Storage accounts |
| Microsoft.KeyVault | Key Vault |
| Microsoft.ContainerRegistry | ACR |
| Microsoft.ContainerService | AKS |
| Microsoft.Insights | Application Insights |
| Microsoft.Network | VNets, Load Balancers |

---

## ARM Template Functions

### Common Functions

```json
{
  "variables": {
    // String functions
    "lowerName": "[toLower(parameters('name'))]",
    "concatName": "[concat('app-', parameters('name'), '-', parameters('env'))]",
    "subString": "[substring(parameters('name'), 0, 5)]",

    // Resource functions
    "rgLocation": "[resourceGroup().location]",
    "resourceId": "[resourceId('Microsoft.Web/sites', variables('webAppName'))]",

    // Logical functions
    "isProd": "[equals(parameters('env'), 'prod')]",
    "skuName": "[if(variables('isProd'), 'P1v3', 'B1')]",

    // Array functions
    "firstItem": "[first(parameters('myArray'))]",
    "arrayLength": "[length(parameters('myArray'))]"
  }
}
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **ARM** | Azure's deployment & management service |
| **ARM Templates** | JSON-based Infrastructure as Code |
| **Bicep** | Simpler DSL that compiles to ARM |
| **Resource Provider** | Service that supplies resources |
| **Deployment** | Process of creating resources from templates |

### Quick Commands

```bash
# Deploy ARM template
az deployment group create -g rg-name -f template.json -p param=value

# Deploy Bicep
az deployment group create -g rg-name -f main.bicep -p param=value

# What-if (preview)
az deployment group what-if -g rg-name -f main.bicep

# Export existing resources
az group export -n rg-name > template.json

# Convert ARM to Bicep
az bicep decompile -f template.json
```
