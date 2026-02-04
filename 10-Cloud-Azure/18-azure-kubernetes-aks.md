# Azure Kubernetes Service (AKS)

## What is AKS?

Azure Kubernetes Service is a managed Kubernetes service that simplifies deploying, managing, and scaling containerized applications.

```
┌─────────────────────────────────────────────────────────────┐
│                    AKS ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Control Plane (Azure Managed - FREE)               │   │
│  │  ├── API Server                                     │   │
│  │  ├── etcd                                           │   │
│  │  ├── Scheduler                                      │   │
│  │  └── Controller Manager                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Node Pool (You Pay)                                │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  Node   │  │  Node   │  │  Node   │            │   │
│  │  │ ┌─────┐ │  │ ┌─────┐ │  │ ┌─────┐ │            │   │
│  │  │ │ Pod │ │  │ │ Pod │ │  │ │ Pod │ │            │   │
│  │  │ │.NET │ │  │ │.NET │ │  │ │.NET │ │            │   │
│  │  │ └─────┘ │  │ └─────┘ │  │ └─────┘ │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Create AKS Cluster

### Azure CLI

```bash
# Variables
RESOURCE_GROUP="rg-myapp-prod"
CLUSTER_NAME="aks-myapp-prod"
LOCATION="eastus"
ACR_NAME="acrmyapp"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create AKS cluster
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --network-plugin azure \
  --enable-addons monitoring \
  --attach-acr $ACR_NAME

# Get credentials
az aks get-credentials \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP

# Verify connection
kubectl get nodes
```

### Bicep

```bicep
param clusterName string
param location string = resourceGroup().location
param nodeCount int = 3
param nodeVmSize string = 'Standard_D2s_v3'

resource aks 'Microsoft.ContainerService/managedClusters@2023-07-01' = {
  name: clusterName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    dnsPrefix: clusterName
    agentPoolProfiles: [
      {
        name: 'nodepool1'
        count: nodeCount
        vmSize: nodeVmSize
        osType: 'Linux'
        mode: 'System'
        enableAutoScaling: true
        minCount: 2
        maxCount: 5
      }
    ]
    networkProfile: {
      networkPlugin: 'azure'
      loadBalancerSku: 'standard'
    }
    addonProfiles: {
      omsAgent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalyticsWorkspace.id
        }
      }
    }
  }
}

resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'log-${clusterName}'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

output clusterName string = aks.name
output kubeletIdentity string = aks.properties.identityProfile.kubeletidentity.objectId
```

---

## Connect ACR to AKS

```bash
# Attach existing ACR to AKS
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --attach-acr $ACR_NAME

# Verify ACR connection
az aks check-acr \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --acr $ACR_NAME.azurecr.io
```

---

## Deploy .NET 10 Application

### Dockerfile

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApp.Api/MyApp.Api.csproj", "MyApp.Api/"]
RUN dotnet restore "MyApp.Api/MyApp.Api.csproj"
COPY . .
WORKDIR "/src/MyApp.Api"
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
EXPOSE 8080
USER app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Build and Push to ACR

```bash
# Build and push
az acr build \
  --registry $ACR_NAME \
  --image myapp-api:v1.0 \
  .
```

### Kubernetes Manifests

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp
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
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ASPNETCORE_URLS
              value: "http://+:8080"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp-api
  ports:
    - port: 80
      targetPort: 8080
```

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: tls-secret
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-api
                port:
                  number: 80
```

### Deploy to AKS

```bash
# Apply manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

# Check deployment
kubectl get pods -n myapp
kubectl get services -n myapp
kubectl get ingress -n myapp

# View logs
kubectl logs -l app=myapp-api -n myapp --tail=100

# Scale deployment
kubectl scale deployment myapp-api --replicas=5 -n myapp
```

---

## ConfigMaps and Secrets

### ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Information"
  API_URL: "https://api.myapp.com"
```

### Secret (from Key Vault)

```bash
# Install CSI driver for Key Vault
az aks enable-addons \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --addons azure-keyvault-secrets-provider

# Create SecretProviderClass
```

```yaml
# k8s/secret-provider.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
  namespace: myapp
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "<managed-identity-client-id>"
    keyvaultName: "kv-myapp-prod"
    objects: |
      array:
        - |
          objectName: DatabasePassword
          objectType: secret
        - |
          objectName: ApiKey
          objectType: secret
    tenantId: "<tenant-id>"
  secretObjects:
    - secretName: myapp-secrets
      type: Opaque
      data:
        - objectName: DatabasePassword
          key: db-password
        - objectName: ApiKey
          key: api-key
```

### Use in Deployment

```yaml
# Updated deployment.yaml
spec:
  containers:
    - name: myapp-api
      envFrom:
        - configMapRef:
            name: myapp-config
      env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-password
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: azure-keyvault-secrets
```

---

## Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-api-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## NGINX Ingress Controller

### Install NGINX Ingress

```bash
# Add Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

### Ingress with NGINX

```yaml
# k8s/ingress-nginx.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: myapp-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-api
                port:
                  number: 80
```

---

## Cluster Autoscaler

```bash
# Enable cluster autoscaler on node pool
az aks nodepool update \
  --name nodepool1 \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10
```

---

## Monitoring with Prometheus

### Install Prometheus Stack

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123
```

### Access Grafana

```bash
# Port forward to Grafana
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring

# Open http://localhost:3000
# Login: admin / admin123
```

---

## CI/CD with GitHub Actions

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
  IMAGE_NAME: myapp-api

permissions:
  id-token: write
  contents: read

jobs:
  build-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ github.sha }}
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Build and Push to ACR
        run: |
          az acr build \
            --registry ${{ env.ACR_NAME }} \
            --image ${{ env.IMAGE_NAME }}:${{ github.sha }} \
            .

  deploy:
    needs: build-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS Credentials
        run: |
          az aks get-credentials \
            --name ${{ env.AKS_CLUSTER }} \
            --resource-group ${{ env.RESOURCE_GROUP }}

      - name: Deploy to AKS
        run: |
          kubectl set image deployment/myapp-api \
            myapp-api=${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n myapp

          kubectl rollout status deployment/myapp-api -n myapp
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **AKS Cluster** | Managed Kubernetes |
| **Node Pool** | Worker VMs |
| **ACR** | Container images |
| **Ingress** | External access |
| **HPA** | Pod autoscaling |
| **Cluster Autoscaler** | Node autoscaling |
| **Key Vault CSI** | Secrets from Key Vault |

### Quick Commands

```bash
# Create cluster
az aks create -n aks-name -g rg --node-count 3 --attach-acr acr-name

# Get credentials
az aks get-credentials -n aks-name -g rg

# Deploy
kubectl apply -f k8s/

# Scale
kubectl scale deployment myapp --replicas=5

# Logs
kubectl logs -l app=myapp --tail=100
```
