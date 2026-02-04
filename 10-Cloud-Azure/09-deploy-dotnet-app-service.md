# Deploy .NET 10 API to Azure App Service

## Overview

This guide covers deploying a .NET 10 Web API to Azure App Service with best practices for production.

```
┌─────────────────────────────────────────────────────────────┐
│           .NET 10 DEPLOYMENT ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │    GitHub    │───▶│    CI/CD     │───▶│ App Service  │ │
│  │    Repo      │    │   Pipeline   │    │  .NET 10     │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│                                                │            │
│                           ┌────────────────────┼───────┐   │
│                           │                    │       │   │
│                           ▼                    ▼       ▼   │
│                    ┌──────────┐         ┌─────────┐┌────┐ │
│                    │Azure SQL │         │Key Vault││ AI │ │
│                    │ Database │         │ Secrets ││    │ │
│                    └──────────┘         └─────────┘└────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Project Setup

### Create .NET 10 Web API

```bash
# Create new Web API project
dotnet new webapi -n MyApp.Api -f net10.0

# Add essential packages
cd MyApp.Api
dotnet add package Azure.Identity
dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
dotnet add package Microsoft.ApplicationInsights.AspNetCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Azure.Security.KeyVault.Secrets
```

### Project Structure

```
MyApp.Api/
├── Controllers/
│   ├── WeatherController.cs
│   └── HealthController.cs
├── Models/
├── Services/
├── Data/
│   └── AppDbContext.cs
├── Program.cs
├── appsettings.json
├── appsettings.Development.json
├── appsettings.Production.json
└── MyApp.Api.csproj
```

---

## Configuration for Azure

### Program.cs

```csharp
using Azure.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add Key Vault configuration in production
if (builder.Environment.IsProduction())
{
    var keyVaultUri = builder.Configuration["KeyVault:Uri"];
    if (!string.IsNullOrEmpty(keyVaultUri))
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUri),
            new DefaultAzureCredential());
    }
}

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry();

// Add DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Add health checks
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddAzureKeyVault(
        new Uri(builder.Configuration["KeyVault:Uri"]!),
        new DefaultAzureCredential(),
        options => options.AddSecret("DatabasePassword"));

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

### appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": ""
  },
  "KeyVault": {
    "Uri": ""
  }
}
```

### appsettings.Production.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "KeyVault": {
    "Uri": "https://kv-myapp-prod.vault.azure.net/"
  }
}
```

### Health Check Controller

```csharp
using Microsoft.AspNetCore.Mvc;

namespace MyApp.Api.Controllers;

[ApiController]
[Route("[controller]")]
public class HealthController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok(new { status = "healthy", timestamp = DateTime.UtcNow });

    [HttpGet("ready")]
    public IActionResult Ready() => Ok(new { status = "ready" });
}
```

---

## Azure Infrastructure Setup

### Create Resources with Azure CLI

```bash
# Variables
RESOURCE_GROUP="rg-myapp-prod"
LOCATION="eastus"
APP_NAME="app-myapi-prod"
PLAN_NAME="asp-myapp-prod"
SQL_SERVER="sql-myapp-prod"
SQL_DB="sqldb-myapp"
KEY_VAULT="kv-myapp-prod"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create App Service Plan
az appservice plan create \
  --name $PLAN_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku P1v3 \
  --is-linux

# Create Web App
az webapp create \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --plan $PLAN_NAME \
  --runtime "DOTNET|8.0"

# Enable managed identity
az webapp identity assign \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP

# Get managed identity principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "principalId" -o tsv)

# Create Key Vault
az keyvault create \
  --name $KEY_VAULT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Grant Key Vault access to App Service
az keyvault set-policy \
  --name $KEY_VAULT \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get list

# Create SQL Server
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user sqladmin \
  --admin-password "$(openssl rand -base64 32)"

# Create SQL Database
az sql db create \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --service-objective S1

# Allow Azure services to access SQL
az sql server firewall-rule create \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

### Store Secrets in Key Vault

```bash
# Store database connection string
az keyvault secret set \
  --vault-name $KEY_VAULT \
  --name "ConnectionStrings--DefaultConnection" \
  --value "Server=tcp:${SQL_SERVER}.database.windows.net,1433;Database=${SQL_DB};Authentication=Active Directory Managed Identity;Encrypt=True;"

# Store other secrets
az keyvault secret set \
  --vault-name $KEY_VAULT \
  --name "ApiSettings--SecretKey" \
  --value "your-secret-key-here"
```

### Configure App Service

```bash
# Set application settings
az webapp config appsettings set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings \
    ASPNETCORE_ENVIRONMENT=Production \
    KeyVault__Uri=https://${KEY_VAULT}.vault.azure.net/

# Configure health check
az webapp config set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --generic-configurations '{"healthCheckPath": "/health"}'

# Enable always on
az webapp config set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --always-on true

# Set minimum TLS version
az webapp config set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --min-tls-version 1.2

# Disable FTP
az webapp config set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --ftps-state Disabled
```

---

## Deployment Methods

### Method 1: ZIP Deploy (CLI)

```bash
# Build and publish
dotnet publish -c Release -o ./publish

# Create ZIP
cd publish && zip -r ../app.zip . && cd ..

# Deploy
az webapp deployment source config-zip \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --src app.zip
```

### Method 2: GitHub Actions

```yaml
# .github/workflows/deploy-api.yml
name: Deploy .NET API to Azure

on:
  push:
    branches: [main]
    paths:
      - 'src/MyApp.Api/**'
      - '.github/workflows/deploy-api.yml'

env:
  AZURE_WEBAPP_NAME: app-myapi-prod
  DOTNET_VERSION: '10.0.x'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: dotnet restore src/MyApp.Api/MyApp.Api.csproj

      - name: Build
        run: dotnet build src/MyApp.Api/MyApp.Api.csproj -c Release --no-restore

      - name: Test
        run: dotnet test src/MyApp.Api.Tests/MyApp.Api.Tests.csproj -c Release --no-build

      - name: Publish
        run: dotnet publish src/MyApp.Api/MyApp.Api.csproj -c Release -o ./publish

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: api-package
          path: ./publish

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: production
      url: ${{ steps.deploy.outputs.webapp-url }}

    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: api-package
          path: ./publish

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure Web App
        id: deploy
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          package: ./publish
```

### Method 3: Azure Pipelines

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - src/MyApp.Api/**

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '10.0.x'
  webAppName: 'app-myapi-prod'
  resourceGroup: 'rg-myapp-prod'

stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            inputs:
              version: $(dotnetVersion)

          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: 'restore'
              projects: 'src/MyApp.Api/MyApp.Api.csproj'

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              projects: 'src/MyApp.Api/MyApp.Api.csproj'
              arguments: '--configuration $(buildConfiguration)'

          - task: DotNetCoreCLI@2
            displayName: 'Test'
            inputs:
              command: 'test'
              projects: 'src/MyApp.Api.Tests/MyApp.Api.Tests.csproj'
              arguments: '--configuration $(buildConfiguration)'

          - task: DotNetCoreCLI@2
            displayName: 'Publish'
            inputs:
              command: 'publish'
              projects: 'src/MyApp.Api/MyApp.Api.csproj'
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
              publishWebProjects: false

          - publish: $(Build.ArtifactStagingDirectory)
            artifact: api-package

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: Deploy
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    appType: 'webAppLinux'
                    appName: $(webAppName)
                    package: '$(Pipeline.Workspace)/api-package/*.zip'
```

---

## Database Migrations

### Run Migrations on Deploy

```csharp
// Program.cs - Apply migrations on startup
using Microsoft.EntityFrameworkCore;

var app = builder.Build();

// Apply pending migrations
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}
```

### Migration in CI/CD

```yaml
# In GitHub Actions
- name: Run EF Migrations
  run: |
    dotnet tool install --global dotnet-ef
    dotnet ef database update --project src/MyApp.Api/MyApp.Api.csproj
  env:
    ConnectionStrings__DefaultConnection: ${{ secrets.DB_CONNECTION_STRING }}
```

---

## Deployment Slots for Zero-Downtime

```bash
# Create staging slot
az webapp deployment slot create \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --slot staging

# Copy production settings to staging
az webapp config appsettings set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --slot staging \
  --settings \
    ASPNETCORE_ENVIRONMENT=Staging \
    KeyVault__Uri=https://${KEY_VAULT}.vault.azure.net/

# Deploy to staging first
az webapp deployment source config-zip \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --slot staging \
  --src app.zip

# Test staging slot
curl https://${APP_NAME}-staging.azurewebsites.net/health

# Swap to production
az webapp deployment slot swap \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --slot staging \
  --target-slot production
```

---

## Verify Deployment

```bash
# Check app status
az webapp show \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "state" -o tsv

# Get app URL
APP_URL=$(az webapp show \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "defaultHostName" -o tsv)

# Test health endpoint
curl https://$APP_URL/health

# View logs
az webapp log tail \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP
```

---

## Summary

| Step | Command/Action |
|------|----------------|
| Create resources | `az webapp create`, `az keyvault create`, `az sql server create` |
| Enable managed identity | `az webapp identity assign` |
| Store secrets | `az keyvault secret set` |
| Configure app | `az webapp config appsettings set` |
| Deploy | ZIP deploy, GitHub Actions, or Azure Pipelines |
| Use slots | Deploy to staging, swap to production |
| Monitor | Health checks, Application Insights |

### Quick Deploy Script

```bash
#!/bin/bash
# deploy.sh
dotnet publish -c Release -o ./publish
cd publish && zip -r ../app.zip . && cd ..
az webapp deployment source config-zip \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --src app.zip
echo "Deployed! URL: https://app-myapi-prod.azurewebsites.net"
```
