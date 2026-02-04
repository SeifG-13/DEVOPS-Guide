# GitHub Actions for Azure

## Overview

Deploy .NET 10 and Angular applications to Azure using GitHub Actions with OIDC authentication.

```
┌─────────────────────────────────────────────────────────────┐
│              GITHUB ACTIONS + AZURE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  GitHub Repository                                  │   │
│  │  ├── .github/workflows/                             │   │
│  │  │   ├── deploy-api.yml                            │   │
│  │  │   └── deploy-angular.yml                        │   │
│  │  ├── src/                                          │   │
│  │  └── ...                                           │   │
│  └─────────────────────────────────────────────────────┘   │
│         │                                                   │
│         │ Push / PR                                         │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  GitHub Actions Runner                              │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  Build  │─▶│  Test   │─▶│ Deploy  │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│         │                                                   │
│         │ OIDC / Service Principal                         │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Azure                                              │   │
│  │  ├── App Service (.NET API)                        │   │
│  │  ├── Static Web Apps (Angular)                     │   │
│  │  └── Container Registry                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Authentication Methods

### Option 1: OIDC (Recommended - No Secrets)

```bash
# Create app registration
az ad app create --display-name "github-actions-myapp"

APP_ID=$(az ad app list --display-name "github-actions-myapp" --query "[0].appId" -o tsv)

# Create service principal
az ad sp create --id $APP_ID

# Add federated credential for GitHub
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "description": "GitHub Actions main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# For pull requests
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-pr",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:pull_request",
    "description": "GitHub Actions pull requests",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Grant permissions to Azure resources
SP_ID=$(az ad sp show --id $APP_ID --query "id" -o tsv)
az role assignment create \
  --role Contributor \
  --assignee $SP_ID \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod
```

### Option 2: Service Principal with Secret

```bash
# Create service principal with secret
az ad sp create-for-rbac \
  --name "github-actions-myapp" \
  --role Contributor \
  --scopes /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod \
  --json-auth

# Output (save as AZURE_CREDENTIALS secret):
# {
#   "clientId": "xxx",
#   "clientSecret": "xxx",
#   "subscriptionId": "xxx",
#   "tenantId": "xxx"
# }
```

---

## GitHub Secrets Setup

### For OIDC

```
Repository Settings > Secrets and variables > Actions

Secrets:
- AZURE_CLIENT_ID: <app-id>
- AZURE_TENANT_ID: <tenant-id>
- AZURE_SUBSCRIPTION_ID: <subscription-id>
```

### For Service Principal

```
Secrets:
- AZURE_CREDENTIALS: {"clientId":"...","clientSecret":"...","subscriptionId":"...","tenantId":"..."}
```

---

## .NET 10 API Deployment

### Complete Workflow

```yaml
# .github/workflows/deploy-api.yml
name: Deploy .NET API to Azure

on:
  push:
    branches: [main]
    paths:
      - 'src/MyApp.Api/**'
      - '.github/workflows/deploy-api.yml'
  pull_request:
    branches: [main]
    paths:
      - 'src/MyApp.Api/**'

env:
  AZURE_WEBAPP_NAME: app-myapi-prod
  DOTNET_VERSION: '10.0.x'
  PROJECT_PATH: 'src/MyApp.Api/MyApp.Api.csproj'
  TEST_PROJECT_PATH: 'tests/MyApp.Api.Tests/MyApp.Api.Tests.csproj'

permissions:
  id-token: write   # Required for OIDC
  contents: read
  pull-requests: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test ${{ env.TEST_PROJECT_PATH }} --configuration Release --no-build --verbosity normal --collect:"XPlat Code Coverage" --results-directory ./coverage

      - name: Code Coverage Report
        uses: irongut/CodeCoverageSummary@v1.3.0
        with:
          filename: coverage/**/coverage.cobertura.xml
          badge: true
          format: markdown
          output: both

      - name: Add Coverage PR Comment
        uses: marocchino/sticky-pull-request-comment@v2
        if: github.event_name == 'pull_request'
        with:
          recreate: true
          path: code-coverage-results.md

      - name: Publish
        run: dotnet publish ${{ env.PROJECT_PATH }} --configuration Release --output ./publish

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: api-package
          path: ./publish

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://${{ env.AZURE_WEBAPP_NAME }}-staging.azurewebsites.net

    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: api-package
          path: ./publish

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Staging Slot
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          slot-name: staging
          package: ./publish

      - name: Health Check
        run: |
          sleep 30
          curl --fail https://${{ env.AZURE_WEBAPP_NAME }}-staging.azurewebsites.net/health || exit 1

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://${{ env.AZURE_WEBAPP_NAME }}.azurewebsites.net

    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Swap Staging to Production
        run: |
          az webapp deployment slot swap \
            --name ${{ env.AZURE_WEBAPP_NAME }} \
            --resource-group rg-myapp-prod \
            --slot staging \
            --target-slot production

      - name: Health Check
        run: |
          sleep 30
          curl --fail https://${{ env.AZURE_WEBAPP_NAME }}.azurewebsites.net/health || exit 1
```

---

## Angular Deployment

### Deploy to Static Web Apps

```yaml
# .github/workflows/deploy-angular.yml
name: Deploy Angular to Azure Static Web Apps

on:
  push:
    branches: [main]
    paths:
      - 'src/frontend/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/frontend/**'

env:
  APP_LOCATION: 'src/frontend'
  OUTPUT_LOCATION: 'dist/my-angular-app/browser'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: '${{ env.APP_LOCATION }}/package-lock.json'

      - name: Install dependencies
        working-directory: ${{ env.APP_LOCATION }}
        run: npm ci

      - name: Lint
        working-directory: ${{ env.APP_LOCATION }}
        run: npm run lint

      - name: Test
        working-directory: ${{ env.APP_LOCATION }}
        run: npm run test -- --watch=false --browsers=ChromeHeadless

      - name: Build
        working-directory: ${{ env.APP_LOCATION }}
        run: npm run build -- --configuration=production

      - name: Deploy to Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.SWA_DEPLOYMENT_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: 'upload'
          app_location: '${{ env.APP_LOCATION }}/${{ env.OUTPUT_LOCATION }}'
          skip_app_build: true

  close-pr:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Close PR Preview
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.SWA_DEPLOYMENT_TOKEN }}
          action: 'close'
```

### Deploy to Blob Storage + CDN

```yaml
# .github/workflows/deploy-angular-blob.yml
name: Deploy Angular to Blob Storage + CDN

on:
  push:
    branches: [main]
    paths:
      - 'src/frontend/**'

env:
  STORAGE_ACCOUNT: stmyappprod
  RESOURCE_GROUP: rg-myapp-prod
  CDN_PROFILE: cdn-myapp-prod
  CDN_ENDPOINT: myapp-prod

permissions:
  id-token: write
  contents: read

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

      - name: Install and Build
        run: |
          npm ci
          npm run build -- --configuration=production

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: angular-build
          path: dist/my-angular-app/browser

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: angular-build
          path: ./dist

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Upload to Blob Storage
        run: |
          az storage blob upload-batch \
            --account-name ${{ env.STORAGE_ACCOUNT }} \
            --destination '$web' \
            --source ./dist \
            --overwrite \
            --auth-mode login

      - name: Purge CDN Cache
        run: |
          az cdn endpoint purge \
            --name ${{ env.CDN_ENDPOINT }} \
            --profile-name ${{ env.CDN_PROFILE }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --content-paths "/*"
```

---

## Docker Build and Deploy to ACR

```yaml
# .github/workflows/docker-build.yml
name: Build and Push to ACR

on:
  push:
    branches: [main]
    paths:
      - 'src/MyApp.Api/**'
      - 'Dockerfile'

env:
  ACR_NAME: acrmyapp
  IMAGE_NAME: myapp-api

permissions:
  id-token: write
  contents: read

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Login to ACR
        run: az acr login --name ${{ env.ACR_NAME }}

      - name: Build and Push
        run: |
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker tag ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} \
                     ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:latest
          docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:latest

  deploy:
    needs: build-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to App Service
        run: |
          az webapp config container set \
            --name app-myapi-prod \
            --resource-group rg-myapp-prod \
            --docker-custom-image-name ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --docker-registry-server-url https://${{ env.ACR_NAME }}.azurecr.io
```

---

## Deploy to AKS

```yaml
# .github/workflows/deploy-aks.yml
name: Deploy to AKS

on:
  push:
    branches: [main]

env:
  ACR_NAME: acrmyapp
  AKS_CLUSTER: aks-myapp-prod
  RESOURCE_GROUP: rg-myapp-prod
  NAMESPACE: production

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS Credentials
        run: |
          az aks get-credentials \
            --name ${{ env.AKS_CLUSTER }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --overwrite-existing

      - name: Update Image Tag
        run: |
          sed -i 's|IMAGE_TAG|${{ github.sha }}|g' k8s/deployment.yaml

      - name: Deploy to AKS
        run: |
          kubectl apply -f k8s/ -n ${{ env.NAMESPACE }}
          kubectl rollout status deployment/myapp-api -n ${{ env.NAMESPACE }}
```

---

## Reusable Workflows

### Reusable Build Workflow

```yaml
# .github/workflows/reusable-dotnet-build.yml
name: Reusable .NET Build

on:
  workflow_call:
    inputs:
      dotnet-version:
        required: false
        type: string
        default: '10.0.x'
      project-path:
        required: true
        type: string
    outputs:
      artifact-name:
        description: 'Name of the uploaded artifact'
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: dotnet-build-${{ github.run_id }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ inputs.dotnet-version }}

      - name: Build and Publish
        run: |
          dotnet restore
          dotnet build --configuration Release --no-restore
          dotnet publish ${{ inputs.project-path }} --configuration Release --output ./publish

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: dotnet-build-${{ github.run_id }}
          path: ./publish
```

### Using Reusable Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    uses: ./.github/workflows/reusable-dotnet-build.yml
    with:
      project-path: 'src/MyApp.Api/MyApp.Api.csproj'

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
```

---

## Summary

| Authentication | Use Case |
|----------------|----------|
| **OIDC** | Recommended, no secrets |
| **Service Principal** | Legacy, requires secret rotation |

### Key Actions

| Action | Purpose |
|--------|---------|
| `azure/login@v2` | Authenticate to Azure |
| `azure/webapps-deploy@v3` | Deploy to App Service |
| `Azure/static-web-apps-deploy@v1` | Deploy to Static Web Apps |
| `actions/upload-artifact@v4` | Upload build artifacts |

### Best Practices

```yaml
# ✓ Use OIDC authentication (no secrets)
# ✓ Use environments with approvals
# ✓ Cache dependencies (npm, NuGet)
# ✓ Run tests before deployment
# ✓ Use deployment slots for zero-downtime
# ✓ Add health checks after deployment
# ✓ Use reusable workflows for consistency
```
