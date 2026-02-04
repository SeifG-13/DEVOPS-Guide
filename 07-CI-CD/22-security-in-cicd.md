# Security in CI/CD

## Security Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                 CI/CD Security Layers                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Code Security                         │   │
│   │  • SAST (Static Analysis)                               │   │
│   │  • Secret Detection                                      │   │
│   │  • Code Review                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 Dependency Security                      │   │
│   │  • Vulnerability Scanning                               │   │
│   │  • License Compliance                                   │   │
│   │  • Outdated Package Detection                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Container Security                      │   │
│   │  • Image Scanning                                       │   │
│   │  • Base Image Verification                              │   │
│   │  • Runtime Security                                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 Infrastructure Security                  │   │
│   │  • Secrets Management                                   │   │
│   │  • Access Control                                       │   │
│   │  • Audit Logging                                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Secret Management

### Environment Variables (Basic)

```yaml
# GitHub Actions
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: ./deploy.sh
```

### GitHub Secrets

```yaml
# Organization secrets, repository secrets, environment secrets
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Uses environment-specific secrets
    steps:
      - name: Deploy
        env:
          PROD_API_KEY: ${{ secrets.API_KEY }}  # From 'production' environment
        run: ./deploy.sh
```

### HashiCorp Vault

```yaml
# GitHub Actions with Vault
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: hashicorp/vault-action@v2
        with:
          url: https://vault.example.com
          method: jwt
          role: github-action
          secrets: |
            secret/data/prod/database password | DB_PASSWORD ;
            secret/data/prod/api key | API_KEY

      - name: Deploy
        env:
          DB_PASSWORD: ${{ env.DB_PASSWORD }}
          API_KEY: ${{ env.API_KEY }}
        run: ./deploy.sh
```

### AWS Secrets Manager

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Get secrets
        run: |
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id prod/myapp \
            --query SecretString --output text)
          echo "::add-mask::$SECRET"
          echo "APP_SECRET=$SECRET" >> $GITHUB_ENV
```

### Kubernetes Secrets

```yaml
# External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-secrets
  data:
    - secretKey: database-password
      remoteRef:
        key: secret/data/prod/database
        property: password
```

## Secret Detection

### GitLeaks

```yaml
# GitHub Actions
jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### TruffleHog

```yaml
jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.pull_request.base.sha }}
          head: ${{ github.event.pull_request.head.sha }}
```

### Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

## Static Application Security Testing (SAST)

### CodeQL

```yaml
# .github/workflows/codeql.yml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    strategy:
      matrix:
        language: ['javascript', 'python']
    steps:
      - uses: actions/checkout@v4

      - uses: github/codeql-action/init@v2
        with:
          languages: ${{ matrix.language }}

      - uses: github/codeql-action/autobuild@v2

      - uses: github/codeql-action/analyze@v2
```

### SonarQube Security

```yaml
jobs:
  sonar:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

### Semgrep

```yaml
jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
```

## Dependency Scanning

### npm audit

```yaml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm audit --audit-level=high
```

### Snyk

```yaml
jobs:
  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

### OWASP Dependency-Check

```yaml
jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'MyApp'
          path: '.'
          format: 'HTML'
          args: >
            --failOnCVSS 7
            --enableRetired

      - uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: reports/
```

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
    groups:
      production-dependencies:
        patterns:
          - "*"
        exclude-patterns:
          - "eslint*"
          - "@types/*"
    labels:
      - "dependencies"
      - "security"
```

## Container Security

### Trivy Image Scanning

```yaml
jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### Grype

```yaml
jobs:
  grype:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp .

      - uses: anchore/scan-action@v3
        with:
          image: "myapp"
          fail-build: true
          severity-cutoff: high
```

### Docker Scout

```yaml
jobs:
  scout:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build image
        run: docker build -t myapp .

      - uses: docker/scout-action@v1
        with:
          command: cves
          image: local://myapp
          sarif-file: scout.sarif
          exit-code: true
```

### Secure Dockerfile

```dockerfile
# Use specific version, not latest
FROM node:20.10.0-alpine3.18

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -S -u 1001 -G appgroup appuser

WORKDIR /app

# Copy package files first
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

# Read-only filesystem where possible
EXPOSE 3000

CMD ["node", "index.js"]
```

## Dynamic Application Security Testing (DAST)

### OWASP ZAP

```yaml
jobs:
  zap:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: docker-compose up -d

      - name: ZAP Scan
        uses: zaproxy/action-full-scan@v0.7.0
        with:
          target: 'http://localhost:3000'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-report
          path: report_html.html
```

### Nuclei

```yaml
jobs:
  nuclei:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: docker-compose up -d

      - uses: projectdiscovery/nuclei-action@main
        with:
          target: http://localhost:3000
          templates: cves/
          severity: critical,high
```

## Access Control

### RBAC for CI/CD

```yaml
# GitHub - Branch protection rules
# Settings → Branches → Add rule
#
# Branch name pattern: main
# ✓ Require pull request reviews
# ✓ Require status checks
# ✓ Require signed commits
# ✓ Do not allow force pushes
# ✓ Do not allow deletions
```

### Kubernetes RBAC for CI/CD

```yaml
# Service account for CI/CD
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "patch", "update"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cicd-deployer
subjects:
  - kind: ServiceAccount
    name: cicd-deployer
    namespace: default
```

## Signed Commits and Images

### GPG Signed Commits

```yaml
# Verify signed commits
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Verify commit signatures
        run: |
          git verify-commit HEAD || exit 1
```

### Cosign Image Signing

```yaml
jobs:
  sign:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - uses: sigstore/cosign-installer@v3

      - name: Sign image
        run: |
          cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}
        env:
          COSIGN_EXPERIMENTAL: 1
```

## Complete Security Pipeline

```yaml
name: Security Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  sast:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v2
        with:
          languages: javascript
      - uses: github/codeql-action/autobuild@v2
      - uses: github/codeql-action/analyze@v2

  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm audit --audit-level=high
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  container-scan:
    runs-on: ubuntu-latest
    needs: [secrets-scan, sast, dependency-scan]
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp:${{ github.sha }} .
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  dast:
    runs-on: ubuntu-latest
    needs: container-scan
    steps:
      - uses: actions/checkout@v4
      - run: docker-compose up -d
      - uses: zaproxy/action-baseline@v0.9.0
        with:
          target: 'http://localhost:3000'
```

## Quick Reference

### Security Tools

| Type | Tools |
|------|-------|
| SAST | CodeQL, Semgrep, SonarQube |
| Secrets | GitLeaks, TruffleHog |
| Dependencies | Snyk, npm audit, Dependabot |
| Containers | Trivy, Grype, Docker Scout |
| DAST | OWASP ZAP, Nuclei |

### Security Checks

| Stage | Checks |
|-------|--------|
| Pre-commit | Secrets, linting |
| Build | SAST, dependencies |
| Post-build | Container scan |
| Pre-deploy | DAST, compliance |

### Severity Levels

| Level | Action |
|-------|--------|
| Critical | Block deployment |
| High | Block deployment |
| Medium | Warning, review |
| Low | Informational |

---

**Previous:** [21-kubernetes-deployments.md](21-kubernetes-deployments.md) | **Next:** [23-pipeline-best-practices.md](23-pipeline-best-practices.md)
