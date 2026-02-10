# Helm in CI/CD Pipelines

## CI/CD Integration Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HELM IN CI/CD                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pipeline Stages:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Lint ──▶ Test ──▶ Package ──▶ Publish ──▶ Deploy            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Lint:     Validate chart structure and templates                  │
│  Test:     Template rendering and integration tests                │
│  Package:  Create chart archive (.tgz)                             │
│  Publish:  Push to chart repository                                │
│  Deploy:   Install/upgrade to cluster                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GitHub Actions

### Chart Linting and Testing

```yaml
# .github/workflows/helm-test.yaml
name: Helm Chart CI

on:
  push:
    branches: [main, develop]
    paths:
      - 'charts/**'
  pull_request:
    branches: [main]
    paths:
      - 'charts/**'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.12.0

      - name: Set up chart-testing
        uses: helm/chart-testing-action@v2.6.0

      - name: Lint charts
        run: ct lint --all

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Helm
        uses: azure/setup-helm@v3

      - name: Create kind cluster
        uses: helm/kind-action@v1.8.0

      - name: Set up chart-testing
        uses: helm/chart-testing-action@v2.6.0

      - name: Run chart-testing (install)
        run: ct install --all
```

### Chart Release

```yaml
# .github/workflows/helm-release.yaml
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
    permissions:
      contents: write

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

      - name: Add dependencies
        run: |
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm repo update

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.5.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

### Deploy with Helm

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  push:
    branches: [main]

env:
  CLUSTER_NAME: my-cluster
  CLUSTER_REGION: us-east-1

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: ${{ env.CLUSTER_REGION }}

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name ${{ env.CLUSTER_NAME }} --region ${{ env.CLUSTER_REGION }}

      - name: Install Helm
        uses: azure/setup-helm@v3

      - name: Deploy with Helm
        run: |
          helm upgrade --install my-app ./charts/my-app \
            --namespace production \
            --create-namespace \
            --values ./charts/my-app/values-production.yaml \
            --set image.tag=${{ github.sha }} \
            --atomic \
            --timeout 10m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/my-app -n production
```

---

## GitLab CI

### Complete Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - package
  - publish
  - deploy

variables:
  HELM_VERSION: "3.12.0"
  CHART_PATH: "./charts/my-app"

.helm-setup: &helm-setup
  before_script:
    - curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash -s -- --version v${HELM_VERSION}
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
    - helm dependency update ${CHART_PATH}

lint:
  stage: lint
  image: alpine/helm:${HELM_VERSION}
  script:
    - helm lint ${CHART_PATH}
    - helm lint ${CHART_PATH} --strict
  rules:
    - changes:
        - charts/**/*

template-test:
  stage: test
  image: alpine/helm:${HELM_VERSION}
  <<: *helm-setup
  script:
    - helm template my-app ${CHART_PATH} > /dev/null
    - helm template my-app ${CHART_PATH} -f ${CHART_PATH}/values-production.yaml > /dev/null

integration-test:
  stage: test
  image: alpine/k8s:1.28.0
  services:
    - name: registry.gitlab.com/gitlab-org/cluster-integration/test-utils/k3s-gitlab-ci/releases/v1.28.0-k3s1:latest
      alias: k3s
  <<: *helm-setup
  script:
    - export KUBECONFIG=/output/kubeconfig.yaml
    - helm install my-app ${CHART_PATH} --wait --timeout 5m
    - helm test my-app
    - helm uninstall my-app

package:
  stage: package
  image: alpine/helm:${HELM_VERSION}
  <<: *helm-setup
  script:
    - helm package ${CHART_PATH} --destination ./packages
  artifacts:
    paths:
      - packages/*.tgz
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_TAG

publish:
  stage: publish
  image: alpine/helm:${HELM_VERSION}
  script:
    - helm registry login ${CI_REGISTRY} -u ${CI_REGISTRY_USER} -p ${CI_REGISTRY_PASSWORD}
    - helm push packages/*.tgz oci://${CI_REGISTRY}/${CI_PROJECT_PATH}
  rules:
    - if: $CI_COMMIT_TAG

deploy_staging:
  stage: deploy
  image: alpine/helm:${HELM_VERSION}
  environment:
    name: staging
    url: https://staging.example.com
  <<: *helm-setup
  script:
    - helm upgrade --install my-app ${CHART_PATH}
        --namespace staging
        --values ${CHART_PATH}/values-staging.yaml
        --set image.tag=${CI_COMMIT_SHA}
        --atomic
        --timeout 10m
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy_production:
  stage: deploy
  image: alpine/helm:${HELM_VERSION}
  environment:
    name: production
    url: https://app.example.com
  <<: *helm-setup
  script:
    - helm upgrade --install my-app ${CHART_PATH}
        --namespace production
        --values ${CHART_PATH}/values-production.yaml
        --set image.tag=${CI_COMMIT_TAG}
        --atomic
        --timeout 10m
  rules:
    - if: $CI_COMMIT_TAG
  when: manual
```

---

## Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: helm
                    image: alpine/helm:3.12.0
                    command:
                    - cat
                    tty: true
                  - name: kubectl
                    image: bitnami/kubectl:1.28
                    command:
                    - cat
                    tty: true
            '''
        }
    }

    environment {
        CHART_PATH = './charts/my-app'
        KUBECONFIG = credentials('kubeconfig')
    }

    stages {
        stage('Lint') {
            steps {
                container('helm') {
                    sh 'helm lint ${CHART_PATH}'
                    sh 'helm lint ${CHART_PATH} --strict'
                }
            }
        }

        stage('Template Test') {
            steps {
                container('helm') {
                    sh 'helm dependency update ${CHART_PATH}'
                    sh 'helm template my-app ${CHART_PATH} > manifests.yaml'
                }
                container('kubectl') {
                    sh 'kubectl apply --dry-run=client -f manifests.yaml'
                }
            }
        }

        stage('Package') {
            when {
                tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
            }
            steps {
                container('helm') {
                    sh 'helm package ${CHART_PATH} --destination ./packages'
                }
                archiveArtifacts artifacts: 'packages/*.tgz'
            }
        }

        stage('Deploy Staging') {
            when {
                branch 'develop'
            }
            steps {
                container('helm') {
                    sh '''
                        helm upgrade --install my-app ${CHART_PATH} \
                            --namespace staging \
                            --values ${CHART_PATH}/values-staging.yaml \
                            --set image.tag=${GIT_COMMIT} \
                            --atomic \
                            --timeout 10m
                    '''
                }
            }
        }

        stage('Deploy Production') {
            when {
                tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                container('helm') {
                    sh '''
                        helm upgrade --install my-app ${CHART_PATH} \
                            --namespace production \
                            --values ${CHART_PATH}/values-production.yaml \
                            --set image.tag=${TAG_NAME} \
                            --atomic \
                            --timeout 10m
                    '''
                }
            }
        }
    }

    post {
        failure {
            container('helm') {
                sh 'helm rollback my-app --namespace ${NAMESPACE} || true'
            }
        }
    }
}
```

---

## ArgoCD Integration

### Application Manifest

```yaml
# argocd/my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/my-charts
    targetRevision: HEAD
    path: charts/my-app
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "1.0.0"

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

### OCI Registry with ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  source:
    chart: my-app
    repoURL: ghcr.io/myorg/charts
    targetRevision: 1.0.0
    helm:
      releaseName: my-app
      valueFiles:
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

---

## Flux Integration

```yaml
# flux/my-app.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: my-charts
  namespace: flux-system
spec:
  interval: 10m
  url: https://myorg.github.io/helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: my-app
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: my-app
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: my-charts
        namespace: flux-system
  values:
    replicaCount: 3
    image:
      tag: "1.0.0"
  valuesFrom:
    - kind: ConfigMap
      name: my-app-values
  upgrade:
    remediation:
      retries: 3
  rollback:
    cleanupOnFail: true
```

---

## Chart Testing (ct)

```yaml
# ct.yaml - chart-testing configuration
remote: origin
target-branch: main
chart-dirs:
  - charts
chart-repos:
  - bitnami=https://charts.bitnami.com/bitnami
helm-extra-args: --timeout 300s
validate-maintainers: false
check-version-increment: true
```

```bash
# Run chart-testing
ct lint --all
ct lint-and-install --all
ct install --charts charts/my-app

# With custom config
ct lint --config ct.yaml
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CI/CD BEST PRACTICES                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pipeline Design:                                                   │
│  ✓ Always lint before deployment                                  │
│  ✓ Run template tests                                              │
│  ✓ Use --atomic for deployments                                    │
│  ✓ Implement proper rollback                                       │
│                                                                     │
│  Version Management:                                                │
│  ✓ Pin Helm version in pipelines                                   │
│  ✓ Version charts with SemVer                                      │
│  ✓ Use image tags from CI                                          │
│  ✓ Lock dependency versions                                        │
│                                                                     │
│  Security:                                                          │
│  ✓ Use OIDC for cloud auth                                         │
│  ✓ Don't expose credentials                                        │
│  ✓ Scan charts for vulnerabilities                                 │
│  ✓ Use signed charts when possible                                 │
│                                                                     │
│  Testing:                                                           │
│  ✓ Test in isolated namespace                                      │
│  ✓ Run helm test after install                                     │
│  ✓ Verify rollout status                                           │
│  ✓ Clean up test resources                                         │
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
│  Pipeline Stages:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Lint      │ helm lint, chart-testing                        │   │
│  │ Test      │ Template validation, integration tests          │   │
│  │ Package   │ helm package                                    │   │
│  │ Publish   │ Push to repository/OCI                          │   │
│  │ Deploy    │ helm upgrade --install --atomic                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Tools:                                                             │
│  • chart-testing (ct) - Lint and test charts                       │
│  • chart-releaser - Publish to GitHub Pages                        │
│  • helm-diff - Preview changes                                     │
│  • ArgoCD/Flux - GitOps deployment                                 │
│                                                                     │
│  Key Patterns:                                                      │
│  • Use --atomic for safe deployments                               │
│  • Pin versions everywhere                                         │
│  • Separate values per environment                                 │
│  • Automate testing and releases                                   │
│                                                                     │
│  Next: Learn about Helm plugins                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
