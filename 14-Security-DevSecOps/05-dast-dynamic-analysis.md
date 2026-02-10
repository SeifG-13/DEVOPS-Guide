# DAST - Dynamic Application Security Testing

## What is DAST?

DAST tests running applications by simulating attacks from the outside (black-box testing).

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DAST OVERVIEW                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │   DAST Tool ───▶ Running App ───▶ Vulnerability Report     │   │
│  │      │               ▲                                      │   │
│  │      │   Requests    │   Responses                         │   │
│  │      └───────────────┘                                      │   │
│  │                                                             │   │
│  │   • Tests like a real attacker                             │   │
│  │   • No source code needed                                  │   │
│  │   • Finds runtime vulnerabilities                          │   │
│  │   • Tests configuration issues                             │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  DAST Detects:                                                      │
│  • SQL Injection (runtime)      • Security misconfigurations       │
│  • XSS (reflected/stored)       • Authentication flaws             │
│  • CSRF vulnerabilities         • Session management issues        │
│  • Server misconfigurations     • Information disclosure           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## DAST Tools

### OWASP ZAP

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OWASP ZAP                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Zed Attack Proxy (ZAP) - Most popular free DAST tool              │
│                                                                     │
│  Features:                                                          │
│  • Active and passive scanning                                     │
│  • Spidering/crawling                                              │
│  • Fuzzing                                                         │
│  • API scanning (OpenAPI/GraphQL)                                  │
│  • Authentication support                                          │
│  • CI/CD integration                                               │
│  • Extensive marketplace                                           │
│                                                                     │
│  Scan Types:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Baseline │ Quick passive scan (~1 min)                      │   │
│  │ Full     │ Complete active scan (longer)                    │   │
│  │ API      │ API-specific scanning                            │   │
│  │ AJAX     │ JavaScript-heavy applications                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# Docker (recommended)
docker pull zaproxy/zap-stable

# Direct download
# https://www.zaproxy.org/download/

# Homebrew (macOS)
brew install --cask zap
```

### ZAP Baseline Scan

```bash
# Quick baseline scan (passive only)
docker run -t zaproxy/zap-stable zap-baseline.py \
  -t https://target-app.example.com \
  -r report.html

# With authentication
docker run -t zaproxy/zap-stable zap-baseline.py \
  -t https://target-app.example.com \
  -r report.html \
  --auth-login-url https://target-app.example.com/login \
  --auth-username admin \
  --auth-password password
```

### ZAP Full Scan

```bash
# Full active scan
docker run -t zaproxy/zap-stable zap-full-scan.py \
  -t https://target-app.example.com \
  -r report.html \
  -m 60  # max scan time in minutes

# With context file
docker run -v $(pwd):/zap/wrk/:rw \
  zaproxy/zap-stable zap-full-scan.py \
  -t https://target-app.example.com \
  -n /zap/wrk/context.context \
  -r report.html
```

### ZAP API Scan

```bash
# Scan OpenAPI spec
docker run -v $(pwd):/zap/wrk/:rw \
  zaproxy/zap-stable zap-api-scan.py \
  -t /zap/wrk/openapi.yaml \
  -f openapi \
  -r api-report.html

# Scan GraphQL
docker run zaproxy/zap-stable zap-api-scan.py \
  -t https://api.example.com/graphql \
  -f graphql \
  -r graphql-report.html
```

---

## ZAP Automation Framework

### Configuration File

```yaml
# zap-config.yaml
env:
  contexts:
    - name: "My App"
      urls:
        - "https://app.example.com"
      includePaths:
        - "https://app.example.com.*"
      excludePaths:
        - ".*logout.*"
      authentication:
        method: "form"
        parameters:
          loginUrl: "https://app.example.com/login"
          loginRequestData: "username={%username%}&password={%password%}"
        verification:
          method: "response"
          loggedInRegex: "\\Qwelcome\\E"
      users:
        - name: "test-user"
          credentials:
            username: "testuser"
            password: "testpass123"

jobs:
  - type: spider
    parameters:
      context: "My App"
      user: "test-user"
      maxDuration: 5

  - type: spiderAjax
    parameters:
      context: "My App"
      maxDuration: 5

  - type: passiveScan-wait
    parameters:
      maxDuration: 10

  - type: activeScan
    parameters:
      context: "My App"
      policy: "API-Scan"

  - type: report
    parameters:
      template: "traditional-html"
      reportFile: "zap-report.html"
      reportTitle: "Security Scan Report"
    risks:
      - high
      - medium
      - low
```

```bash
# Run automation
docker run -v $(pwd):/zap/wrk/:rw \
  zaproxy/zap-stable zap.sh -cmd \
  -autorun /zap/wrk/zap-config.yaml
```

---

## Nuclei

### Fast Vulnerability Scanner

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NUCLEI                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Fast, template-based vulnerability scanner by ProjectDiscovery    │
│                                                                     │
│  Features:                                                          │
│  • Thousands of community templates                                │
│  • Custom template support                                         │
│  • Very fast scanning                                              │
│  • Low false positives                                             │
│  • CI/CD friendly                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```bash
# Install
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
# or
brew install nuclei

# Update templates
nuclei -update-templates

# Basic scan
nuclei -u https://target.example.com

# Specific severity
nuclei -u https://target.example.com -s critical,high

# Specific templates
nuclei -u https://target.example.com -t cves/
nuclei -u https://target.example.com -t exposures/

# Output
nuclei -u https://target.example.com -o results.txt -j
```

### Custom Nuclei Template

```yaml
# custom-template.yaml
id: custom-admin-panel

info:
  name: Admin Panel Detection
  author: your-name
  severity: info
  tags: admin,panel

requests:
  - method: GET
    path:
      - "{{BaseURL}}/admin"
      - "{{BaseURL}}/administrator"
      - "{{BaseURL}}/admin.php"

    matchers-condition: or
    matchers:
      - type: word
        words:
          - "Admin Panel"
          - "Administrator Login"
        condition: or

      - type: status
        status:
          - 200
          - 302
```

---

## CI/CD Integration

### GitHub Actions - ZAP

```yaml
name: DAST Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Nightly

jobs:
  dast:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: |
          # Deploy application to staging environment
          echo "Deploying..."

      - name: ZAP Scan
        uses: zaproxy/action-baseline@v0.9.0
        with:
          target: 'https://staging.example.com'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'

      - name: Upload Report
        uses: actions/upload-artifact@v3
        with:
          name: zap-report
          path: report_html.html
```

### GitLab CI - DAST

```yaml
stages:
  - deploy
  - dast

deploy-staging:
  stage: deploy
  script:
    - kubectl apply -f k8s/staging/
  environment:
    name: staging
    url: https://staging.example.com

dast:
  stage: dast
  image: zaproxy/zap-stable
  script:
    - mkdir -p /zap/wrk
    - zap-baseline.py -t $STAGING_URL -r report.html || true
    - cp /zap/wrk/report.html .
  artifacts:
    paths:
      - report.html
    when: always
  needs:
    - deploy-staging
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any

    stages {
        stage('Deploy Staging') {
            steps {
                sh 'kubectl apply -f k8s/staging/'
                sleep(60)  // Wait for deployment
            }
        }

        stage('DAST Scan') {
            steps {
                script {
                    docker.image('zaproxy/zap-stable').inside {
                        sh '''
                            zap-baseline.py -t https://staging.example.com \
                                -r zap-report.html \
                                -x zap-report.xml
                        '''
                    }
                }
            }
            post {
                always {
                    publishHTML([
                        reportName: 'ZAP Report',
                        reportDir: '.',
                        reportFiles: 'zap-report.html'
                    ])
                }
            }
        }
    }
}
```

---

## Authentication Handling

### Form-Based Authentication

```yaml
# ZAP context for form auth
authentication:
  method: form
  parameters:
    loginUrl: "https://app.example.com/login"
    loginRequestData: "username={%username%}&password={%password%}"
  verification:
    method: response
    loggedInRegex: "\\QLogout\\E"
    loggedOutRegex: "\\QLogin\\E"
```

### Token-Based Authentication

```bash
# ZAP with bearer token
docker run -t zaproxy/zap-stable zap-baseline.py \
  -t https://api.example.com \
  -z "-config replacer.full_list(0).description=auth \
      -config replacer.full_list(0).enabled=true \
      -config replacer.full_list(0).matchtype=REQ_HEADER \
      -config replacer.full_list(0).matchstr=Authorization \
      -config replacer.full_list(0).replacement='Bearer TOKEN_HERE'"
```

---

## DAST Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DAST BEST PRACTICES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Environment:                                                       │
│  • Test against staging, never production                          │
│  • Use dedicated test data                                         │
│  • Isolate test environment                                        │
│  • Reset state between scans                                       │
│                                                                     │
│  Scanning:                                                          │
│  • Start with baseline (passive) scans                             │
│  • Gradually enable active scanning                                │
│  • Configure authentication properly                               │
│  • Exclude logout/destructive endpoints                            │
│  • Set appropriate timeouts                                        │
│                                                                     │
│  Integration:                                                       │
│  • Run after deployment to staging                                 │
│  • Don't block on all findings initially                           │
│  • Focus on critical/high severity                                 │
│  • Correlate with SAST findings                                    │
│                                                                     │
│  Tuning:                                                            │
│  • Whitelist known false positives                                 │
│  • Create custom scan policies                                     │
│  • Adjust scan strength/threshold                                  │
│  • Document exclusions                                             │
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
│  DAST Tools:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ OWASP ZAP  │ Full-featured, free, great for CI/CD           │   │
│  │ Nuclei     │ Fast, template-based, low false positives      │   │
│  │ Burp Suite │ Commercial, comprehensive, manual testing      │   │
│  │ Nikto      │ Web server scanner                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Integration Strategy:                                              │
│  1. Deploy to staging environment                                  │
│  2. Run baseline scan (fast, passive)                              │
│  3. Run full scan on schedule (nightly/weekly)                     │
│  4. Block on critical findings only                                │
│  5. Track and triage findings                                      │
│                                                                     │
│  Key Commands:                                                      │
│  • zap-baseline.py - Quick passive scan                            │
│  • zap-full-scan.py - Complete active scan                         │
│  • zap-api-scan.py - API-specific scanning                         │
│  • nuclei -u <url> - Template-based scanning                       │
│                                                                     │
│  Next: Learn about dependency scanning                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
