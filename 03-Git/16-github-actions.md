# GitHub Actions

## What is GitHub Actions?

GitHub Actions is a CI/CD platform built into GitHub for automating workflows.

```
┌─────────────────────────────────────────────────────────────────┐
│                     GitHub Actions                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Push ──► Trigger ──► Jobs ──► Steps ──► Results               │
│                          │                                       │
│                    ┌─────┴─────┐                                 │
│                    │  Runner   │                                 │
│                    │ (ubuntu)  │                                 │
│                    └───────────┘                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Key Concepts

| Concept | Description |
|---------|-------------|
| Workflow | Automated process (YAML file) |
| Event | Trigger (push, PR, schedule) |
| Job | Set of steps on same runner |
| Step | Individual task |
| Action | Reusable unit |
| Runner | Machine that runs jobs |

## Basic Workflow Structure

```yaml
# .github/workflows/ci.yml

name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

## Triggers (Events)

### Push and Pull Request

```yaml
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - '!src/docs/**'
    tags:
      - 'v*'

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

### Schedule (Cron)

```yaml
on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight
    - cron: '0 */6 * * *'  # Every 6 hours
```

### Manual Trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```

### Other Events

```yaml
on:
  release:
    types: [published]

  issues:
    types: [opened, labeled]

  workflow_call:  # Reusable workflow

  repository_dispatch:
    types: [deploy]
```

## Jobs

### Basic Job

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello"
```

### Matrix Strategy

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

### Job Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Conditional Execution

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

## Steps

### Run Commands

```yaml
steps:
  - name: Single command
    run: echo "Hello"

  - name: Multi-line script
    run: |
      echo "Line 1"
      echo "Line 2"
      npm install
      npm test

  - name: With working directory
    run: npm test
    working-directory: ./app

  - name: With shell
    run: Get-Process
    shell: pwsh
```

### Use Actions

```yaml
steps:
  # Official action
  - uses: actions/checkout@v4

  # With inputs
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'

  # Specific version
  - uses: actions/checkout@v4.1.0

  # From another repo
  - uses: owner/repo@v1

  # Local action
  - uses: ./.github/actions/my-action
```

## Variables and Secrets

### Environment Variables

```yaml
env:
  NODE_ENV: production

jobs:
  build:
    env:
      API_URL: https://api.example.com
    steps:
      - name: Use env var
        run: echo $NODE_ENV
        env:
          STEP_VAR: value
```

### Secrets

```yaml
steps:
  - name: Deploy
    run: ./deploy.sh
    env:
      API_KEY: ${{ secrets.API_KEY }}

  - name: Docker login
    run: docker login -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASS }}
```

### GitHub Context

```yaml
steps:
  - run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
```

## Artifacts

### Upload Artifact

```yaml
steps:
  - name: Build
    run: npm run build

  - name: Upload artifact
    uses: actions/upload-artifact@v4
    with:
      name: build-output
      path: dist/
      retention-days: 5
```

### Download Artifact

```yaml
jobs:
  deploy:
    needs: build
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
```

## Caching

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache node modules
    uses: actions/cache@v4
    with:
      path: node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  - name: Install dependencies
    run: npm ci
```

## Common Workflows

### Node.js CI

```yaml
name: Node.js CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

### Docker Build and Push

```yaml
name: Docker

on:
  push:
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: user/app:${{ github.ref_name }}
```

### Deploy to Cloud

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}
          aws-region: us-east-1

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://my-bucket
```

## Running Workflows

```bash
# List workflows
gh workflow list

# View workflow
gh workflow view ci.yml

# Run workflow manually
gh workflow run ci.yml

# Run with inputs
gh workflow run deploy.yml -f environment=production

# View runs
gh run list

# View specific run
gh run view 123456

# Watch run
gh run watch 123456
```

## Quick Reference

| Syntax | Purpose |
|--------|---------|
| `on: push` | Trigger on push |
| `runs-on: ubuntu-latest` | Runner OS |
| `uses: action@v1` | Use action |
| `run: command` | Run shell command |
| `with:` | Action inputs |
| `env:` | Environment variables |
| `secrets.NAME` | Access secret |
| `needs: job` | Job dependency |
| `if: condition` | Conditional execution |

---

**Previous:** [15-pull-requests.md](15-pull-requests.md) | **Next:** [17-github-features.md](17-github-features.md)
