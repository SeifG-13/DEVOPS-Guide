# CI/CD Pipeline Best Practices

## Pipeline Design Principles

```
┌─────────────────────────────────────────────────────────────────┐
│                 CI/CD Best Practices Overview                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Speed                                                         │
│   ├── Fast feedback loops                                       │
│   ├── Parallel execution                                        │
│   └── Efficient caching                                         │
│                                                                  │
│   Reliability                                                   │
│   ├── Reproducible builds                                       │
│   ├── Idempotent deployments                                    │
│   └── Automated rollbacks                                       │
│                                                                  │
│   Security                                                      │
│   ├── Secret management                                         │
│   ├── Least privilege access                                    │
│   └── Security scanning                                         │
│                                                                  │
│   Maintainability                                               │
│   ├── Pipeline as Code                                          │
│   ├── Modular design                                            │
│   └── Documentation                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Fast Feedback Loops

### Fail Fast Strategy

```yaml
# Run fastest checks first
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint  # Seconds

  unit-tests:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- --testPathPattern=unit  # Minutes

  integration-tests:
    needs: unit-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- --testPathPattern=integration  # Longer

  e2e-tests:
    needs: integration-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:e2e  # Longest
```

### Parallel Execution

```yaml
# Run independent jobs in parallel
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  type-check:
    runs-on: ubuntu-latest
    steps:
      - run: npm run type-check

  security:
    runs-on: ubuntu-latest
    steps:
      - run: npm audit

  # These run in parallel, then build waits for all
  build:
    needs: [lint, type-check, security]
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
```

### Test Parallelization

```yaml
# Split tests across runners
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --shard=${{ matrix.shard }}/4
```

## Caching Best Practices

### Dependency Caching

```yaml
# Node.js
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# Python
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

# Maven
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
```

### Docker Layer Caching

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Cache Strategies

```yaml
# Cache with fallback
- uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      ${{ runner.os }}-node-
      ${{ runner.os }}-
```

## Pipeline as Code

### Version Control Pipelines

```
Repository structure:
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── cd.yml
│       └── release.yml
├── Jenkinsfile
├── azure-pipelines.yml
└── src/
```

### Modular Pipeline Design

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    uses: ./.github/workflows/reusable-lint.yml

  test:
    needs: lint
    uses: ./.github/workflows/reusable-test.yml
    with:
      coverage: true

  build:
    needs: test
    uses: ./.github/workflows/reusable-build.yml
    secrets: inherit
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test

on:
  workflow_call:
    inputs:
      coverage:
        type: boolean
        default: false
    secrets:
      CODECOV_TOKEN:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test ${{ inputs.coverage && '-- --coverage' || '' }}
      - if: inputs.coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

## Environment Management

### Environment Promotion

```yaml
jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    environment: development
    steps:
      - run: ./deploy.sh dev

  deploy-staging:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - run: ./deploy.sh staging

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: ./deploy.sh production
```

### Environment-Specific Configuration

```yaml
# Use environment secrets and variables
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
    steps:
      - name: Deploy
        env:
          API_URL: ${{ vars.API_URL }}  # Environment variable
          API_KEY: ${{ secrets.API_KEY }}  # Environment secret
        run: ./deploy.sh
```

## Error Handling

### Graceful Failure Handling

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        id: tests
        continue-on-error: true
        run: npm test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results/

      - name: Check test status
        if: steps.tests.outcome == 'failure'
        run: |
          echo "Tests failed, but we saved the results"
          exit 1
```

### Retry Logic

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: nick-invision/retry@v2
        with:
          timeout_minutes: 5
          max_attempts: 3
          retry_wait_seconds: 30
          command: ./deploy.sh
```

### Rollback on Failure

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Get current version
        id: current
        run: echo "version=$(kubectl get deploy myapp -o jsonpath='{.spec.template.spec.containers[0].image}')" >> $GITHUB_OUTPUT

      - name: Deploy new version
        id: deploy
        run: |
          kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=5m

      - name: Rollback on failure
        if: failure() && steps.deploy.outcome == 'failure'
        run: |
          echo "Deployment failed, rolling back to ${{ steps.current.outputs.version }}"
          kubectl rollout undo deployment/myapp
```

## Notifications

### Slack Notifications

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: ./deploy.sh

      - name: Notify success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Deployment succeeded for ${{ github.repository }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Succeeded*\nRepo: ${{ github.repository }}\nBranch: ${{ github.ref_name }}\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Deployment failed for ${{ github.repository }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Documentation and Comments

### Self-Documenting Pipelines

```yaml
name: CI/CD Pipeline

# This pipeline runs on push to main and PRs
# It builds, tests, and deploys the application

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Step 1: Static analysis and linting
  lint:
    name: 'Lint & Type Check'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: npm ci
      - name: Run ESLint
        run: npm run lint
      - name: Type check with TypeScript
        run: npm run type-check

  # Step 2: Run test suite
  test:
    name: 'Unit & Integration Tests'
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: npm ci
      - name: Run test suite with coverage
        run: npm test -- --coverage

  # Step 3: Build application
  build:
    name: 'Build Application'
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build production bundle
        run: npm run build
```

## Complete Best Practices Pipeline

```yaml
name: Production Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

# Ensure only one deployment runs at a time
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  # Fast checks first
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  # Security scanning in parallel
  security:
    name: Security
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: github/codeql-action/init@v2
        with:
          languages: javascript
      - uses: github/codeql-action/autobuild@v2
      - uses: github/codeql-action/analyze@v2

  # Tests with coverage
  test:
    name: Test
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v3

  # Build and push container
  build:
    name: Build
    needs: [test, security]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ github.sha }}
    permissions:
      packages: write
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
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  # Deploy to staging (auto)
  deploy-staging:
    name: Deploy Staging
    if: github.event_name != 'pull_request'
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace staging \
            --set image.tag=${{ needs.build.outputs.image-tag }} \
            --wait

  # Deploy to production (manual approval)
  deploy-production:
    name: Deploy Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            -f ./charts/myapp/values-production.yaml \
            --set image.tag=${{ needs.build.outputs.image-tag }} \
            --wait

      - name: Notify deployment
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text": "✅ Production deployment complete"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Quick Reference Checklist

### Pipeline Design

- [ ] Fast feedback (fail fast)
- [ ] Parallel execution where possible
- [ ] Efficient caching
- [ ] Clear job dependencies
- [ ] Concurrency control

### Security

- [ ] Secret management
- [ ] Least privilege access
- [ ] Dependency scanning
- [ ] Container scanning
- [ ] Signed commits/images

### Reliability

- [ ] Reproducible builds
- [ ] Idempotent deployments
- [ ] Automated rollbacks
- [ ] Health checks
- [ ] Retry logic

### Maintainability

- [ ] Pipeline as Code
- [ ] Modular/reusable workflows
- [ ] Clear documentation
- [ ] Version pinning
- [ ] Notifications

### Monitoring

- [ ] Build metrics
- [ ] Deployment tracking
- [ ] Failure notifications
- [ ] Audit logging

---

**Previous:** [22-security-in-cicd.md](22-security-in-cicd.md)

---

## Congratulations!

You've completed the CI/CD section of the DevOps roadmap. You now understand:

- CI/CD fundamentals and pipeline design
- Jenkins installation, configuration, and pipelines
- Jenkins shared libraries, agents, and backup
- GitHub Actions workflows and advanced features
- Azure DevOps pipelines
- Build automation and testing
- Code quality and security scanning
- Artifact management and versioning
- Containerized builds
- Deployment strategies
- Kubernetes deployments with GitOps
- Security best practices
- Pipeline optimization

**Next Topic:** [08-Ansible](../08-Ansible/)
