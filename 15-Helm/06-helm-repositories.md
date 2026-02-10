# Helm Repositories

## Repository Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM REPOSITORIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What is a Helm Repository?                                        │
│  • HTTP server hosting chart packages                              │
│  • Contains index.yaml with chart metadata                         │
│  • Charts packaged as .tgz files                                   │
│                                                                     │
│  Repository Types:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ HTTP/HTTPS    │ Traditional web server                      │   │
│  │ OCI Registry  │ Container registry (Docker, Harbor)         │   │
│  │ ChartMuseum   │ Dedicated chart repository                  │   │
│  │ GitHub Pages  │ Static file hosting                         │   │
│  │ S3/GCS/Azure  │ Cloud storage                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Working with Repositories

### Repository Commands

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# Add with authentication
helm repo add private https://charts.example.com \
  --username admin \
  --password secret

# Add with certificate
helm repo add secure https://charts.example.com \
  --ca-file ca.crt \
  --cert-file client.crt \
  --key-file client.key

# Update repositories
helm repo update

# Update specific repository
helm repo update bitnami

# List repositories
helm repo list

# Remove repository
helm repo remove stable

# Generate repository index
helm repo index ./charts
```

### Searching Repositories

```bash
# Search local repos
helm search repo nginx

# Search with versions
helm search repo nginx --versions

# Search with version constraint
helm search repo nginx --version "^15.0.0"

# Search all repos
helm search repo ""

# Search Artifact Hub
helm search hub wordpress
helm search hub prometheus --max-col-width 80
```

---

## Popular Public Repositories

```bash
# Bitnami (Comprehensive application charts)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Prometheus Community
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Grafana
helm repo add grafana https://grafana.github.io/helm-charts

# Ingress NGINX
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Jetstack (cert-manager)
helm repo add jetstack https://charts.jetstack.io

# HashiCorp
helm repo add hashicorp https://helm.releases.hashicorp.com

# Argo
helm repo add argo https://argoproj.github.io/argo-helm

# Elastic
helm repo add elastic https://helm.elastic.co

# Apache (Airflow, Kafka, etc.)
helm repo add apache-airflow https://airflow.apache.org

# Update all
helm repo update
```

---

## OCI Registries

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OCI REGISTRY SUPPORT                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Helm 3.8+ supports OCI (Open Container Initiative) registries     │
│                                                                     │
│  Benefits:                                                          │
│  • Same infrastructure as container images                         │
│  • Better security (signing, scanning)                             │
│  • No need for index.yaml                                          │
│  • Leverage existing registry authentication                       │
│                                                                     │
│  Supported Registries:                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Docker Hub       │ docker.io                                │   │
│  │ GitHub Container │ ghcr.io                                  │   │
│  │ Harbor           │ harbor.example.com                       │   │
│  │ AWS ECR          │ xxx.dkr.ecr.region.amazonaws.com         │   │
│  │ Google Artifact  │ region-docker.pkg.dev                    │   │
│  │ Azure ACR        │ xxx.azurecr.io                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Using OCI Registries

```bash
# Login to registry
helm registry login ghcr.io -u username

# Pull chart from OCI registry
helm pull oci://ghcr.io/myorg/mychart --version 1.0.0

# Install from OCI registry
helm install my-release oci://ghcr.io/myorg/mychart --version 1.0.0

# Push chart to OCI registry
helm push mychart-1.0.0.tgz oci://ghcr.io/myorg

# Show chart info
helm show all oci://ghcr.io/myorg/mychart --version 1.0.0

# Template from OCI
helm template my-release oci://ghcr.io/myorg/mychart --version 1.0.0

# Logout
helm registry logout ghcr.io
```

### OCI in Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0

dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts

  - name: redis
    version: "17.x.x"
    repository: oci://ghcr.io/bitnami-labs/charts
```

---

## Setting Up ChartMuseum

```bash
# Install ChartMuseum
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm install chartmuseum chartmuseum/chartmuseum \
  --set env.open.DISABLE_API=false \
  --set persistence.enabled=true \
  --set persistence.size=5Gi
```

```yaml
# ChartMuseum values.yaml
env:
  open:
    STORAGE: local
    DISABLE_API: false
    ALLOW_OVERWRITE: true
    AUTH_ANONYMOUS_GET: true
    CHART_POST_FORM_FIELD_NAME: chart
    PROV_POST_FORM_FIELD_NAME: prov

persistence:
  enabled: true
  size: 10Gi
  storageClass: standard

ingress:
  enabled: true
  hosts:
    - name: charts.example.com
      path: /
  tls:
    - secretName: charts-tls
      hosts:
        - charts.example.com
```

### Using ChartMuseum

```bash
# Add as repository
helm repo add myrepo https://charts.example.com

# Upload chart (with API enabled)
curl --data-binary "@mychart-1.0.0.tgz" https://charts.example.com/api/charts

# Delete chart
curl -X DELETE https://charts.example.com/api/charts/mychart/1.0.0

# Using helm-push plugin
helm plugin install https://github.com/chartmuseum/helm-push
helm cm-push mychart/ myrepo
```

---

## GitHub Pages Repository

### Setting Up

```bash
# Create charts directory
mkdir -p charts

# Package your chart
helm package ./my-chart -d charts/

# Generate index
helm repo index charts/ --url https://username.github.io/helm-charts

# Commit and push
git add charts/
git commit -m "Add helm charts"
git push

# Enable GitHub Pages for the repository
# Settings -> Pages -> Source: main branch, /charts folder
```

### GitHub Actions Automation

```yaml
# .github/workflows/release-charts.yaml
name: Release Charts

on:
  push:
    branches:
      - main
    paths:
      - 'charts/**'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.12.0

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.5.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

---

## S3/GCS/Azure Storage

### S3 Repository

```bash
# Install helm-s3 plugin
helm plugin install https://github.com/hypnoglow/helm-s3.git

# Initialize S3 bucket as repository
helm s3 init s3://my-helm-charts/stable

# Add repository
helm repo add my-s3-repo s3://my-helm-charts/stable

# Push chart
helm s3 push ./my-chart-1.0.0.tgz my-s3-repo

# Update repository
helm repo update
```

### GCS Repository

```bash
# Install helm-gcs plugin
helm plugin install https://github.com/hayorov/helm-gcs.git

# Initialize GCS bucket
helm gcs init gs://my-helm-charts

# Add repository
helm repo add my-gcs-repo gs://my-helm-charts

# Push chart
helm gcs push ./my-chart-1.0.0.tgz my-gcs-repo
```

---

## Private Repository Authentication

### Basic Auth

```bash
# Add with credentials
helm repo add private https://charts.example.com \
  --username admin \
  --password secret

# Using environment variables
export HELM_REPO_USERNAME=admin
export HELM_REPO_PASSWORD=secret
helm repo add private https://charts.example.com
```

### Token Authentication

```bash
# Bearer token
helm repo add private https://charts.example.com \
  --pass-credentials

# With custom headers (via plugin or wrapper)
```

### Certificate Authentication

```bash
# mTLS
helm repo add private https://charts.example.com \
  --ca-file /path/to/ca.crt \
  --cert-file /path/to/client.crt \
  --key-file /path/to/client.key

# Skip TLS verification (not recommended)
helm repo add private https://charts.example.com \
  --insecure-skip-tls-verify
```

---

## Repository Index

```yaml
# index.yaml structure
apiVersion: v1
entries:
  my-app:
    - apiVersion: v2
      appVersion: "2.0.0"
      created: "2024-01-15T10:30:00.000000000Z"
      description: My application chart
      digest: sha256:abc123...
      name: my-app
      type: application
      urls:
        - https://charts.example.com/my-app-1.0.0.tgz
      version: 1.0.0

    - apiVersion: v2
      appVersion: "1.5.0"
      created: "2024-01-10T10:30:00.000000000Z"
      description: My application chart
      digest: sha256:def456...
      name: my-app
      type: application
      urls:
        - https://charts.example.com/my-app-0.9.0.tgz
      version: 0.9.0

generated: "2024-01-15T10:30:00.000000000Z"
```

### Generating Index

```bash
# Generate for current directory
helm repo index .

# Generate with URL
helm repo index . --url https://charts.example.com

# Merge with existing index
helm repo index . --merge index.yaml --url https://charts.example.com
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                REPOSITORY BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Security:                                                          │
│  ✓ Use HTTPS for all repositories                                  │
│  ✓ Enable authentication for private repos                        │
│  ✓ Sign charts with GPG (provenance)                               │
│  ✓ Scan charts for vulnerabilities                                 │
│                                                                     │
│  Organization:                                                      │
│  ✓ Separate repos for different environments                       │
│  ✓ Use meaningful chart names                                      │
│  ✓ Version charts properly                                         │
│  ✓ Keep repository indexes small                                   │
│                                                                     │
│  Operations:                                                        │
│  ✓ Use OCI for new deployments                                     │
│  ✓ Automate chart publishing                                       │
│  ✓ Mirror public charts internally                                 │
│  ✓ Regular cleanup of old versions                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Repository Types:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ HTTP/HTTPS    │ Traditional, index.yaml based              │   │
│  │ OCI           │ Modern, container registry based           │   │
│  │ ChartMuseum   │ Feature-rich, API support                  │   │
│  │ GitHub Pages  │ Free, static hosting                       │   │
│  │ Cloud Storage │ S3, GCS, Azure Blob                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Commands:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ helm repo add     │ Add repository                          │   │
│  │ helm repo update  │ Update repository index                 │   │
│  │ helm repo list    │ List repositories                       │   │
│  │ helm search repo  │ Search charts                           │   │
│  │ helm repo index   │ Generate index.yaml                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Recommendation: Use OCI registries for new projects               │
│                                                                     │
│  Next: Learn about Helm values and configuration                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
