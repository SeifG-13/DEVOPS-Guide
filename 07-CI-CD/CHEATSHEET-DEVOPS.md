# CI/CD Cheat Sheet for DevOps Engineers

## Quick Reference - CI/CD Concepts

### CI/CD Pipeline Stages
```
Source → Build → Test → Security Scan → Package → Deploy → Monitor
```

| Stage | Purpose | Tools |
|-------|---------|-------|
| Source | Version control | Git, GitHub, GitLab |
| Build | Compile code | Maven, npm, dotnet |
| Test | Validate code | JUnit, pytest, xUnit |
| Scan | Security analysis | SonarQube, Trivy |
| Package | Create artifacts | Docker, NuGet |
| Deploy | Release to env | Kubernetes, ArgoCD |
| Monitor | Observe production | Prometheus, Grafana |

---

## GitHub Actions

### Basic Workflow
```yaml
name: CI/CD Pipeline

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

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build

  docker:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    needs: docker
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/deployment.yaml
          images: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

### Useful Actions
```yaml
# Caching
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

# Secrets
${{ secrets.MY_SECRET }}

# Matrix builds
strategy:
  matrix:
    node-version: [18, 20, 21]
    os: [ubuntu-latest, windows-latest]
```

---

## GitLab CI/CD

### .gitlab-ci.yml
```yaml
stages:
  - build
  - test
  - security
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  image: node:20
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

test:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm test
  coverage: '/Coverage: \d+\.\d+%/'

security_scan:
  stage: security
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker run --rm aquasec/trivy image $DOCKER_IMAGE

deploy_staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl apply -f k8s/
  only:
    - develop

deploy_production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - kubectl apply -f k8s/
  only:
    - main
  when: manual
```

---

## Azure DevOps Pipelines

### azure-pipelines.yml
```yaml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - task: UseDotNet@2
            inputs:
              version: '8.0.x'

          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: 'restore'

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              arguments: '--configuration $(buildConfiguration)'

          - task: DotNetCoreCLI@2
            displayName: 'Test'
            inputs:
              command: 'test'
              arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'

          - task: Docker@2
            displayName: 'Build and Push'
            inputs:
              containerRegistry: 'myACR'
              repository: 'myapp'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest

  - stage: Deploy
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployToAKS
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  inputs:
                    action: 'deploy'
                    manifests: '$(Pipeline.Workspace)/k8s/*.yaml'
```

---

## Jenkins

### Jenkinsfile (Declarative)
```groovy
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    junit 'test-results/*.xml'
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}").push()
                        docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }

    post {
        success {
            slackSend channel: '#deployments', message: "Build ${BUILD_NUMBER} succeeded!"
        }
        failure {
            slackSend channel: '#deployments', message: "Build ${BUILD_NUMBER} failed!"
        }
    }
}
```

---

## Interview Q&A

### Q1: What is the difference between CI and CD?
**A:**
- **CI (Continuous Integration)**: Frequently merge code changes, automate build and test
- **CD (Continuous Delivery)**: Automate deployment to staging, manual approval for production
- **CD (Continuous Deployment)**: Fully automated deployment to production

### Q2: Explain blue-green deployment
**A:** Two identical production environments (blue/green). Deploy to inactive environment, test, then switch traffic. Benefits:
- Zero downtime
- Easy rollback (switch back)
- Full testing in production-like environment

### Q3: Explain canary deployment
**A:** Gradually roll out changes to a small subset of users before full deployment. Steps:
1. Deploy new version to small percentage (5%)
2. Monitor metrics and errors
3. Gradually increase traffic
4. Full rollout or rollback

### Q4: What is a rolling deployment?
**A:** Gradually replace old instances with new ones. In Kubernetes:
- `maxSurge`: Extra pods during update
- `maxUnavailable`: Max unavailable pods

### Q5: How do you handle secrets in CI/CD?
**A:**
- CI/CD platform secrets (GitHub Secrets, GitLab Variables)
- External secret managers (Vault, AWS Secrets Manager)
- Never commit secrets to repository
- Rotate secrets regularly
- Audit secret access

### Q6: What is a pipeline artifact?
**A:** Output from a build stage passed to subsequent stages:
- Compiled binaries
- Docker images
- Test reports
- Coverage reports

### Q7: How do you implement CI/CD for microservices?
**A:**
- Mono-repo: Path-based triggers, shared libraries
- Multi-repo: Separate pipelines per service
- Contract testing between services
- Independent deployment per service
- Service mesh for traffic management

### Q8: What metrics should you track in CI/CD?
**A:**
- **Lead time**: Code commit to production
- **Deployment frequency**: How often you deploy
- **Change failure rate**: Failed deployments
- **MTTR**: Mean time to recover
- Build duration, test coverage, queue time

### Q9: How do you handle database migrations in CI/CD?
**A:**
- Version control migrations
- Run migrations as pipeline stage
- Use migration tools (Flyway, EF Migrations)
- Test migrations in staging first
- Have rollback plan

### Q10: What is infrastructure as code in CI/CD?
**A:** Define infrastructure in version-controlled files:
- Terraform/Pulumi for cloud resources
- Ansible for configuration
- Pipeline stages for infra changes
- Same review process as application code

---

## Pipeline Best Practices

### Security
```yaml
# Scan dependencies
- name: Dependency scan
  run: npm audit --audit-level=high

# Container scanning
- name: Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myimage:latest

# SAST scanning
- name: SonarQube
  run: sonar-scanner
```

### Performance
```yaml
# Parallel jobs
jobs:
  test-unit:
    runs-on: ubuntu-latest
  test-integration:
    runs-on: ubuntu-latest
  test-e2e:
    runs-on: ubuntu-latest

# Caching
- uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ hashFiles('package-lock.json') }}
```

### Reliability
```yaml
# Retry on failure
- name: Deploy with retry
  uses: nick-fields/retry@v2
  with:
    max_attempts: 3
    command: kubectl apply -f k8s/

# Timeout
jobs:
  build:
    timeout-minutes: 30
```

---

## Common Pipeline Patterns

### Feature Branch Workflow
```
feature/* → PR → develop → staging
main → production
```

### GitFlow
```
feature/* → develop → release/* → main
hotfix/* → main + develop
```

### Trunk-Based
```
main (protected)
  └── short-lived branches → PR → main → deploy
```

---

## Deployment Strategies Comparison

| Strategy | Downtime | Rollback | Risk | Complexity |
|----------|----------|----------|------|------------|
| Recreate | Yes | Slow | High | Low |
| Rolling | No | Medium | Medium | Medium |
| Blue-Green | No | Fast | Low | Medium |
| Canary | No | Fast | Low | High |
| A/B Testing | No | Fast | Low | High |
