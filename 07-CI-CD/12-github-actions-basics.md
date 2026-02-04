# GitHub Actions Basics

## What is GitHub Actions?

GitHub Actions is a CI/CD platform integrated directly into GitHub repositories.

```
┌─────────────────────────────────────────────────────────────────┐
│                 GitHub Actions Overview                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   GitHub Repository                                             │
│   ├── .github/                                                  │
│   │   └── workflows/                                           │
│   │       ├── ci.yml                                           │
│   │       ├── deploy.yml                                       │
│   │       └── release.yml                                      │
│   ├── src/                                                      │
│   └── ...                                                       │
│                                                                  │
│   Workflow Structure:                                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Workflow (ci.yml)                                       │   │
│   │  ├── Event (push, pull_request)                         │   │
│   │  └── Jobs                                               │   │
│   │      ├── Job 1 (build)                                  │   │
│   │      │   └── Steps                                      │   │
│   │      │       ├── Checkout                               │   │
│   │      │       ├── Setup Node                             │   │
│   │      │       └── Run tests                              │   │
│   │      └── Job 2 (deploy)                                 │   │
│   │          └── Steps                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Workflow Basics

### Minimal Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test
```

### Complete Workflow Structure

```yaml
name: CI/CD Pipeline

# Triggers
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger

# Environment variables
env:
  NODE_VERSION: '20'

# Jobs
jobs:
  build:
    name: Build and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

## Event Triggers

### Push and Pull Request

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
    paths:
      - 'src/**'
      - '!src/**/*.md'
    tags:
      - 'v*'

  pull_request:
    branches:
      - main
    types: [opened, synchronize, reopened]
```

### Schedule (Cron)

```yaml
on:
  schedule:
    # Every day at 2 AM UTC
    - cron: '0 2 * * *'
    # Every Monday at 9 AM UTC
    - cron: '0 9 * * 1'
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
      debug:
        description: 'Enable debug mode'
        required: false
        type: boolean
        default: false
```

### Other Events

```yaml
on:
  # Issue events
  issues:
    types: [opened, labeled]

  # Release events
  release:
    types: [published]

  # Repository dispatch (external trigger)
  repository_dispatch:
    types: [deploy]

  # Workflow call (reusable workflows)
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
```

## Jobs and Steps

### Job Configuration

```yaml
jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest
    timeout-minutes: 30

    # Job outputs
    outputs:
      version: ${{ steps.version.outputs.value }}

    # Environment
    environment:
      name: production
      url: https://myapp.example.com

    # Concurrency
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - uses: actions/checkout@v4

      - name: Get version
        id: version
        run: echo "value=$(cat VERSION)" >> $GITHUB_OUTPUT
```

### Step Types

```yaml
steps:
  # Use an action
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0

  # Run a command
  - name: Build
    run: npm run build

  # Multi-line command
  - name: Deploy
    run: |
      echo "Deploying..."
      ./deploy.sh
      echo "Done"

  # Working directory
  - name: Build frontend
    working-directory: ./frontend
    run: npm run build

  # Environment variables
  - name: Test with env
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    run: npm test

  # Conditional step
  - name: Deploy to prod
    if: github.ref == 'refs/heads/main'
    run: ./deploy-prod.sh
```

## Runners

### GitHub-Hosted Runners

```yaml
jobs:
  linux:
    runs-on: ubuntu-latest  # or ubuntu-22.04, ubuntu-20.04

  windows:
    runs-on: windows-latest  # or windows-2022, windows-2019

  macos:
    runs-on: macos-latest  # or macos-14, macos-13
```

### Self-Hosted Runners

```yaml
jobs:
  build:
    runs-on: self-hosted

  # With labels
  build-special:
    runs-on: [self-hosted, linux, x64, gpu]
```

## Environment Variables and Secrets

### Environment Variables

```yaml
# Workflow level
env:
  APP_NAME: myapp
  NODE_ENV: production

jobs:
  build:
    # Job level
    env:
      BUILD_TYPE: release

    steps:
      - name: Build
        # Step level
        env:
          DEBUG: true
        run: |
          echo "Building $APP_NAME"
          echo "Build type: $BUILD_TYPE"
          echo "Debug: $DEBUG"
```

### Using Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: aws s3 sync ./dist s3://my-bucket

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
```

### GitHub Context Variables

```yaml
steps:
  - name: Show context
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"
```

## Actions

### Using Actions

```yaml
steps:
  # Official action with version
  - uses: actions/checkout@v4

  # Action with inputs
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'

  # Third-party action
  - uses: docker/build-push-action@v5
    with:
      push: true
      tags: user/app:latest

  # Action from same repository
  - uses: ./.github/actions/my-action

  # Action from different repository
  - uses: owner/repo/path@ref
```

### Common Actions

```yaml
# Checkout
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Full history
    token: ${{ secrets.PAT }}  # For private repos

# Setup languages
- uses: actions/setup-node@v4
  with:
    node-version: '20'
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
- uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
- uses: actions/setup-go@v5
  with:
    go-version: '1.21'

# Caching
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}

# Upload/Download artifacts
- uses: actions/upload-artifact@v4
  with:
    name: build
    path: dist/
- uses: actions/download-artifact@v4
  with:
    name: build
```

## Job Dependencies

### Sequential Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  test:
    needs: build  # Runs after build
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: [build, test]  # Runs after both
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Passing Data Between Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - id: version
        run: echo "value=1.0.${{ github.run_number }}" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

## Complete Example

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm test

  build:
    name: Build
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - uses: docker/build-push-action@v5
        with:
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.example.com
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image-tag }}"
```

## Quick Reference

### Workflow Syntax

| Element | Description |
|---------|-------------|
| `name` | Workflow name |
| `on` | Trigger events |
| `env` | Environment variables |
| `jobs` | Job definitions |
| `runs-on` | Runner selection |
| `steps` | Job steps |
| `uses` | Use an action |
| `run` | Run a command |

### Common Events

| Event | Trigger |
|-------|---------|
| `push` | Code pushed |
| `pull_request` | PR opened/updated |
| `schedule` | Cron schedule |
| `workflow_dispatch` | Manual trigger |
| `release` | Release published |

---

**Previous:** [11-jenkins-backup-restore.md](11-jenkins-backup-restore.md) | **Next:** [13-github-actions-advanced.md](13-github-actions-advanced.md)
