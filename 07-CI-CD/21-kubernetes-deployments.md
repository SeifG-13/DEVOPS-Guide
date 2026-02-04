# Kubernetes Deployments in CI/CD

## CI/CD to Kubernetes Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                 CI/CD to Kubernetes Pipeline                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Code Push                                                     │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────┐                                              │
│   │   CI Build  │  Build, Test, Create Image                   │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │   Registry  │  Store Container Image                       │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │  CD Deploy  │  Apply Kubernetes Manifests                  │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │ Kubernetes  │  Run Application                             │
│   └─────────────┘                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## kubectl in CI/CD

### Setting Up kubectl

```yaml
# GitHub Actions
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Deploy
        run: kubectl apply -f k8s/
```

### kubectl with Cloud Providers

```yaml
# GKE
- uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}

- uses: google-github-actions/get-gke-credentials@v2
  with:
    cluster_name: my-cluster
    location: us-central1

- run: kubectl apply -f k8s/

# EKS
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- run: aws eks update-kubeconfig --name my-cluster
- run: kubectl apply -f k8s/

# AKS
- uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- uses: azure/aks-set-context@v3
  with:
    resource-group: my-rg
    cluster-name: my-cluster

- run: kubectl apply -f k8s/
```

## Helm Deployments

### Helm in CI/CD

```yaml
# GitHub Actions
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-helm@v3
        with:
          version: 'v3.12.0'

      - name: Configure kubectl
        run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Helm Deploy
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --set replicas=3 \
            --wait \
            --timeout 5m
```

### Helm Chart Structure

```
charts/myapp/
├── Chart.yaml
├── values.yaml
├── values-staging.yaml
├── values-production.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── configmap.yaml
```

### values.yaml

```yaml
# values.yaml
replicaCount: 2

image:
  repository: myregistry/myapp
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Environment-Specific Values

```yaml
# values-production.yaml
replicaCount: 5

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

ingress:
  hosts:
    - host: myapp.example.com
```

```bash
# Deploy with environment values
helm upgrade --install myapp ./charts/myapp \
  -f ./charts/myapp/values.yaml \
  -f ./charts/myapp/values-production.yaml \
  --set image.tag=$VERSION
```

## Kustomize Deployments

### Kustomize Structure

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

### Base Configuration

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: myapp
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
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
        image: myapp:latest
        ports:
        - containerPort: 8080
```

### Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
  - ../../base

patchesStrategicMerge:
  - replica-patch.yaml

images:
  - name: myapp
    newTag: v1.2.3
```

```yaml
# overlays/production/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

### Kustomize in CI/CD

```yaml
# GitHub Actions
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Update image tag
        run: |
          cd k8s/overlays/production
          kustomize edit set image myapp=myregistry/myapp:${{ github.sha }}

      - name: Deploy
        run: |
          kubectl apply -k k8s/overlays/production
```

## GitOps with ArgoCD

### ArgoCD Application

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: HEAD
    path: k8s/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### ArgoCD with Helm

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/helm-charts
    targetRevision: HEAD
    path: charts/myapp
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: v1.2.3

  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### CI Pipeline for GitOps

```yaml
# GitHub Actions - Update config repo
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v4

      - id: build
        run: |
          TAG=${{ github.sha }}
          docker build -t myregistry/myapp:$TAG .
          docker push myregistry/myapp:$TAG
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  update-config:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/myapp-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}

      - name: Update image tag
        run: |
          cd k8s/overlays/production
          kustomize edit set image myapp=myregistry/myapp:${{ needs.build.outputs.image-tag }}

      - name: Commit and push
        run: |
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add .
          git commit -m "Update myapp to ${{ needs.build.outputs.image-tag }}"
          git push
```

## FluxCD

### Flux GitRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp-config
  ref:
    branch: main
  secretRef:
    name: github-token
```

### Flux Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./k8s/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
```

### Flux HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: myapp
      version: '1.x'
      sourceRef:
        kind: HelmRepository
        name: myorg
        namespace: flux-system
  values:
    image:
      tag: v1.2.3
    replicas: 5
```

## Jenkins Kubernetes Deployment

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        REGISTRY = 'myregistry'
        IMAGE_NAME = 'myapp'
        KUBECONFIG = credentials('kubeconfig')
    }

    stages {
        stage('Build') {
            steps {
                script {
                    docker.build("${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://myregistry', 'registry-creds') {
                        docker.image("${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}").push()
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        helm upgrade --install ${IMAGE_NAME} ./charts/${IMAGE_NAME} \
                            --namespace staging \
                            --set image.tag=${BUILD_NUMBER} \
                            --wait
                    """
                }
            }
        }

        stage('Integration Tests') {
            steps {
                sh './scripts/integration-tests.sh'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
                    sh """
                        helm upgrade --install ${IMAGE_NAME} ./charts/${IMAGE_NAME} \
                            --namespace production \
                            -f ./charts/${IMAGE_NAME}/values-production.yaml \
                            --set image.tag=${BUILD_NUMBER} \
                            --wait
                    """
                }
            }
        }
    }
}
```

## Complete Pipeline Example

```yaml
# GitHub Actions - Full Kubernetes deployment
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ github.sha }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-kubectl@v3

      - uses: azure/setup-helm@v3

      - name: Configure kubectl
        run: echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > $HOME/.kube/config

      - name: Deploy
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace staging \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --wait --timeout 5m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n staging
          kubectl get pods -n staging -l app=myapp

  deploy-production:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-kubectl@v3

      - uses: azure/setup-helm@v3

      - name: Configure kubectl
        run: echo "${{ secrets.KUBECONFIG_PROD }}" | base64 -d > $HOME/.kube/config

      - name: Deploy
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            -f ./charts/myapp/values-production.yaml \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --wait --timeout 10m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n production
```

## Quick Reference

### Deployment Tools

| Tool | Type | Best For |
|------|------|----------|
| kubectl | Imperative | Simple deployments |
| Helm | Templating | Complex apps, releases |
| Kustomize | Patching | Environment overlays |
| ArgoCD | GitOps | Declarative, audited |
| FluxCD | GitOps | Automated sync |

### Common Commands

| Task | Command |
|------|---------|
| Apply manifests | `kubectl apply -f k8s/` |
| Helm install | `helm upgrade --install app ./chart` |
| Kustomize apply | `kubectl apply -k overlays/prod` |
| Check rollout | `kubectl rollout status deploy/app` |
| Rollback | `kubectl rollout undo deploy/app` |

---

**Previous:** [20-deployment-strategies.md](20-deployment-strategies.md) | **Next:** [22-security-in-cicd.md](22-security-in-cicd.md)
