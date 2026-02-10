# GitOps & ArgoCD Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET GitOps Workflow

### Typical .NET GitOps Flow
```
Developer → Push Code → CI Pipeline → Build Image → Push to Registry
                                                          ↓
                                              Update Image Tag in GitOps Repo
                                                          ↓
ArgoCD ← Detect Change ← GitOps Repository
    ↓
Deploy to Kubernetes
```

---

## Repository Structure for .NET

### Application Repository (Source Code)
```
my-dotnet-app/
├── src/
│   ├── MyApp.Api/
│   │   ├── Controllers/
│   │   ├── Program.cs
│   │   └── MyApp.Api.csproj
│   └── MyApp.Core/
├── tests/
├── Dockerfile
└── .github/
    └── workflows/
        └── ci.yml
```

### GitOps Repository (Kubernetes Manifests)
```
gitops-repo/
├── apps/
│   └── myapp.yaml              # ArgoCD Application
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── deployment-patch.yaml
    ├── staging/
    └── production/
```

---

## Kubernetes Manifests for .NET

### Base Deployment
```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:latest  # Replaced by Kustomize
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              valueFrom:
                configMapKeyRef:
                  name: myapp-config
                  key: environment
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

### Base ConfigMap
```yaml
# base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  environment: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  Logging__LogLevel__Default: "Information"
```

### Base Kustomization
```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

### Production Overlay
```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp-prod

resources:
  - ../../base

images:
  - name: myregistry/myapp
    newTag: "1.2.3"  # Updated by CI pipeline

replicas:
  - name: myapp
    count: 5

patches:
  - path: patches/deployment-patch.yaml

configMapGenerator:
  - name: myapp-config
    behavior: merge
    literals:
      - environment=Production
      - Logging__LogLevel__Default=Warning
```

---

## CI Pipeline for Image Updates

### GitHub Actions - Update GitOps Repo
```yaml
# .github/workflows/ci.yml
name: Build and Update GitOps

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Build and Test
        run: |
          dotnet restore
          dotnet build --no-restore
          dotnet test --no-build

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}

  update-gitops:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: org/gitops-repo
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Update image tag
        run: |
          cd gitops/overlays/production
          kustomize edit set image myregistry/myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}

      - name: Commit and push
        run: |
          cd gitops
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Update myapp to sha-${{ github.sha }}"
          git push
```

---

## ArgoCD Application for .NET

### Application Definition
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  # Health checks for .NET
  ignoreDifferences:
    - group: ""
      kind: ConfigMap
      jsonPointers:
        - /data/appsettings.json  # Ignore runtime config changes
```

---

## Handling Secrets

### Sealed Secrets for .NET
```bash
# Install kubeseal
brew install kubeseal

# Create secret
kubectl create secret generic myapp-secrets \
  --from-literal=ConnectionStrings__Default="Server=..." \
  --from-literal=JWT__Secret="..." \
  --dry-run=client -o yaml > secret.yaml

# Seal it
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

```yaml
# sealed-secret.yaml (safe to commit)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: myapp-secrets
spec:
  encryptedData:
    ConnectionStrings__Default: AgB3...encrypted...
    JWT__Secret: AgC4...encrypted...
```

### External Secrets
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: myapp-secrets
  data:
    - secretKey: ConnectionStrings__Default
      remoteRef:
        key: myapp-db-connection
    - secretKey: JWT__Secret
      remoteRef:
        key: myapp-jwt-secret
```

---

## Interview Q&A

### Q1: How do you deploy a .NET application using GitOps?
**A:**
1. CI pipeline builds Docker image on code push
2. Pipeline updates image tag in GitOps repository
3. ArgoCD detects change and syncs to Kubernetes
4. Application is deployed with new image

### Q2: How do you handle appsettings.json in GitOps?
**A:**
- Use ConfigMaps for non-sensitive settings
- Use Secrets (Sealed/External) for sensitive settings
- Environment variables override appsettings.json
- Mount config files if needed

```csharp
builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddEnvironmentVariables();  // Highest priority
```

### Q3: How do you manage database migrations with GitOps?
**A:**
Options:
1. **Init container**: Run migrations before app starts
2. **Kubernetes Job**: Separate migration job with sync wave
3. **CI pipeline**: Run migrations before updating image tag

```yaml
# Sync wave for migrations
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: PreSync
```

### Q4: How do you handle environment-specific configuration?
**A:**
Use Kustomize overlays:
```yaml
# overlays/production/kustomization.yaml
configMapGenerator:
  - name: myapp-config
    literals:
      - ASPNETCORE_ENVIRONMENT=Production
      - Logging__LogLevel__Default=Warning
```

### Q5: How do you rollback a .NET deployment in GitOps?
**A:**
1. Git revert the commit that updated image tag
2. Push to GitOps repo
3. ArgoCD syncs previous version
4. Or use: `argocd app rollback myapp <revision>`

### Q6: How do you handle health checks with ArgoCD?
**A:**
ArgoCD uses Kubernetes probe status:
```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080

# ArgoCD shows "Healthy" when all pods pass probes
```

---

## Multi-Environment Setup

### ApplicationSet for All Environments
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            branch: develop
            replicas: "1"
          - env: staging
            branch: main
            replicas: "2"
          - env: production
            branch: main
            replicas: "5"
  template:
    metadata:
      name: 'myapp-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops-repo.git
        targetRevision: '{{branch}}'
        path: 'overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'myapp-{{env}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Best Practices for .NET GitOps

1. **Separate repos** - Source code and GitOps manifests
2. **Use Kustomize** - Environment-specific overlays
3. **Seal secrets** - Never plain text in Git
4. **Health endpoints** - Implement /health/ready and /health/live
5. **Image immutability** - Use SHA tags, not :latest
6. **Sync waves** - Order migrations before app
7. **Monitor sync status** - Alerts for failed syncs
8. **PR-based promotion** - Review before production
