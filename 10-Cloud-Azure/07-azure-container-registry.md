# Azure Container Registry (ACR)

## What is ACR?

Azure Container Registry is a managed Docker registry service for storing and managing container images.

```
┌─────────────────────────────────────────────────────────────┐
│              AZURE CONTAINER REGISTRY                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────────┐    ┌────────────┐ │
│  │ Developer   │    │      ACR        │    │   Deploy   │ │
│  │             │    │                 │    │            │ │
│  │ Dockerfile  │───▶│ myacr.azurecr.io│───▶│    AKS     │ │
│  │ docker push │    │                 │    │ App Service│ │
│  │             │    │ ┌─────────────┐ │    │    ACI     │ │
│  │ CI/CD       │    │ │ Repository: │ │    │            │ │
│  │ az acr build│    │ │  myapp      │ │    └────────────┘ │
│  └─────────────┘    │ │  - v1.0     │ │                   │
│                     │ │  - v1.1     │ │                   │
│                     │ │  - latest   │ │                   │
│                     │ └─────────────┘ │                   │
│                     └─────────────────┘                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## ACR SKUs

| SKU | Storage | Throughput | Features |
|-----|---------|------------|----------|
| **Basic** | 10 GB | Low | Dev/test |
| **Standard** | 100 GB | Medium | Production |
| **Premium** | 500 GB | High | Geo-replication, private link, content trust |

---

## Creating ACR

### Azure CLI

```bash
# Create ACR
az acr create \
  --name acrmyapp \
  --resource-group rg-myapp-prod \
  --sku Standard \
  --location eastus

# Enable admin account (not recommended for production)
az acr update \
  --name acrmyapp \
  --admin-enabled true

# Get admin credentials
az acr credential show --name acrmyapp

# Get login server
az acr show --name acrmyapp --query "loginServer" -o tsv
# Output: acrmyapp.azurecr.io
```

### Bicep

```bicep
param acrName string
param location string = resourceGroup().location

resource acr 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: acrName
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    adminUserEnabled: false
    policies: {
      quarantinePolicy: {
        status: 'disabled'
      }
      retentionPolicy: {
        days: 30
        status: 'enabled'
      }
    }
  }
}

output acrLoginServer string = acr.properties.loginServer
output acrId string = acr.id
```

---

## Authentication

### Login Methods

```bash
# Method 1: Azure CLI (interactive)
az acr login --name acrmyapp

# Method 2: Docker login with Azure AD token
TOKEN=$(az acr login --name acrmyapp --expose-token --query "accessToken" -o tsv)
docker login acrmyapp.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password $TOKEN

# Method 3: Admin credentials (not recommended)
az acr credential show --name acrmyapp
docker login acrmyapp.azurecr.io -u acrmyapp -p <password>

# Method 4: Service Principal
docker login acrmyapp.azurecr.io -u <sp-app-id> -p <sp-password>
```

### Grant Access with RBAC

```bash
# Get ACR resource ID
ACR_ID=$(az acr show --name acrmyapp --query "id" -o tsv)

# Grant pull access to App Service managed identity
az role assignment create \
  --role AcrPull \
  --assignee <managed-identity-principal-id> \
  --scope $ACR_ID

# Grant push access to CI/CD service principal
az role assignment create \
  --role AcrPush \
  --assignee <sp-app-id> \
  --scope $ACR_ID

# Common ACR roles:
# - AcrPull: Pull images
# - AcrPush: Push and pull images
# - AcrDelete: Delete images
# - Contributor: Full access
```

---

## Building and Pushing Images

### Local Build and Push

```bash
# Login to ACR
az acr login --name acrmyapp

# Build image locally
docker build -t myapp:v1.0 .

# Tag for ACR
docker tag myapp:v1.0 acrmyapp.azurecr.io/myapp:v1.0
docker tag myapp:v1.0 acrmyapp.azurecr.io/myapp:latest

# Push to ACR
docker push acrmyapp.azurecr.io/myapp:v1.0
docker push acrmyapp.azurecr.io/myapp:latest
```

### ACR Tasks (Build in Cloud)

```bash
# Quick build (no local Docker needed)
az acr build \
  --registry acrmyapp \
  --image myapp:v1.0 \
  .

# Build with specific Dockerfile
az acr build \
  --registry acrmyapp \
  --image myapp:v1.0 \
  --file Dockerfile.production \
  .

# Build from Git repo
az acr build \
  --registry acrmyapp \
  --image myapp:v1.0 \
  https://github.com/myorg/myrepo.git

# Multi-platform build
az acr build \
  --registry acrmyapp \
  --image myapp:v1.0 \
  --platform linux/amd64,linux/arm64 \
  .
```

---

## .NET 10 Dockerfile Example

### Dockerfile

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy csproj and restore
COPY ["MyApp.Api/MyApp.Api.csproj", "MyApp.Api/"]
RUN dotnet restore "MyApp.Api/MyApp.Api.csproj"

# Copy everything and build
COPY . .
WORKDIR "/src/MyApp.Api"
RUN dotnet build "MyApp.Api.csproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApp.Api.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
EXPOSE 8080

# Security: Run as non-root
USER app

COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Build and Push

```bash
# Build with ACR Tasks
az acr build \
  --registry acrmyapp \
  --image myapp-api:v1.0 \
  --image myapp-api:latest \
  .

# Verify image
az acr repository show-tags \
  --name acrmyapp \
  --repository myapp-api \
  -o table
```

---

## Managing Images

### List Repositories and Tags

```bash
# List repositories
az acr repository list --name acrmyapp -o table

# Show tags for repository
az acr repository show-tags \
  --name acrmyapp \
  --repository myapp-api \
  -o table

# Show image details
az acr repository show \
  --name acrmyapp \
  --image myapp-api:v1.0

# Show manifest
az acr repository show-manifests \
  --name acrmyapp \
  --repository myapp-api \
  -o table
```

### Delete Images

```bash
# Delete specific tag
az acr repository delete \
  --name acrmyapp \
  --image myapp-api:v1.0 \
  --yes

# Delete repository (all tags)
az acr repository delete \
  --name acrmyapp \
  --repository myapp-api \
  --yes

# Purge old images (keep last 10)
az acr run \
  --registry acrmyapp \
  --cmd "acr purge --filter 'myapp-api:.*' --ago 30d --keep 10" \
  /dev/null
```

---

## CI/CD Integration

### Azure Pipelines

```yaml
# azure-pipelines.yml
trigger:
  - main

variables:
  acrName: 'acrmyapp'
  imageName: 'myapp-api'
  tag: '$(Build.BuildId)'

stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: Docker@2
            displayName: 'Build and Push'
            inputs:
              containerRegistry: 'acr-service-connection'
              repository: '$(imageName)'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(tag)
                latest
```

### GitHub Actions

```yaml
# .github/workflows/build-push.yml
name: Build and Push to ACR

on:
  push:
    branches: [main]

env:
  ACR_NAME: acrmyapp
  IMAGE_NAME: myapp-api

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: ACR Login
        run: az acr login --name ${{ env.ACR_NAME }}

      - name: Build and Push
        run: |
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker tag ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} \
                     ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:latest
          docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:latest

      # OR use ACR Build (no Docker needed)
      - name: ACR Build
        run: |
          az acr build \
            --registry ${{ env.ACR_NAME }} \
            --image ${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --image ${{ env.IMAGE_NAME }}:latest \
            .
```

---

## ACR with AKS

### Enable ACR Integration

```bash
# Attach ACR to AKS (grants AcrPull to AKS managed identity)
az aks update \
  --name aks-myapp-prod \
  --resource-group rg-myapp-prod \
  --attach-acr acrmyapp

# Verify integration
az aks check-acr \
  --name aks-myapp-prod \
  --resource-group rg-myapp-prod \
  --acr acrmyapp.azurecr.io
```

### Kubernetes Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp-api
  template:
    metadata:
      labels:
        app: myapp-api
    spec:
      containers:
        - name: myapp-api
          image: acrmyapp.azurecr.io/myapp-api:v1.0
          ports:
            - containerPort: 8080
          resources:
            limits:
              memory: "256Mi"
              cpu: "500m"
```

---

## ACR with App Service

### Configure App Service

```bash
# Enable managed identity
az webapp identity assign \
  --name app-myapp-prod \
  --resource-group rg-myapp-prod

# Get principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name app-myapp-prod \
  --resource-group rg-myapp-prod \
  --query "principalId" -o tsv)

# Grant ACR pull access
ACR_ID=$(az acr show --name acrmyapp --query "id" -o tsv)
az role assignment create \
  --role AcrPull \
  --assignee $PRINCIPAL_ID \
  --scope $ACR_ID

# Configure App Service to use ACR
az webapp config container set \
  --name app-myapp-prod \
  --resource-group rg-myapp-prod \
  --docker-custom-image-name acrmyapp.azurecr.io/myapp-api:latest \
  --docker-registry-server-url https://acrmyapp.azurecr.io
```

---

## Security Features

### Image Scanning

```bash
# Enable Microsoft Defender for Containers (Premium SKU)
az security pricing create \
  --name Containers \
  --tier Standard

# View scan results
az acr repository show \
  --name acrmyapp \
  --image myapp-api:latest \
  --query "changeableAttributes"
```

### Content Trust (Premium)

```bash
# Enable content trust
az acr config content-trust update \
  --registry acrmyapp \
  --status enabled

# Sign and push image
export DOCKER_CONTENT_TRUST=1
docker push acrmyapp.azurecr.io/myapp-api:v1.0
```

### Private Endpoint

```bash
# Create private endpoint for ACR
az network private-endpoint create \
  --name pe-acr-myapp \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-myapp-prod \
  --subnet snet-private-endpoints \
  --private-connection-resource-id $(az acr show --name acrmyapp --query "id" -o tsv) \
  --group-id registry \
  --connection-name acr-connection

# Disable public access
az acr update \
  --name acrmyapp \
  --public-network-enabled false
```

---

## Summary

| Feature | Description |
|---------|-------------|
| **ACR** | Managed Docker registry |
| **ACR Tasks** | Build images in cloud |
| **SKUs** | Basic, Standard, Premium |
| **Authentication** | Azure AD, Service Principal, Admin |
| **AKS Integration** | `az aks update --attach-acr` |
| **App Service** | Managed Identity + AcrPull role |

### Quick Commands

```bash
# Create ACR
az acr create -n acrmyapp -g rg --sku Standard

# Login
az acr login -n acrmyapp

# Build and push
az acr build -r acrmyapp -t myapp:v1 .

# List images
az acr repository list -n acrmyapp -o table
az acr repository show-tags -n acrmyapp --repository myapp

# Grant access
az role assignment create --role AcrPull --assignee <id> --scope <acr-id>
```
