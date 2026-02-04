# GitHub Actions Advanced

## Matrix Builds

### Basic Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

### Matrix with Include/Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]
    include:
      # Add specific configuration
      - os: ubuntu-latest
        node: 20
        experimental: true
    exclude:
      # Skip specific combination
      - os: windows-latest
        node: 16

steps:
  - run: echo "Node ${{ matrix.node }} on ${{ matrix.os }}"
  - run: echo "Experimental: ${{ matrix.experimental || false }}"
```

### Fail-Fast and Max-Parallel

```yaml
strategy:
  fail-fast: false  # Don't cancel other jobs on failure
  max-parallel: 2   # Run 2 jobs at a time
  matrix:
    version: [10, 12, 14, 16, 18, 20]
```

## Reusable Workflows

### Creating Reusable Workflow

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version'
        required: false
        default: '20'
        type: string
      environment:
        description: 'Deployment environment'
        required: true
        type: string
    secrets:
      npm-token:
        description: 'NPM token'
        required: true
    outputs:
      artifact-name:
        description: 'Name of the uploaded artifact'
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.upload.outputs.name }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}

      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.npm-token }}

      - run: npm run build

      - id: upload
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ inputs.environment }}
          path: dist/
```

### Calling Reusable Workflow

```yaml
# .github/workflows/main.yml
name: Main Pipeline

on:
  push:
    branches: [main]

jobs:
  build-staging:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
      environment: staging
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  build-production:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
      environment: production
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  deploy:
    needs: [build-staging, build-production]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build-staging.outputs.artifact-name }}"
```

### Reusable Workflow from Another Repository

```yaml
jobs:
  call-external:
    uses: organization/repo/.github/workflows/workflow.yml@main
    with:
      config-path: ./config.json
    secrets: inherit  # Pass all secrets
```

## Composite Actions

### Creating Composite Action

```yaml
# .github/actions/setup-project/action.yml
name: 'Setup Project'
description: 'Setup Node.js and install dependencies'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  cache:
    description: 'Enable cache'
    required: false
    default: 'true'

outputs:
  cache-hit:
    description: 'Whether cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache dependencies
      id: cache
      if: inputs.cache == 'true'
      uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

    - name: Install dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      shell: bash
      run: npm ci
```

### Using Composite Action

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup project
        uses: ./.github/actions/setup-project
        with:
          node-version: '20'
          cache: 'true'

      - run: npm run build
```

## Environments and Deployments

### Environment Configuration

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://example.com
    steps:
      - run: ./deploy.sh production
```

### Environment Protection Rules

```
Repository Settings → Environments → production

Protection rules:
✓ Required reviewers: user1, user2
✓ Wait timer: 15 minutes
✓ Deployment branches: main only
```

### Environment Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy
        env:
          # Environment-specific secret
          API_KEY: ${{ secrets.PROD_API_KEY }}
        run: ./deploy.sh
```

## Workflow Dispatch and Inputs

### Manual Trigger with Inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
      version:
        description: 'Version to deploy'
        required: true
        type: string
        default: 'latest'
      dry-run:
        description: 'Perform dry run'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
          echo "Dry run: ${{ inputs.dry-run }}"

          if [ "${{ inputs.dry-run }}" = "true" ]; then
            echo "DRY RUN - Would deploy ${{ inputs.version }} to ${{ inputs.environment }}"
          else
            ./deploy.sh ${{ inputs.environment }} ${{ inputs.version }}
          fi
```

## Concurrency Control

### Limit Concurrent Runs

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # Cancel previous runs

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
```

### Environment-Based Concurrency

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: false  # Queue deployments
    steps:
      - run: ./deploy.sh
```

## Conditional Execution

### Complex Conditions

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    if: |
      github.event_name == 'push' ||
      (github.event_name == 'pull_request' &&
       !contains(github.event.pull_request.labels.*.name, 'skip-ci'))

    steps:
      - name: Only on main
        if: github.ref == 'refs/heads/main'
        run: echo "On main branch"

      - name: Only for tags
        if: startsWith(github.ref, 'refs/tags/')
        run: echo "Tagged release"

      - name: Only if file changed
        if: contains(github.event.commits.*.modified, 'package.json')
        run: npm install

      - name: Continue on error
        continue-on-error: true
        run: ./optional-step.sh

      - name: Always run
        if: always()
        run: ./cleanup.sh

      - name: On failure only
        if: failure()
        run: ./notify-failure.sh
```

### Path Filters

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
      - '!src/**/*.md'
  pull_request:
    paths-ignore:
      - 'docs/**'
      - '*.md'
```

## Self-Hosted Runners

### Runner Labels

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64, gpu]
    steps:
      - uses: actions/checkout@v4
      - run: ./train-model.sh
```

### Runner Groups

```yaml
jobs:
  deploy:
    runs-on:
      group: production-runners
      labels: [self-hosted, linux]
    steps:
      - run: ./deploy.sh
```

## Caching Strategies

### Dependency Caching

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### Build Caching

```yaml
- uses: actions/cache@v4
  with:
    path: .next/cache
    key: nextjs-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.jsx', '**/*.ts', '**/*.tsx') }}
    restore-keys: |
      nextjs-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-
      nextjs-${{ runner.os }}-
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

## Artifacts and Outputs

### Upload and Download Artifacts

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - uses: actions/checkout@v4

      - id: version
        run: echo "value=$(cat VERSION)" >> $GITHUB_OUTPUT

      - run: npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: build-artifacts
          path: |
            dist/
            !dist/**/*.map
          retention-days: 5

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-artifacts
          path: dist/

      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

### Multiple Artifacts

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: test-results-${{ matrix.os }}
    path: test-results/

# Later job
- uses: actions/download-artifact@v4
  with:
    pattern: test-results-*
    merge-multiple: true
    path: all-results/
```

## GitHub Container Registry

### Build and Push to GHCR

```yaml
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

## Complete Advanced Workflow

```yaml
name: Advanced CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
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
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - run: echo "Deploying to staging"

  deploy-production:
    if: github.event_name == 'workflow_dispatch' && inputs.environment == 'production'
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: echo "Deploying to production"
```

## Quick Reference

### Workflow Triggers

| Event | Use Case |
|-------|----------|
| `workflow_call` | Reusable workflow |
| `workflow_dispatch` | Manual trigger |
| `repository_dispatch` | External trigger |
| `schedule` | Cron jobs |

### Matrix Options

| Option | Description |
|--------|-------------|
| `fail-fast` | Cancel all on first failure |
| `max-parallel` | Limit concurrent jobs |
| `include` | Add extra combinations |
| `exclude` | Skip combinations |

### Concurrency Options

| Option | Description |
|--------|-------------|
| `group` | Concurrency group name |
| `cancel-in-progress` | Cancel running jobs |

---

**Previous:** [12-github-actions-basics.md](12-github-actions-basics.md) | **Next:** [14-azure-devops.md](14-azure-devops.md)
