# Dependency Scanning (SCA)

## What is SCA?

Software Composition Analysis (SCA) identifies vulnerabilities in third-party dependencies.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SCA OVERVIEW                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Modern applications use 80%+ third-party code                     │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │   Your Code (20%)        Third-Party Dependencies (80%)     │   │
│  │   ┌──────────┐          ┌─────────────────────────────┐    │   │
│  │   │          │          │ Libraries, Frameworks,      │    │   │
│  │   │  Custom  │◀────────▶│ Packages (npm, pip, maven)  │    │   │
│  │   │   Code   │          │                             │    │   │
│  │   │          │          │ May contain vulnerabilities │    │   │
│  │   └──────────┘          └─────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  SCA identifies:                                                    │
│  • Known vulnerabilities (CVEs)                                    │
│  • Outdated dependencies                                           │
│  • License compliance issues                                       │
│  • Transitive dependencies                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Vulnerability Databases

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VULNERABILITY DATABASES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  NVD (National Vulnerability Database)                              │
│  • Official US government database                                 │
│  • CVE identifiers                                                 │
│  • CVSS scores                                                     │
│                                                                     │
│  GitHub Advisory Database                                           │
│  • Curated security advisories                                     │
│  • Package-specific                                                │
│  • GHSA identifiers                                                │
│                                                                     │
│  OSV (Open Source Vulnerabilities)                                  │
│  • Google's open database                                          │
│  • Machine-readable format                                         │
│  • Cross-ecosystem                                                 │
│                                                                     │
│  Snyk Vulnerability Database                                        │
│  • Commercial + open                                               │
│  • Additional context                                              │
│  • Remediation advice                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## SCA Tools

### Snyk

```bash
# Install
npm install -g snyk
# or
brew install snyk

# Authenticate
snyk auth

# Test project
snyk test

# Monitor (continuous)
snyk monitor

# Fix vulnerabilities
snyk fix

# Output formats
snyk test --json > snyk-results.json
snyk test --sarif > snyk-results.sarif

# Container scanning
snyk container test myimage:tag

# IaC scanning
snyk iac test ./terraform/
```

### Snyk CI/CD Integration

```yaml
# GitHub Actions
name: Snyk Security
on: [push, pull_request]

jobs:
  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: snyk.sarif
```

### OWASP Dependency-Check

```bash
# Download
wget https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.0/dependency-check-8.4.0-release.zip
unzip dependency-check-8.4.0-release.zip

# Run scan
./dependency-check/bin/dependency-check.sh \
  --project "My Project" \
  --scan ./src \
  --format HTML \
  --out ./reports

# Docker
docker run --rm \
  -v $(pwd):/src \
  -v $(pwd)/reports:/report \
  owasp/dependency-check \
  --scan /src \
  --format "ALL" \
  --out /report
```

### Dependency-Check CI/CD

```yaml
# GitHub Actions
- name: OWASP Dependency Check
  uses: dependency-check/Dependency-Check_Action@main
  with:
    project: 'My Project'
    path: '.'
    format: 'HTML'
    args: >
      --failOnCVSS 7
      --enableRetired

- name: Upload Report
  uses: actions/upload-artifact@v3
  with:
    name: dependency-check-report
    path: reports/
```

---

## GitHub Dependabot

### Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  # npm dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"

  # Python dependencies
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"

  # Docker dependencies
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"

  # Terraform providers
  - package-ecosystem: "terraform"
    directory: "/terraform"
    schedule:
      interval: "weekly"
```

### Security Alerts

```yaml
# Enable in repository settings
# Security → Dependabot alerts → Enable

# Auto-merge security updates
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot Auto-merge
on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  dependabot:
    runs-on: ubuntu-latest
    if: ${{ github.actor == 'dependabot[bot]' }}
    steps:
      - name: Fetch Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@v1

      - name: Auto-merge patch updates
        if: ${{ steps.metadata.outputs.update-type == 'version-update:semver-patch' }}
        run: gh pr merge --auto --merge "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Trivy (Dependencies)

```bash
# Install
brew install trivy
# or
apt-get install trivy

# Scan filesystem for vulnerabilities
trivy fs .

# Scan specific lockfile
trivy fs --scanners vuln package-lock.json

# Scan with severity filter
trivy fs --severity CRITICAL,HIGH .

# Output formats
trivy fs --format json -o results.json .
trivy fs --format sarif -o results.sarif .

# Ignore unfixed vulnerabilities
trivy fs --ignore-unfixed .
```

### Trivy CI/CD

```yaml
# GitHub Actions
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    scan-ref: '.'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

---

## Package Manager Audits

### npm

```bash
# Audit dependencies
npm audit

# Audit with JSON output
npm audit --json

# Fix vulnerabilities
npm audit fix

# Force fix (breaking changes)
npm audit fix --force

# Production only
npm audit --production
```

### pip (Python)

```bash
# Install safety
pip install safety

# Audit dependencies
safety check

# With requirements file
safety check -r requirements.txt

# JSON output
safety check --json

# pip-audit (newer alternative)
pip install pip-audit
pip-audit
```

### Go

```bash
# Native vulnerability check
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...

# With output format
govulncheck -json ./...
```

### Maven (Java)

```xml
<!-- pom.xml - OWASP Dependency Check plugin -->
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>8.4.0</version>
  <executions>
    <execution>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <failBuildOnCVSS>7</failBuildOnCVSS>
  </configuration>
</plugin>
```

```bash
mvn dependency-check:check
```

---

## SBOM (Software Bill of Materials)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SBOM                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SBOM = Complete inventory of software components                   │
│                                                                     │
│  Formats:                                                           │
│  • SPDX (ISO standard)                                             │
│  • CycloneDX (OWASP)                                               │
│  • SWID (ISO standard)                                             │
│                                                                     │
│  Benefits:                                                          │
│  • Track all dependencies                                          │
│  • Respond quickly to new CVEs                                     │
│  • License compliance                                              │
│  • Supply chain security                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Generate SBOM

```bash
# Using Syft
syft . -o spdx-json > sbom.spdx.json
syft . -o cyclonedx-json > sbom.cyclonedx.json

# Using Trivy
trivy fs --format spdx-json -o sbom.spdx.json .
trivy fs --format cyclonedx -o sbom.cyclonedx.json .

# Using CycloneDX tools
# Node.js
npx @cyclonedx/cyclonedx-npm --output-file sbom.json

# Python
pip install cyclonedx-bom
cyclonedx-py -o sbom.json
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SCA BEST PRACTICES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Prevention:                                                        │
│  • Keep dependencies updated                                       │
│  • Use lockfiles (package-lock.json, Pipfile.lock)                │
│  • Pin versions in production                                      │
│  • Review new dependencies before adding                           │
│  • Minimize dependency count                                       │
│                                                                     │
│  Detection:                                                         │
│  • Scan on every commit/PR                                         │
│  • Monitor continuously (Snyk, Dependabot)                         │
│  • Include transitive dependencies                                 │
│  • Generate and store SBOMs                                        │
│                                                                     │
│  Response:                                                          │
│  • Set severity thresholds (fail on critical/high)                 │
│  • Automate patch updates where safe                               │
│  • Have process for manual updates                                 │
│  • Track remediation time                                          │
│                                                                     │
│  Governance:                                                        │
│  • Define approved packages                                        │
│  • License compliance checking                                     │
│  • Regular dependency review                                       │
│  • Document exceptions                                             │
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
│  SCA Tools:                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Snyk          │ Commercial, comprehensive, developer-friendly│   │
│  │ Dependabot    │ Free, GitHub native, auto-PRs              │   │
│  │ OWASP DC      │ Free, thorough, many languages             │   │
│  │ Trivy         │ Free, fast, container + filesystem         │   │
│  │ npm audit     │ Built-in, npm only                         │   │
│  │ safety        │ Python specific                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Actions:                                                       │
│  1. Enable Dependabot/Snyk for continuous monitoring              │
│  2. Add SCA to CI/CD pipeline                                      │
│  3. Set severity thresholds for blocking                           │
│  4. Generate SBOMs for compliance                                  │
│  5. Automate safe updates                                          │
│                                                                     │
│  Next: Learn about container security                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
