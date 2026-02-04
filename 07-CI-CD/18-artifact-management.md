# Artifact Management

## What are Build Artifacts?

Build artifacts are the output files produced during the build process that need to be stored, versioned, and distributed.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Artifact Lifecycle                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Build Process                                                 │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────┐                                              │
│   │  Artifacts  │  JAR, WAR, Docker images, binaries           │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │   Storage   │  Artifact Repository                         │
│   │             │  (Nexus, Artifactory, Registry)              │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ├─────────────┬─────────────┐                         │
│          ▼             ▼             ▼                         │
│   ┌───────────┐ ┌───────────┐ ┌───────────┐                   │
│   │    Dev    │ │  Staging  │ │Production │                   │
│   └───────────┘ └───────────┘ └───────────┘                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Types of Artifacts

| Type | Examples | Repository Type |
|------|----------|-----------------|
| Java | JAR, WAR, EAR | Maven (Nexus, Artifactory) |
| JavaScript | npm packages | npm Registry |
| Python | wheels, sdist | PyPI, Artifactory |
| Docker | Container images | Docker Registry |
| Binaries | executables | Generic repository |
| Helm | Charts | Helm Repository |

## Artifact Repositories

### Nexus Repository

```yaml
# Docker Compose setup
services:
  nexus:
    image: sonatype/nexus3:latest
    ports:
      - "8081:8081"
    volumes:
      - nexus-data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms512m -Xmx2g

volumes:
  nexus-data:
```

### JFrog Artifactory

```yaml
# Docker Compose setup
services:
  artifactory:
    image: releases-docker.jfrog.io/jfrog/artifactory-oss:latest
    ports:
      - "8082:8082"
    volumes:
      - artifactory-data:/var/opt/jfrog/artifactory

volumes:
  artifactory-data:
```

## Maven Artifacts

### Publishing to Nexus

```xml
<!-- pom.xml -->
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>http://nexus.example.com/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>http://nexus.example.com/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

```xml
<!-- settings.xml -->
<servers>
  <server>
    <id>nexus-releases</id>
    <username>${env.NEXUS_USER}</username>
    <password>${env.NEXUS_PASSWORD}</password>
  </server>
  <server>
    <id>nexus-snapshots</id>
    <username>${env.NEXUS_USER}</username>
    <password>${env.NEXUS_PASSWORD}</password>
  </server>
</servers>
```

### Maven Deploy in CI

```yaml
# GitHub Actions
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'

      - name: Deploy to Nexus
        run: mvn deploy -DskipTests
        env:
          NEXUS_USER: ${{ secrets.NEXUS_USER }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
```

## npm Artifacts

### Publishing to npm Registry

```json
// package.json
{
  "name": "@myorg/my-package",
  "version": "1.0.0",
  "publishConfig": {
    "registry": "https://npm.example.com/",
    "access": "public"
  }
}
```

```yaml
# .npmrc
//npm.example.com/:_authToken=${NPM_TOKEN}
registry=https://npm.example.com/
```

### npm Publish in CI

```yaml
# GitHub Actions
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'

      - run: npm ci
      - run: npm run build
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### GitHub Packages (npm)

```yaml
# GitHub Actions
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://npm.pkg.github.com'

      - run: npm ci
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Docker Images

### Publishing to Docker Hub

```yaml
# GitHub Actions
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: myorg/myapp
          tags: |
            type=semver,pattern={{version}}
            type=sha

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
```

### GitHub Container Registry

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### Amazon ECR

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - uses: aws-actions/amazon-ecr-login@v2
        id: login-ecr

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.sha }}
```

## Python Packages

### Publishing to PyPI

```yaml
# GitHub Actions
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - run: pip install build twine
      - run: python -m build
      - run: twine upload dist/*
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_TOKEN }}
```

### Publishing to Private PyPI

```yaml
- run: twine upload --repository-url https://pypi.example.com/ dist/*
  env:
    TWINE_USERNAME: ${{ secrets.PYPI_USER }}
    TWINE_PASSWORD: ${{ secrets.PYPI_PASS }}
```

## CI Pipeline Artifacts

### GitHub Actions Artifacts

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: |
            dist/
            !dist/**/*.map
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: ./deploy.sh
```

### Jenkins Artifacts

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'npm run build'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/**/*',
                        fingerprint: true,
                        onlyIfSuccessful: true
                }
            }
        }
    }
}
```

### Azure DevOps Artifacts

```yaml
steps:
  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: '$(Build.ArtifactStagingDirectory)'
      artifactName: 'drop'

  # Or Pipeline Artifacts
  - publish: $(Build.ArtifactStagingDirectory)
    artifact: drop
```

## Versioning Strategies

### Semantic Versioning

```
MAJOR.MINOR.PATCH

1.0.0 → 1.0.1 (patch - bug fix)
1.0.0 → 1.1.0 (minor - new feature)
1.0.0 → 2.0.0 (major - breaking change)

Pre-release: 1.0.0-alpha.1, 1.0.0-beta.1, 1.0.0-rc.1
Build metadata: 1.0.0+build.123
```

### Automated Versioning

```yaml
# Using semantic-release
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

```json
// package.json or .releaserc
{
  "release": {
    "branches": ["main"],
    "plugins": [
      "@semantic-release/commit-analyzer",
      "@semantic-release/release-notes-generator",
      "@semantic-release/npm",
      "@semantic-release/github"
    ]
  }
}
```

### Git-based Versioning

```yaml
# Generate version from git
- name: Get version
  id: version
  run: |
    VERSION=$(git describe --tags --always)
    echo "version=$VERSION" >> $GITHUB_OUTPUT

- name: Build with version
  run: docker build --build-arg VERSION=${{ steps.version.outputs.version }} .
```

## Artifact Cleanup

### Retention Policies

```yaml
# GitHub Actions artifact retention
- uses: actions/upload-artifact@v4
  with:
    name: build
    path: dist/
    retention-days: 30  # Auto-delete after 30 days
```

### Nexus Cleanup Policies

```json
// Cleanup policy in Nexus
{
  "name": "cleanup-old-snapshots",
  "format": "maven2",
  "mode": "delete",
  "criteria": {
    "lastBlobUpdated": "30",
    "lastDownloaded": "7",
    "preRelease": "PRERELEASES"
  }
}
```

### Docker Image Cleanup

```bash
# Delete old images from registry
# Keep last 10 tags
REPO="myrepo/myapp"
TAGS=$(docker hub list-tags $REPO | sort -V | head -n -10)
for tag in $TAGS; do
    docker hub delete $REPO:$tag
done
```

## Complete Artifact Pipeline

```yaml
name: Build and Publish

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get version
        id: version
        run: |
          if [[ "$GITHUB_REF" == refs/tags/* ]]; then
            VERSION=${GITHUB_REF#refs/tags/v}
          else
            VERSION=$(git describe --tags --always)
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: build-${{ steps.version.outputs.version }}
          path: dist/
          retention-days: 30

  docker:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          name: build-${{ needs.build.outputs.version }}
          path: dist/

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
            type=semver,pattern={{version}}
            type=sha

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}

  npm:
    if: startsWith(github.ref, 'refs/tags/')
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          name: build-${{ needs.build.outputs.version }}
          path: dist/

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'

      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## Quick Reference

### Artifact Registries

| Type | Registry | Auth Method |
|------|----------|-------------|
| Docker | Docker Hub, GHCR, ECR | Token/IAM |
| Maven | Nexus, Artifactory | Username/Password |
| npm | npmjs, GitHub Packages | Token |
| Python | PyPI, Artifactory | Token |
| Helm | Harbor, ChartMuseum | Token |

### Versioning Commands

| Tool | Command |
|------|---------|
| npm | `npm version patch/minor/major` |
| Maven | `mvn versions:set -DnewVersion=1.0.0` |
| Python | Update `pyproject.toml` |
| Docker | Use tags |

### Artifact Retention

| Environment | Retention |
|-------------|-----------|
| Development | 7 days |
| Staging | 30 days |
| Production | 1 year+ |
| Releases | Permanent |

---

**Previous:** [17-code-quality.md](17-code-quality.md) | **Next:** [19-containerized-builds.md](19-containerized-builds.md)
