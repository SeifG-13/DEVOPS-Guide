# CI/CD Pipeline Security

## Pipeline Security Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE SECURITY                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Why Pipeline Security Matters:                                     │
│  • Pipelines have access to production systems                     │
│  • Credentials and secrets flow through pipelines                  │
│  • Supply chain attacks target CI/CD                               │
│  • Compromised pipeline = compromised everything                   │
│                                                                     │
│  Attack Vectors:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Code Injection   │ Malicious code in build scripts          │   │
│  │ Credential Theft │ Exposed secrets in logs/env              │   │
│  │ Artifact Poison  │ Compromised build outputs                │   │
│  │ Runner Takeover  │ Persistent access via runners            │   │
│  │ Dependency Attack│ Malicious packages/images                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Secure Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURE PIPELINE DESIGN                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │ Source  │───▶│  Build  │───▶│  Test   │───▶│ Deploy  │        │
│  │ Control │    │  Stage  │    │  Stage  │    │  Stage  │        │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘        │
│       │              │              │              │               │
│       ▼              ▼              ▼              ▼               │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │Signed   │    │Isolated │    │Security │    │Approval │        │
│  │Commits  │    │Runners  │    │ Scans   │    │  Gates  │        │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘        │
│                                                                     │
│  Security Controls at Each Stage:                                  │
│  • Source: Signed commits, branch protection, code review         │
│  • Build: Isolated runners, pinned dependencies, SBOM             │
│  • Test: SAST, DAST, dependency scanning, secret scanning         │
│  • Deploy: Image signing, approval gates, audit logging           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GitHub Actions Security

### Secure Workflow Configuration

```yaml
name: Secure CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Restrict permissions at workflow level
permissions:
  contents: read
  packages: write
  security-events: write

jobs:
  build:
    runs-on: ubuntu-latest

    # Restrict permissions at job level
    permissions:
      contents: read
      packages: write

    # Limit environment access
    environment:
      name: production
      url: https://myapp.example.com

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          # Don't persist credentials
          persist-credentials: false

      # Pin actions to commit SHA
      - name: Setup Node
        uses: actions/setup-node@8f152de45cc393bb48ce5d89d36b731f54556e65  # v4.0.0
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci --ignore-scripts

      - name: Build
        run: npm run build
        env:
          # Use secrets, never hardcode
          API_KEY: ${{ secrets.API_KEY }}

      # Scan for secrets in code
      - name: Secret Scanning
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
```

### Branch Protection Rules

```yaml
# Settings via GitHub API
name: Configure Branch Protection

on:
  workflow_dispatch:

jobs:
  configure:
    runs-on: ubuntu-latest
    steps:
      - name: Set branch protection
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.repos.updateBranchProtection({
              owner: context.repo.owner,
              repo: context.repo.repo,
              branch: 'main',
              required_status_checks: {
                strict: true,
                contexts: ['security-scan', 'test']
              },
              enforce_admins: true,
              required_pull_request_reviews: {
                dismiss_stale_reviews: true,
                require_code_owner_reviews: true,
                required_approving_review_count: 2
              },
              restrictions: null,
              required_linear_history: true,
              allow_force_pushes: false,
              allow_deletions: false,
              required_signatures: true
            });
```

### OIDC Authentication (No Long-lived Secrets)

```yaml
name: Deploy with OIDC

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # AWS - No access keys needed
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-east-1

      # GCP - No service account keys
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
          service_account: github-actions@project.iam.gserviceaccount.com

      # Azure - No client secrets
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## GitLab CI Security

### Secure Pipeline Configuration

```yaml
# .gitlab-ci.yml
default:
  image: alpine:latest

# Define security stages
stages:
  - validate
  - security
  - build
  - test
  - deploy

# Use protected variables
variables:
  SECURE_LOG_LEVEL: "debug"

# Secret detection
secret_detection:
  stage: security
  image: registry.gitlab.com/gitlab-org/security-products/analyzers/secrets:latest
  script:
    - /analyzer run
  artifacts:
    reports:
      secret_detection: gl-secret-detection-report.json

# SAST scanning
sast:
  stage: security
  image: registry.gitlab.com/gitlab-org/security-products/analyzers/semgrep:latest
  script:
    - /analyzer run
  artifacts:
    reports:
      sast: gl-sast-report.json

# Container scanning
container_scanning:
  stage: security
  image: registry.gitlab.com/gitlab-org/security-products/analyzers/container-scanning:latest
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  script:
    - gtcs scan
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json

# Secure deployment
deploy_production:
  stage: deploy
  script:
    - ./deploy.sh
  environment:
    name: production
    url: https://myapp.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  # Require approval
  allow_failure: false
```

### Runner Security

```toml
# /etc/gitlab-runner/config.toml
concurrent = 4
check_interval = 0

[[runners]]
  name = "secure-runner"
  url = "https://gitlab.com/"
  token = "TOKEN"
  executor = "docker"

  [runners.docker]
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = true
    volumes = ["/cache"]
    # Restrict capabilities
    cap_drop = ["ALL"]
    # Read-only root filesystem
    read_only = true
    # Network isolation
    network_mode = "bridge"
    # Resource limits
    memory = "2g"
    cpus = "2"
```

---

## Jenkins Security

### Pipeline Security

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            // Run as non-root
            args '-u root:root'
        }
    }

    options {
        // Prevent concurrent builds
        disableConcurrentBuilds()
        // Timeout builds
        timeout(time: 1, unit: 'HOURS')
        // Don't keep old builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        // Use credentials binding
        DB_CREDS = credentials('database-credentials')
        // Mask secrets in logs
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
    }

    stages {
        stage('Security Scan') {
            steps {
                // Run security scans
                sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL .'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                submitter "deployers"
            }
            steps {
                // Use short-lived credentials
                withAWS(role: 'DeployRole', roleAccount: '123456789') {
                    sh './deploy.sh'
                }
            }
        }
    }

    post {
        always {
            // Clean workspace
            cleanWs()
        }
    }
}
```

### Security Configuration

```groovy
// Jenkins configuration as code
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: "ldap.example.com"
          rootDN: "dc=example,dc=com"
          userSearchBase: "ou=users"

  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            entries:
              - group: "admins"
          - name: "developer"
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Read"
            entries:
              - group: "developers"

  globalNodeProperties:
    - envVars:
        env:
          - key: "JAVA_OPTS"
            value: "-Djenkins.security.SystemReadPermission=true"
```

---

## Secret Management in Pipelines

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRET HANDLING PRINCIPLES                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DO:                                                                │
│  ✓ Use platform secret management (GitHub Secrets, GitLab CI Vars) │
│  ✓ Use external secret managers (Vault, AWS SM)                    │
│  ✓ Use OIDC for cloud authentication                               │
│  ✓ Rotate secrets regularly                                        │
│  ✓ Audit secret access                                             │
│                                                                     │
│  DON'T:                                                             │
│  ✗ Hardcode secrets in code or configs                             │
│  ✗ Log secrets (even accidentally)                                 │
│  ✗ Store secrets in environment variables long-term                │
│  ✗ Share secrets across environments                               │
│  ✗ Use long-lived credentials                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### HashiCorp Vault Integration

```yaml
# GitHub Actions with Vault
name: Deploy with Vault

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Import Secrets from Vault
        uses: hashicorp/vault-action@v2
        with:
          url: https://vault.example.com
          method: jwt
          role: github-actions
          secrets: |
            secret/data/myapp/prod db_password | DB_PASSWORD ;
            secret/data/myapp/prod api_key | API_KEY

      - name: Deploy
        run: |
          ./deploy.sh
        env:
          DB_PASSWORD: ${{ env.DB_PASSWORD }}
          API_KEY: ${{ env.API_KEY }}
```

---

## Artifact Security

### Image Signing with Cosign

```yaml
# GitHub Actions - Sign and Verify Images
name: Build and Sign Image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write  # For keyless signing

    steps:
      - uses: actions/checkout@v4

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        id: build
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Sign Image (Keyless)
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

      - name: Verify Image
        run: |
          cosign verify \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

### SBOM Generation

```yaml
# Generate SBOM with Syft
name: Generate SBOM

on:
  push:
    branches: [main]

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Attest SBOM
        uses: actions/attest-sbom@v1
        with:
          subject-path: sbom.spdx.json
          sbom-format: spdx-json
```

---

## Pipeline Hardening Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PIPELINE SECURITY CHECKLIST                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Source Control:                                                    │
│  □ Enable branch protection rules                                  │
│  □ Require signed commits                                          │
│  □ Require code reviews (2+ approvers)                             │
│  □ Protect secrets with access controls                            │
│                                                                     │
│  Build Environment:                                                 │
│  □ Use ephemeral runners                                           │
│  □ Pin action/image versions to SHA                                │
│  □ Disable privileged mode                                         │
│  □ Apply resource limits                                           │
│                                                                     │
│  Secrets:                                                           │
│  □ Use OIDC instead of long-lived credentials                      │
│  □ Integrate with secret manager                                   │
│  □ Mask secrets in logs                                            │
│  □ Rotate secrets regularly                                        │
│                                                                     │
│  Scanning:                                                          │
│  □ Run SAST on every PR                                            │
│  □ Run dependency scanning                                         │
│  □ Run secret scanning                                             │
│  □ Scan container images                                           │
│                                                                     │
│  Deployment:                                                        │
│  □ Require manual approval for production                          │
│  □ Sign artifacts and verify signatures                            │
│  □ Generate and attest SBOM                                        │
│  □ Audit all deployments                                           │
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
│  Key Security Principles:                                           │
│  • Least privilege for pipeline permissions                        │
│  • Ephemeral and isolated build environments                       │
│  • No long-lived credentials (use OIDC)                            │
│  • Sign and verify all artifacts                                   │
│  • Audit everything                                                │
│                                                                     │
│  Platform Features:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ GitHub    │ OIDC, permissions, environments, secrets        │   │
│  │ GitLab    │ Protected variables, runners, security scanning │   │
│  │ Jenkins   │ Credentials binding, role-based access          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Essential Tools:                                                   │
│  • Cosign - Image signing                                          │
│  • Syft - SBOM generation                                          │
│  • TruffleHog - Secret scanning                                    │
│  • Trivy - Vulnerability scanning                                  │
│                                                                     │
│  Next: Learn about runtime security                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
