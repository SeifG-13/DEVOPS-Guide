# Introduction to CI/CD

## What is CI/CD?

CI/CD is a method to frequently deliver apps by introducing automation into the stages of app development.

```
┌─────────────────────────────────────────────────────────────────┐
│                      CI/CD Pipeline                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Continuous Integration (CI)                                   │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐       │
│   │  Code   │──►│  Build  │──►│  Test   │──►│ Analyze │       │
│   │ Commit  │   │         │   │         │   │         │       │
│   └─────────┘   └─────────┘   └─────────┘   └─────────┘       │
│                                                                  │
│   Continuous Delivery (CD)                                      │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐                     │
│   │ Package │──►│ Deploy  │──►│ Manual  │                     │
│   │Artifact │   │ Staging │   │ Approve │                     │
│   └─────────┘   └─────────┘   └─────────┘                     │
│                                                                  │
│   Continuous Deployment (CD)                                    │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐                     │
│   │ Package │──►│ Deploy  │──►│ Deploy  │                     │
│   │Artifact │   │ Staging │   │  Prod   │  (Automatic)        │
│   └─────────┘   └─────────┘   └─────────┘                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## CI vs CD vs CD

| Term | Full Name | Description |
|------|-----------|-------------|
| **CI** | Continuous Integration | Automatically build and test code changes |
| **CD** | Continuous Delivery | Automatically prepare releases for deployment |
| **CD** | Continuous Deployment | Automatically deploy every change to production |

```
┌─────────────────────────────────────────────────────────────────┐
│                   CI/CD Comparison                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Continuous Integration                                        │
│   ─────────────────────────────────────────────────            │
│   Code → Build → Test → Merge                                  │
│   • Multiple daily integrations                                 │
│   • Automated testing                                           │
│   • Fast feedback                                               │
│                                                                  │
│   Continuous Delivery                                           │
│   ─────────────────────────────────────────────────            │
│   ... → Package → Deploy to Staging → [Manual] → Production    │
│   • Always deployable                                           │
│   • Manual production deployment                                │
│   • Release on demand                                           │
│                                                                  │
│   Continuous Deployment                                         │
│   ─────────────────────────────────────────────────            │
│   ... → Package → Deploy to Staging → [Auto] → Production      │
│   • Every change goes to production                            │
│   • Fully automated                                             │
│   • Requires mature testing                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Benefits of CI/CD

### For Development Teams

| Benefit | Description |
|---------|-------------|
| **Faster Feedback** | Know immediately if code breaks |
| **Reduced Risk** | Smaller, frequent releases |
| **Higher Quality** | Automated testing catches bugs early |
| **Less Manual Work** | Automation handles repetitive tasks |

### For Business

| Benefit | Description |
|---------|-------------|
| **Faster Time to Market** | Release features quickly |
| **Reduced Costs** | Less time fixing bugs |
| **Customer Satisfaction** | Frequent updates and fixes |
| **Competitive Advantage** | Rapid innovation |

## Pipeline Stages

```
┌─────────────────────────────────────────────────────────────────┐
│                    Typical Pipeline Stages                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Source                                                     │
│      └── Code checkout, trigger on commit/PR                   │
│                                                                  │
│   2. Build                                                      │
│      └── Compile code, resolve dependencies                    │
│                                                                  │
│   3. Test                                                       │
│      ├── Unit tests                                            │
│      ├── Integration tests                                     │
│      └── Code coverage                                         │
│                                                                  │
│   4. Analysis                                                   │
│      ├── Static code analysis                                  │
│      ├── Security scanning                                     │
│      └── Quality gates                                         │
│                                                                  │
│   5. Package                                                    │
│      └── Create artifacts (JAR, Docker image, etc.)           │
│                                                                  │
│   6. Deploy to Staging                                          │
│      └── Deploy to test environment                            │
│                                                                  │
│   7. Acceptance Tests                                           │
│      ├── Smoke tests                                           │
│      ├── UI tests                                              │
│      └── Performance tests                                     │
│                                                                  │
│   8. Deploy to Production                                       │
│      └── Blue-green, canary, or rolling deployment            │
│                                                                  │
│   9. Monitor                                                    │
│      └── Health checks, alerts, rollback if needed            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## CI/CD Tools

### CI/CD Platforms

| Tool | Type | Description |
|------|------|-------------|
| **Jenkins** | Self-hosted | Most popular, highly extensible |
| **GitHub Actions** | Cloud/Self-hosted | Native GitHub integration |
| **GitLab CI** | Cloud/Self-hosted | Built into GitLab |
| **Azure DevOps** | Cloud/Self-hosted | Microsoft ecosystem |
| **CircleCI** | Cloud | Fast, easy configuration |
| **Travis CI** | Cloud | Popular for open source |
| **TeamCity** | Self-hosted | JetBrains product |
| **Bamboo** | Self-hosted | Atlassian product |

### Supporting Tools

| Category | Tools |
|----------|-------|
| **Version Control** | Git, GitHub, GitLab, Bitbucket |
| **Build Tools** | Maven, Gradle, npm, Make |
| **Testing** | JUnit, pytest, Jest, Selenium |
| **Code Quality** | SonarQube, ESLint, Checkstyle |
| **Artifact Storage** | Nexus, Artifactory, Docker Registry |
| **Deployment** | Kubernetes, Ansible, Terraform |
| **Monitoring** | Prometheus, Grafana, ELK Stack |

## Pipeline as Code

Modern CI/CD uses configuration files stored in version control.

### Jenkins (Jenkinsfile)

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
            }
        }
    }
}
```

### GitHub Actions

```yaml
name: CI/CD Pipeline
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: mvn clean compile
      - name: Test
        run: mvn test
      - name: Deploy
        run: kubectl apply -f deployment.yaml
```

### Azure DevOps

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: mvn clean compile
          - script: mvn test
  - stage: Deploy
    jobs:
      - job: DeployJob
        steps:
          - script: kubectl apply -f deployment.yaml
```

## CI/CD Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                   CI/CD Best Practices                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Build                                                         │
│   ✓ Build once, deploy many                                    │
│   ✓ Keep builds fast (< 10 minutes)                            │
│   ✓ Fail fast - run quick tests first                          │
│                                                                  │
│   Test                                                          │
│   ✓ Automate all tests                                         │
│   ✓ Test in production-like environments                       │
│   ✓ Maintain high code coverage                                │
│                                                                  │
│   Deploy                                                        │
│   ✓ Use infrastructure as code                                 │
│   ✓ Deploy to staging before production                        │
│   ✓ Implement rollback mechanisms                              │
│                                                                  │
│   Security                                                      │
│   ✓ Scan for vulnerabilities                                   │
│   ✓ Manage secrets securely                                    │
│   ✓ Implement access controls                                  │
│                                                                  │
│   General                                                       │
│   ✓ Version control everything                                 │
│   ✓ Make pipeline visible to all                               │
│   ✓ Monitor and measure pipeline metrics                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## DevOps and CI/CD

```
┌─────────────────────────────────────────────────────────────────┐
│                    DevOps Infinity Loop                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                         ┌─────────┐                             │
│              Plan ──────│         │────── Monitor               │
│            /            │  CI/CD  │            \                │
│         Code            │ Pipeline│           Operate           │
│            \            │         │            /                │
│              Build ─────│         │────── Deploy                │
│                         └─────────┘                             │
│                    Test ─────┴───── Release                     │
│                                                                  │
│   DEV Side (CI)                    OPS Side (CD)                │
│   • Plan                           • Deploy                     │
│   • Code                           • Operate                    │
│   • Build                          • Monitor                    │
│   • Test                                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Reference

### Pipeline Triggers

| Trigger | Description |
|---------|-------------|
| Push | On code push to branch |
| Pull Request | On PR creation/update |
| Schedule | Cron-based triggers |
| Manual | User-initiated |
| Webhook | External event trigger |

### Key Metrics

| Metric | Description |
|--------|-------------|
| Lead Time | Time from commit to production |
| Deployment Frequency | How often you deploy |
| MTTR | Mean time to recovery |
| Change Failure Rate | % of deployments causing failures |

---

**Next:** [02-version-control-integration.md](02-version-control-integration.md)
