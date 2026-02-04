# Azure App Service

## What is App Service?

Azure App Service is a fully managed platform for building, deploying, and scaling web applications.

```
┌─────────────────────────────────────────────────────────────┐
│                  AZURE APP SERVICE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  App Service Plan (Compute)                         │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  • CPU / Memory                             │   │   │
│  │  │  • OS: Linux or Windows                     │   │   │
│  │  │  • Region: East US                          │   │   │
│  │  │  • SKU: B1, P1v3, etc.                      │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │         │                                           │   │
│  │  ┌──────┴──────┬──────────────┬────────────────┐   │   │
│  │  ▼             ▼              ▼                │   │   │
│  │ ┌────────┐  ┌────────┐  ┌────────┐            │   │   │
│  │ │Web App │  │Web App │  │Function│            │   │   │
│  │ │.NET API│  │Angular │  │  App   │            │   │   │
│  │ └────────┘  └────────┘  └────────┘            │   │   │
│  │                                                │   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Features: Custom domains, SSL, Scaling, Deployment slots   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## App Service Plans

### SKU Tiers

| Tier | SKU | Features | Use Case |
|------|-----|----------|----------|
| **Free** | F1 | 60 min/day, no custom domain | Testing |
| **Shared** | D1 | Custom domain, no SSL | Dev |
| **Basic** | B1-B3 | Custom domain, SSL, manual scale | Dev/Test |
| **Standard** | S1-S3 | Auto-scale, slots, backups | Production |
| **Premium** | P1v3-P3v3 | Better perf, more slots, VNet | Production |
| **Isolated** | I1v2-I3v2 | Dedicated environment, VNet | Enterprise |

### Create App Service Plan

```bash
# Create Linux App Service Plan
az appservice plan create \
  --name asp-myapp-prod \
  --resource-group rg-myapp-prod \
  --location eastus \
  --sku P1v3 \
  --is-linux

# Create Windows App Service Plan
az appservice plan create \
  --name asp-myapp-windows \
  --resource-group rg-myapp-prod \
  --location eastus \
  --sku S1

# List plans
az appservice plan list -o table

# Show plan details
az appservice plan show --name asp-myapp-prod --resource-group rg-myapp-prod
```

---

## Creating Web Apps

### For .NET 10 API

```bash
# Create .NET web app on Linux
az webapp create \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --plan asp-myapp-prod \
  --runtime "DOTNET|8.0"

# List available runtimes
az webapp list-runtimes --os linux
az webapp list-runtimes --os windows

# Common .NET runtimes:
# Linux: DOTNET|8.0, DOTNET|7.0, DOTNET|6.0
# Windows: dotnet:8, dotnet:7, dotnet:6
```

### Bicep

```bicep
param appName string
param location string = resourceGroup().location
param appServicePlanId string

resource webApp 'Microsoft.Web/sites@2022-09-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlanId
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: true
      http20Enabled: true
      minTlsVersion: '1.2'
      ftpsState: 'Disabled'
    }
    httpsOnly: true
  }
  identity: {
    type: 'SystemAssigned'
  }
}

output webAppName string = webApp.name
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
output principalId string = webApp.identity.principalId
```

---

## App Configuration

### Application Settings

```bash
# Set single setting
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --settings ASPNETCORE_ENVIRONMENT=Production

# Set multiple settings
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --settings \
    ASPNETCORE_ENVIRONMENT=Production \
    MyApp__ApiKey=@Microsoft.KeyVault(SecretUri=https://kv-myapp.vault.azure.net/secrets/ApiKey/)

# List settings
az webapp config appsettings list \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  -o table

# Delete setting
az webapp config appsettings delete \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --setting-names MyOldSetting
```

### Connection Strings

```bash
# Set connection string
az webapp config connection-string set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --connection-string-type SQLAzure \
  --settings DefaultConnection="Server=tcp:sql-myapp.database.windows.net;Database=mydb;..."

# Key Vault reference
az webapp config connection-string set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --connection-string-type SQLAzure \
  --settings DefaultConnection="@Microsoft.KeyVault(SecretUri=https://kv-myapp.vault.azure.net/secrets/DbConnection/)"
```

### General Configuration

```bash
# Set startup command
az webapp config set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --startup-file "dotnet MyApp.Api.dll"

# Enable always on (keeps app warm)
az webapp config set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --always-on true

# Set min TLS version
az webapp config set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --min-tls-version 1.2

# Disable FTP
az webapp config set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --ftps-state Disabled
```

---

## Deployment Methods

### 1. ZIP Deploy

```bash
# Create deployment package
dotnet publish -c Release -o ./publish
cd publish && zip -r ../app.zip . && cd ..

# Deploy ZIP
az webapp deployment source config-zip \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --src app.zip

# Or using publish profile
curl -X POST \
  -u '$app-myapi-prod:<password>' \
  --data-binary @app.zip \
  https://app-myapi-prod.scm.azurewebsites.net/api/zipdeploy
```

### 2. Git Deploy

```bash
# Configure local Git deployment
az webapp deployment source config-local-git \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod

# Get Git URL
az webapp deployment source config-local-git \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --query "url" -o tsv

# Push to Azure
git remote add azure https://<user>@app-myapi-prod.scm.azurewebsites.net/app-myapi-prod.git
git push azure main
```

### 3. GitHub Actions Deploy

```bash
# Get publish profile
az webapp deployment list-publishing-profiles \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --xml
```

### 4. Container Deploy

```bash
# Configure container deployment
az webapp config container set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --docker-custom-image-name acrmyapp.azurecr.io/myapi:latest \
  --docker-registry-server-url https://acrmyapp.azurecr.io

# Enable continuous deployment
az webapp deployment container config \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --enable-cd true
```

---

## Deployment Slots

```
┌─────────────────────────────────────────────────────────────┐
│                  DEPLOYMENT SLOTS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  app-myapi-prod.azurewebsites.net (Production)             │
│       ▲                                                     │
│       │ SWAP                                                │
│       ▼                                                     │
│  app-myapi-prod-staging.azurewebsites.net (Staging)        │
│                                                             │
│  Benefits:                                                  │
│  • Zero-downtime deployments                               │
│  • Test in production environment                          │
│  • Easy rollback (swap back)                               │
│  • Warm up before swap                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create and Use Slots

```bash
# Create staging slot
az webapp deployment slot create \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --slot staging

# Deploy to staging
az webapp deployment source config-zip \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --slot staging \
  --src app.zip

# List slots
az webapp deployment slot list \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  -o table

# Swap staging to production
az webapp deployment slot swap \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --slot staging \
  --target-slot production

# Swap with preview (two-phase)
az webapp deployment slot swap \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --slot staging \
  --action preview

# Complete or cancel swap
az webapp deployment slot swap \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --slot staging \
  --action swap  # or --action reset
```

### Slot Settings

```bash
# Configure slot-specific settings (sticky)
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --slot staging \
  --slot-settings ASPNETCORE_ENVIRONMENT=Staging

# Mark setting as slot-specific
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --settings MY_SETTING=value \
  --slot-settings MY_SETTING
```

---

## Scaling

### Manual Scaling

```bash
# Scale up (change plan size)
az appservice plan update \
  --name asp-myapp-prod \
  --resource-group rg-myapp-prod \
  --sku P2v3

# Scale out (add instances)
az appservice plan update \
  --name asp-myapp-prod \
  --resource-group rg-myapp-prod \
  --number-of-workers 3
```

### Auto-Scaling

```bash
# Create autoscale rule
az monitor autoscale create \
  --name autoscale-myapp \
  --resource-group rg-myapp-prod \
  --resource /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.Web/serverfarms/asp-myapp-prod \
  --resource-type Microsoft.Web/serverfarms \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Add scale-out rule (CPU > 70%)
az monitor autoscale rule create \
  --autoscale-name autoscale-myapp \
  --resource-group rg-myapp-prod \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 1

# Add scale-in rule (CPU < 30%)
az monitor autoscale rule create \
  --autoscale-name autoscale-myapp \
  --resource-group rg-myapp-prod \
  --condition "CpuPercentage < 30 avg 5m" \
  --scale in 1
```

---

## Custom Domains and SSL

### Add Custom Domain

```bash
# Add custom domain
az webapp config hostname add \
  --webapp-name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --hostname api.myapp.com

# Verify domain ownership
# Add TXT record: asuid.api.myapp.com → <verification-id>
# Add CNAME: api.myapp.com → app-myapi-prod.azurewebsites.net
```

### SSL Certificate

```bash
# Create managed certificate (free)
az webapp config ssl create \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --hostname api.myapp.com

# Bind certificate
az webapp config ssl bind \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --certificate-thumbprint <thumbprint> \
  --ssl-type SNI

# Upload custom certificate
az webapp config ssl upload \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --certificate-file mycert.pfx \
  --certificate-password <password>
```

---

## Logging and Diagnostics

### Enable Logging

```bash
# Enable application logging
az webapp log config \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --application-logging filesystem \
  --level information

# Enable web server logging
az webapp log config \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --web-server-logging filesystem

# Stream logs in real-time
az webapp log tail \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod

# Download logs
az webapp log download \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --log-file logs.zip
```

### Health Check

```bash
# Configure health check
az webapp config set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --generic-configurations '{"healthCheckPath": "/health"}'
```

---

## VNet Integration

```bash
# Create subnet for App Service
az network vnet subnet create \
  --name snet-appservice \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --address-prefix 10.0.4.0/24 \
  --delegations Microsoft.Web/serverFarms

# Enable VNet integration
az webapp vnet-integration add \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --vnet vnet-myapp-prod \
  --subnet snet-appservice

# Route all traffic through VNet
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --settings WEBSITE_VNET_ROUTE_ALL=1
```

---

## Summary

| Feature | Description |
|---------|-------------|
| **App Service Plan** | Compute resources (CPU, memory) |
| **Web App** | Your application |
| **Deployment Slots** | Zero-downtime deployments |
| **Scaling** | Manual or auto-scale |
| **Custom Domains** | Your own domain + SSL |
| **VNet Integration** | Connect to private resources |

### Quick Commands

```bash
# Create plan and app
az appservice plan create -n asp-name -g rg --sku B1 --is-linux
az webapp create -n app-name -g rg -p asp-name --runtime "DOTNET|8.0"

# Deploy
az webapp deployment source config-zip -n app-name -g rg --src app.zip

# Slots
az webapp deployment slot create -n app-name -g rg --slot staging
az webapp deployment slot swap -n app-name -g rg --slot staging

# Logs
az webapp log tail -n app-name -g rg

# Settings
az webapp config appsettings set -n app-name -g rg --settings KEY=value
```
