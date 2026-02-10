# SAST - Static Application Security Testing

## What is SAST?

SAST analyzes source code, bytecode, or binaries to find security vulnerabilities without executing the application.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SAST OVERVIEW                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │   Source Code ───▶ SAST Tool ───▶ Vulnerability Report     │   │
│  │                                                             │   │
│  │   • No execution required                                   │   │
│  │   • Full code coverage                                      │   │
│  │   • Early in SDLC (shift-left)                             │   │
│  │   • Language-specific analysis                              │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  SAST Detects:                                                      │
│  • SQL Injection                • Hardcoded credentials            │
│  • XSS vulnerabilities          • Insecure crypto                  │
│  • Path traversal               • Code quality issues              │
│  • Buffer overflows             • Dangerous functions              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## SAST vs DAST

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SAST vs DAST                                      │
├─────────────────┬───────────────────────┬───────────────────────────┤
│ Aspect          │ SAST                  │ DAST                      │
├─────────────────┼───────────────────────┼───────────────────────────┤
│ Testing Type    │ White-box             │ Black-box                 │
│ Requires        │ Source code           │ Running application       │
│ When            │ Build/commit time     │ Test/staging environment  │
│ Coverage        │ All code paths        │ Exposed endpoints only    │
│ False Positives │ Higher                │ Lower                     │
│ Context         │ Code context          │ Runtime context           │
│ Speed           │ Fast                  │ Slower                    │
│ Finds           │ Code vulnerabilities  │ Runtime vulnerabilities   │
└─────────────────┴───────────────────────┴───────────────────────────┘
```

---

## Popular SAST Tools

### Open Source

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OPEN SOURCE SAST TOOLS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Semgrep                                                            │
│  ────────                                                           │
│  • Multi-language (Python, JS, Go, Java, etc.)                     │
│  • Custom rules with simple syntax                                 │
│  • Fast and lightweight                                            │
│  • Great CI/CD integration                                         │
│                                                                     │
│  SonarQube (Community Edition)                                      │
│  ─────────────────────────────                                      │
│  • Code quality + security                                         │
│  • 25+ languages                                                   │
│  • Quality gates                                                   │
│  • CI/CD integration                                               │
│                                                                     │
│  CodeQL                                                             │
│  ───────                                                            │
│  • GitHub's semantic analysis                                      │
│  • Query language for code                                         │
│  • Free for open source                                            │
│  • Deep data flow analysis                                         │
│                                                                     │
│  Bandit (Python)                                                    │
│  ───────────────                                                    │
│  • Python-specific                                                 │
│  • Fast and easy to use                                            │
│  • Configurable plugins                                            │
│                                                                     │
│  ESLint Security Plugins (JavaScript)                               │
│  ────────────────────────────────────                               │
│  • eslint-plugin-security                                          │
│  • eslint-plugin-no-secrets                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Commercial

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMMERCIAL SAST TOOLS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Checkmarx       │ Enterprise SAST, many languages                 │
│  Veracode        │ Cloud-based, comprehensive                      │
│  Fortify         │ Deep analysis, enterprise scale                 │
│  Snyk Code       │ ML-powered, developer-friendly                  │
│  SonarQube (EE)  │ Advanced rules, branch analysis                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Semgrep

### Installation and Usage

```bash
# Install Semgrep
pip install semgrep
# or
brew install semgrep

# Run with auto config (recommended rules)
semgrep --config auto .

# Run specific rulesets
semgrep --config p/security-audit .
semgrep --config p/owasp-top-ten .
semgrep --config p/python .

# Output formats
semgrep --config auto --json -o results.json .
semgrep --config auto --sarif -o results.sarif .
```

### Custom Semgrep Rule

```yaml
# custom-rules.yaml
rules:
  - id: hardcoded-password
    patterns:
      - pattern: password = "..."
      - pattern: PASSWORD = "..."
      - pattern: pwd = "..."
    message: "Hardcoded password detected"
    languages: [python, javascript, java]
    severity: ERROR
    metadata:
      cwe: "CWE-259"
      owasp: "A07:2021"

  - id: sql-injection
    patterns:
      - pattern: |
          cursor.execute("..." + $VAR + "...")
      - pattern: |
          cursor.execute(f"...{$VAR}...")
    message: "Possible SQL injection"
    languages: [python]
    severity: ERROR

  - id: dangerous-eval
    pattern: eval($X)
    message: "Use of eval() is dangerous"
    languages: [python, javascript]
    severity: WARNING
```

```bash
# Run custom rules
semgrep --config custom-rules.yaml .
```

### CI/CD Integration

```yaml
# GitHub Actions
name: Semgrep
on: [push, pull_request]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
          generateSarif: true

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: semgrep.sarif
```

---

## SonarQube

### Docker Setup

```yaml
# docker-compose.yml
version: "3"
services:
  sonarqube:
    image: sonarqube:community
    ports:
      - "9000:9000"
    environment:
      - SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs

volumes:
  sonarqube_data:
  sonarqube_logs:
```

### Running Analysis

```bash
# Install scanner
brew install sonar-scanner
# or download from sonarqube.org

# Create sonar-project.properties
cat > sonar-project.properties << EOF
sonar.projectKey=my-project
sonar.projectName=My Project
sonar.sources=src
sonar.host.url=http://localhost:9000
sonar.token=your-token-here
sonar.language=python
EOF

# Run scan
sonar-scanner
```

### Quality Gate Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SONARQUBE QUALITY GATE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Recommended Security Conditions:                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Metric                      │ Operator │ Value              │   │
│  ├─────────────────────────────┼──────────┼────────────────────┤   │
│  │ Security Hotspots Reviewed  │ >=       │ 100%               │   │
│  │ New Security Rating         │ <=       │ A (no vulns)       │   │
│  │ New Vulnerabilities         │ =        │ 0                  │   │
│  │ Security Rating             │ <=       │ B                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Pipeline fails if Quality Gate fails                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CodeQL

### GitHub Integration

```yaml
# .github/workflows/codeql.yml
name: "CodeQL"

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      security-events: write

    strategy:
      matrix:
        language: ['javascript', 'python']

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: ${{ matrix.language }}
          queries: +security-extended

      - name: Autobuild
        uses: github/codeql-action/autobuild@v2

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
```

### Custom CodeQL Query

```ql
// Find SQL injection in Python
import python
import semmle.python.security.dataflow.SqlInjection

from SqlInjectionConfiguration config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "SQL injection from $@.", source.getNode(), "user input"
```

---

## Bandit (Python)

### Usage

```bash
# Install
pip install bandit

# Basic scan
bandit -r ./src

# With confidence level
bandit -r ./src -ll  # medium and high confidence

# Exclude tests
bandit -r ./src --exclude ./src/tests

# Output formats
bandit -r ./src -f json -o bandit-results.json
bandit -r ./src -f sarif -o bandit-results.sarif

# Specific tests only
bandit -r ./src -t B101,B102,B103
```

### Configuration

```yaml
# .bandit.yaml
exclude_dirs:
  - tests
  - venv

skips:
  - B101  # assert_used
  - B311  # random (not for crypto)

assert_used:
  skips:
    - '*_test.py'
    - 'test_*.py'
```

### CI Integration

```yaml
# GitHub Actions
- name: Run Bandit
  run: |
    pip install bandit
    bandit -r src/ -f sarif -o bandit.sarif || true

- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: bandit.sarif
```

---

## Pipeline Integration

### Complete SAST Pipeline

```yaml
# GitLab CI example
stages:
  - security-scan

sast:
  stage: security-scan
  image: python:3.11
  before_script:
    - pip install semgrep bandit
  script:
    # Run Semgrep
    - semgrep --config auto --sarif -o semgrep.sarif . || true

    # Run Bandit
    - bandit -r src/ -f sarif -o bandit.sarif || true

    # Fail on critical findings
    - |
      CRITICAL=$(semgrep --config auto --json . | jq '[.results[] | select(.extra.severity == "ERROR")] | length')
      if [ "$CRITICAL" -gt 0 ]; then
        echo "Found $CRITICAL critical vulnerabilities"
        exit 1
      fi
  artifacts:
    reports:
      sast:
        - semgrep.sarif
        - bandit.sarif
  allow_failure: false
```

### Security Gate Logic

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SAST GATE STRATEGY                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Severity-Based Gates:                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Critical/High  │ Block pipeline immediately                 │   │
│  │ Medium         │ Require review, can proceed                │   │
│  │ Low            │ Log only, fix in backlog                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Baseline Approach:                                                 │
│  • Only fail on NEW vulnerabilities                                │
│  • Track existing issues separately                                │
│  • Gradually reduce baseline over time                             │
│                                                                     │
│  Delta Scanning:                                                    │
│  • Scan only changed files in PRs                                  │
│  • Full scan on main branch                                        │
│  • Faster feedback loop                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Handling False Positives

```yaml
# Semgrep inline suppression
def get_user(user_id):
    # nosemgrep: sql-injection
    query = f"SELECT * FROM users WHERE id = {user_id}"  # Actually safe
```

```python
# Bandit inline suppression
password = "not-a-real-password"  # nosec B105

# Or using comments
subprocess.call(cmd, shell=True)  # nosec B602 - input validated
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FALSE POSITIVE MANAGEMENT                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Best Practices:                                                    │
│  1. Review all suppressions in code review                         │
│  2. Document why it's a false positive                             │
│  3. Centralize suppression rules where possible                    │
│  4. Periodically review suppressions                               │
│  5. Tune tool rules to reduce noise                                │
│                                                                     │
│  Track Metrics:                                                     │
│  • False positive rate per tool                                    │
│  • Time spent on triage                                            │
│  • Suppressions added over time                                    │
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
│  Recommended Tools:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ General      │ Semgrep (free), SonarQube                    │   │
│  │ Python       │ Bandit, Semgrep                              │   │
│  │ JavaScript   │ ESLint security plugins, Semgrep             │   │
│  │ GitHub       │ CodeQL (free for public repos)               │   │
│  │ Enterprise   │ Checkmarx, Veracode, Snyk Code               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Integration Points:                                                │
│  • IDE (real-time feedback)                                        │
│  • Pre-commit hooks (prevent bad commits)                          │
│  • CI/CD pipeline (gate releases)                                  │
│  • Scheduled scans (catch regressions)                             │
│                                                                     │
│  Best Practices:                                                    │
│  • Start with auto/recommended rules                               │
│  • Tune to reduce false positives                                  │
│  • Block on critical findings                                      │
│  • Track metrics over time                                         │
│                                                                     │
│  Next: Learn about DAST (Dynamic Analysis)                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
