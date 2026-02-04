# Deploy Angular to Azure Blob Storage + CDN

## Architecture Overview

Deploy Angular as static files to Blob Storage with Azure CDN for global distribution.

```
┌─────────────────────────────────────────────────────────────┐
│         AZURE BLOB STORAGE + CDN ARCHITECTURE                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Users (Global)                                             │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Azure CDN (Edge Locations)             │   │
│  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐     │   │
│  │  │ NYC │  │ LON │  │ TOK │  │ SYD │  │ ... │     │   │
│  │  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘     │   │
│  └─────────────────────────┬───────────────────────────┘   │
│                            │ Cache Miss                     │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           Azure Blob Storage (Origin)               │   │
│  │  ┌───────────────────────────────────────────────┐ │   │
│  │  │  $web container (static website enabled)      │ │   │
│  │  │  ├── index.html                               │ │   │
│  │  │  ├── main.js                                  │ │   │
│  │  │  ├── styles.css                               │ │   │
│  │  │  └── assets/                                  │ │   │
│  │  └───────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Benefits:                                                  │
│  • Low cost storage                                        │
│  • Global CDN distribution                                 │
│  • Custom caching rules                                    │
│  • Custom domains + SSL                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Setup Storage Account

### Create Storage Account

```bash
# Variables
RESOURCE_GROUP="rg-myapp-prod"
STORAGE_ACCOUNT="stmyappprod"  # Must be globally unique, lowercase
LOCATION="eastus"

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --https-only true \
  --min-tls-version TLS1_2

# Enable static website hosting
az storage blob service-properties update \
  --account-name $STORAGE_ACCOUNT \
  --static-website \
  --index-document index.html \
  --404-document index.html

# Get static website URL
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "primaryEndpoints.web" -o tsv
# Output: https://stmyappprod.z13.web.core.windows.net/
```

---

## Build and Upload Angular App

### Build for Production

```bash
# Build Angular app
ng build --configuration=production

# Output is in dist/my-angular-app/browser/
```

### Upload to Blob Storage

```bash
# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "[0].value" -o tsv)

# Upload all files to $web container
az storage blob upload-batch \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --destination '$web' \
  --source dist/my-angular-app/browser/ \
  --overwrite

# Set correct content types
az storage blob upload-batch \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --destination '$web' \
  --source dist/my-angular-app/browser/ \
  --content-type "text/html" \
  --pattern "*.html" \
  --overwrite

az storage blob upload-batch \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --destination '$web' \
  --source dist/my-angular-app/browser/ \
  --content-type "application/javascript" \
  --pattern "*.js" \
  --overwrite

az storage blob upload-batch \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --destination '$web' \
  --source dist/my-angular-app/browser/ \
  --content-type "text/css" \
  --pattern "*.css" \
  --overwrite
```

### Upload Script

```bash
#!/bin/bash
# deploy-angular.sh

STORAGE_ACCOUNT="stmyappprod"
RESOURCE_GROUP="rg-myapp-prod"
BUILD_DIR="dist/my-angular-app/browser"

# Build
echo "Building Angular app..."
ng build --configuration=production

# Get storage key
STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "[0].value" -o tsv)

# Clear existing files
echo "Clearing existing files..."
az storage blob delete-batch \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --source '$web'

# Upload new files
echo "Uploading files..."
az storage blob upload-batch \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --destination '$web' \
  --source $BUILD_DIR \
  --overwrite

echo "Deployment complete!"
echo "URL: https://${STORAGE_ACCOUNT}.z13.web.core.windows.net/"
```

---

## Setup Azure CDN

### Create CDN Profile and Endpoint

```bash
# Create CDN profile
az cdn profile create \
  --name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --sku Standard_Microsoft

# Get static website hostname
ORIGIN_HOST="${STORAGE_ACCOUNT}.z13.web.core.windows.net"

# Create CDN endpoint
az cdn endpoint create \
  --name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --origin $ORIGIN_HOST \
  --origin-host-header $ORIGIN_HOST \
  --enable-compression true \
  --query-string-caching-behavior IgnoreQueryString

# Get CDN endpoint URL
az cdn endpoint show \
  --name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --query "hostName" -o tsv
# Output: myapp-prod.azureedge.net
```

### Configure Caching Rules

```bash
# Set caching rules for different file types
az cdn endpoint rule add \
  --name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --order 1 \
  --rule-name "CacheStaticAssets" \
  --match-variable UrlFileExtension \
  --operator Contains \
  --match-values js css png jpg jpeg gif svg woff woff2 \
  --action-name CacheExpiration \
  --cache-behavior Override \
  --cache-duration "7.00:00:00"

# Disable caching for index.html
az cdn endpoint rule add \
  --name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --order 2 \
  --rule-name "NoCache-IndexHtml" \
  --match-variable UrlFileName \
  --operator Equal \
  --match-values "index.html" \
  --action-name CacheExpiration \
  --cache-behavior BypassCache
```

### Purge CDN Cache

```bash
# Purge all content
az cdn endpoint purge \
  --name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --content-paths "/*"

# Purge specific paths
az cdn endpoint purge \
  --name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --content-paths "/index.html" "/main.js"
```

---

## Custom Domain with SSL

### Add Custom Domain

```bash
# Add custom domain
az cdn custom-domain create \
  --name app-mycompany-com \
  --endpoint-name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --hostname app.mycompany.com
```

### DNS Configuration

```
# CNAME record:
app.mycompany.com  CNAME  myapp-prod.azureedge.net

# For cdnverify (validation):
cdnverify.app.mycompany.com  CNAME  cdnverify.myapp-prod.azureedge.net
```

### Enable HTTPS

```bash
# Enable managed SSL certificate
az cdn custom-domain enable-https \
  --name app-mycompany-com \
  --endpoint-name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP
```

---

## SPA Routing (URL Rewrite)

Angular SPAs need all routes to return `index.html`.

### Configure URL Rewrite Rule

```bash
# Create URL rewrite rule for SPA routing
az cdn endpoint rule add \
  --name myapp-prod \
  --profile-name cdn-myapp-prod \
  --resource-group $RESOURCE_GROUP \
  --order 3 \
  --rule-name "SPA-Routing" \
  --match-variable UrlFileExtension \
  --operator LessThan \
  --match-values 1 \
  --action-name UrlRewrite \
  --source-pattern "/" \
  --destination "/index.html" \
  --preserve-unmatched-path false
```

### Alternative: Use Standard Rules Engine

For more complex scenarios, use the Azure portal's Standard Rules Engine to configure:

1. **Condition**: URL Path does not match pattern `/assets/*` AND does not contain `.`
2. **Action**: URL Rewrite to `/index.html`

---

## GitHub Actions Deployment

```yaml
# .github/workflows/deploy-angular-blob.yml
name: Deploy Angular to Blob Storage + CDN

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'angular.json'
      - 'package.json'

env:
  STORAGE_ACCOUNT: stmyappprod
  RESOURCE_GROUP: rg-myapp-prod
  CDN_PROFILE: cdn-myapp-prod
  CDN_ENDPOINT: myapp-prod

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build -- --configuration=production

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: angular-build
          path: dist/my-angular-app/browser

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: angular-build
          path: ./dist

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Upload to Blob Storage
        run: |
          az storage blob upload-batch \
            --account-name ${{ env.STORAGE_ACCOUNT }} \
            --destination '$web' \
            --source ./dist \
            --overwrite

      - name: Purge CDN
        run: |
          az cdn endpoint purge \
            --name ${{ env.CDN_ENDPOINT }} \
            --profile-name ${{ env.CDN_PROFILE }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --content-paths "/*"
```

---

## Azure Pipelines Deployment

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - src/**

variables:
  storageAccount: 'stmyappprod'
  resourceGroup: 'rg-myapp-prod'
  cdnProfile: 'cdn-myapp-prod'
  cdnEndpoint: 'myapp-prod'

stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'

          - script: npm ci
            displayName: 'Install dependencies'

          - script: npm run build -- --configuration=production
            displayName: 'Build'

          - publish: dist/my-angular-app/browser
            artifact: angular-build

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
                - download: current
                  artifact: angular-build

                - task: AzureCLI@2
                  displayName: 'Upload to Blob Storage'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az storage blob upload-batch \
                        --account-name $(storageAccount) \
                        --destination '$web' \
                        --source $(Pipeline.Workspace)/angular-build \
                        --overwrite

                - task: AzureCLI@2
                  displayName: 'Purge CDN'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az cdn endpoint purge \
                        --name $(cdnEndpoint) \
                        --profile-name $(cdnProfile) \
                        --resource-group $(resourceGroup) \
                        --content-paths "/*"
```

---

## Cost Comparison

| Approach | Monthly Cost (Est.) | Best For |
|----------|---------------------|----------|
| **Static Web Apps (Free)** | $0 | Small projects |
| **Static Web Apps (Standard)** | ~$9 | Production with SLA |
| **Blob + CDN** | ~$1-5 | High traffic, cost-sensitive |
| **App Service** | ~$13+ | Dynamic content |

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Storage Account** | Host static files |
| **$web container** | Static website container |
| **Azure CDN** | Global distribution + caching |
| **Custom Domain** | Your domain name |
| **CDN Rules** | Caching + SPA routing |

### Quick Commands

```bash
# Create storage with static website
az storage account create -n stname -g rg --sku Standard_LRS
az storage blob service-properties update --account-name stname --static-website --index-document index.html

# Upload files
az storage blob upload-batch --account-name stname --destination '$web' --source dist/

# Create CDN
az cdn profile create -n cdn-profile -g rg --sku Standard_Microsoft
az cdn endpoint create -n endpoint -g rg --profile-name cdn-profile --origin stname.z13.web.core.windows.net

# Purge CDN
az cdn endpoint purge -n endpoint --profile-name cdn-profile -g rg --content-paths "/*"
```
