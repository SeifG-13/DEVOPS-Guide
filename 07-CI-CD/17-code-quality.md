# Code Quality in CI/CD

## Code Quality Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Code Quality Pipeline                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Source Code                                                   │
│       │                                                         │
│       ├──────────────────────────────────────────────────────┐  │
│       │                                                      │  │
│       ▼                                                      │  │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │  │
│   │   Linting   │───▶│  Formatting │───▶│Static       │    │  │
│   │             │    │             │    │Analysis     │    │  │
│   └─────────────┘    └─────────────┘    └─────────────┘    │  │
│       │                                        │            │  │
│       ▼                                        ▼            │  │
│   ┌─────────────┐                      ┌─────────────┐     │  │
│   │ Code Review │                      │  Security   │     │  │
│   │   (PR)      │                      │  Scanning   │     │  │
│   └─────────────┘                      └─────────────┘     │  │
│       │                                        │            │  │
│       └────────────────┬───────────────────────┘            │  │
│                        ▼                                     │  │
│                ┌─────────────┐                              │  │
│                │Quality Gate │                              │  │
│                │  Pass/Fail  │                              │  │
│                └─────────────┘                              │  │
│                                                              │  │
└─────────────────────────────────────────────────────────────────┘
```

## Linting

### ESLint (JavaScript/TypeScript)

```javascript
// .eslintrc.js
module.exports = {
  env: {
    browser: true,
    es2021: true,
    node: true,
  },
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'prettier',
  ],
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module',
  },
  rules: {
    'no-console': 'warn',
    'no-unused-vars': 'error',
    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/no-explicit-any': 'error',
  },
  ignorePatterns: ['dist/', 'node_modules/'],
};
```

### ESLint in CI

```yaml
# GitHub Actions
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      # Or with reviewdog for PR comments
      - uses: reviewdog/action-eslint@v1
        with:
          reporter: github-pr-review
          eslint_flags: 'src/'
```

### Pylint (Python)

```ini
# .pylintrc
[MAIN]
fail-under=8.0
ignore=tests,migrations

[MESSAGES CONTROL]
disable=C0114,C0115,C0116

[FORMAT]
max-line-length=88

[DESIGN]
max-args=5
max-locals=15
```

```yaml
# CI
- run: pip install pylint
- run: pylint src/ --fail-under=8.0
```

### Checkstyle (Java)

```xml
<!-- checkstyle.xml -->
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
  "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
  "https://checkstyle.org/dtds/configuration_1_3.dtd">
<module name="Checker">
  <module name="TreeWalker">
    <module name="AvoidStarImport"/>
    <module name="UnusedImports"/>
    <module name="LineLength">
      <property name="max" value="120"/>
    </module>
    <module name="MethodLength">
      <property name="max" value="50"/>
    </module>
  </module>
</module>
```

```xml
<!-- Maven pom.xml -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <version>3.3.0</version>
  <configuration>
    <configLocation>checkstyle.xml</configLocation>
    <failOnViolation>true</failOnViolation>
  </configuration>
</plugin>
```

## Code Formatting

### Prettier (JavaScript/TypeScript)

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 80
}
```

```yaml
# CI - Check formatting
jobs:
  format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx prettier --check "src/**/*.{js,ts,tsx}"
```

### Black (Python)

```toml
# pyproject.toml
[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | migrations
)/
'''
```

```yaml
# CI
- run: pip install black
- run: black --check src/
```

### Google Java Format

```yaml
# CI
- uses: axel-op/googlejavaformat-action@v3
  with:
    args: "--replace"
    skip-commit: true
```

## Static Analysis

### SonarQube

```yaml
# GitHub Actions
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

```properties
# sonar-project.properties
sonar.projectKey=my-project
sonar.organization=my-org
sonar.sources=src
sonar.tests=tests
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.coverage.exclusions=**/*.test.js,**/*.spec.js

# Quality Gate conditions
sonar.qualitygate.wait=true
```

### SonarCloud

```yaml
# GitHub Actions
jobs:
  sonarcloud:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - run: npm ci
      - run: npm test -- --coverage
      - uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### CodeClimate

```yaml
# GitHub Actions
jobs:
  codeclimate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: paambaati/codeclimate-action@v5.0.0
        env:
          CC_TEST_REPORTER_ID: ${{ secrets.CC_TEST_REPORTER_ID }}
        with:
          coverageCommand: npm test -- --coverage
          coverageLocations: ${{github.workspace}}/coverage/lcov.info:lcov
```

## Security Analysis

### Snyk

```yaml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

### Trivy

```yaml
jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

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

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      production-dependencies:
        patterns:
          - "*"
        exclude-patterns:
          - "@types/*"
          - "eslint*"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Quality Gates

### Defining Quality Gates

```yaml
# GitHub Actions - Quality gate checks
jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci

      # Lint check
      - name: Lint
        run: npm run lint

      # Type check
      - name: Type Check
        run: npm run type-check

      # Test with coverage
      - name: Test
        run: npm test -- --coverage --coverageReporters=json-summary

      # Coverage threshold
      - name: Check Coverage
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80%"
            exit 1
          fi

      # Security scan
      - name: Security
        run: npm audit --audit-level=high
```

### Branch Protection Rules

```
Repository Settings → Branches → Add rule

Branch name pattern: main

Protect matching branches:
✓ Require a pull request before merging
  ✓ Require approvals (1)
  ✓ Dismiss stale pull request approvals
✓ Require status checks to pass before merging
  ✓ Require branches to be up to date
  Status checks:
    - lint
    - test
    - security
✓ Require conversation resolution before merging
```

## Pre-commit Hooks

### Husky + lint-staged

```json
// package.json
{
  "scripts": {
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.{js,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx lint-staged
```

### Pre-commit (Python)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml

  - repo: https://github.com/psf/black
    rev: 23.9.1
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 6.1.0
    hooks:
      - id: flake8

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.5.1
    hooks:
      - id: mypy
```

```bash
# Install
pip install pre-commit
pre-commit install

# Run on all files
pre-commit run --all-files
```

## Code Review Automation

### Reviewdog

```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # ESLint with PR comments
      - uses: reviewdog/action-eslint@v1
        with:
          reporter: github-pr-review
          eslint_flags: 'src/'

      # StyleLint
      - uses: reviewdog/action-stylelint@v1
        with:
          reporter: github-pr-review
          stylelint_input: 'src/**/*.css'
```

### Danger.js

```javascript
// dangerfile.js
import { danger, warn, fail, message } from 'danger';

// Check PR size
const bigPRThreshold = 500;
if (danger.github.pr.additions + danger.github.pr.deletions > bigPRThreshold) {
  warn('This PR is quite large. Consider breaking it into smaller PRs.');
}

// Check for tests
const hasTests = danger.git.modified_files.some(f => f.includes('.test.'));
const hasSourceChanges = danger.git.modified_files.some(f =>
  f.includes('src/') && !f.includes('.test.')
);
if (hasSourceChanges && !hasTests) {
  warn('This PR modifies source files but has no test changes.');
}

// Check for changelog
const hasChangelog = danger.git.modified_files.includes('CHANGELOG.md');
if (!hasChangelog) {
  warn('Please update CHANGELOG.md');
}
```

```yaml
# CI
- uses: danger/danger-js@v11
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Complete Quality Pipeline

```yaml
name: Code Quality

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx prettier --check "src/**/*.{js,ts,tsx}"

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v3

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: github/codeql-action/init@v2
        with:
          languages: javascript
      - uses: github/codeql-action/autobuild@v2
      - uses: github/codeql-action/analyze@v2

  sonar:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

## Quick Reference

### Linting Tools

| Language | Tool | Config File |
|----------|------|-------------|
| JavaScript | ESLint | `.eslintrc` |
| TypeScript | ESLint + TS plugin | `.eslintrc` |
| Python | Pylint, Flake8 | `.pylintrc`, `.flake8` |
| Java | Checkstyle, PMD | `checkstyle.xml` |
| Go | golangci-lint | `.golangci.yml` |

### Formatting Tools

| Language | Tool | Command |
|----------|------|---------|
| JavaScript | Prettier | `prettier --write` |
| Python | Black | `black src/` |
| Java | Google Java Format | `google-java-format` |
| Go | gofmt | `gofmt -w` |

### Quality Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| Code Coverage | % of code tested | >80% |
| Cyclomatic Complexity | Code complexity | <10 |
| Duplication | Repeated code | <3% |
| Technical Debt | Maintainability | A rating |

---

**Previous:** [16-testing-in-ci.md](16-testing-in-ci.md) | **Next:** [18-artifact-management.md](18-artifact-management.md)
