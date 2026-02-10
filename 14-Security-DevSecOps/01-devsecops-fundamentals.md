# DevSecOps Fundamentals

## What is DevSecOps?

DevSecOps integrates security practices into every phase of the software development lifecycle.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS DEFINED                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DevSecOps = Development + Security + Operations                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  "Security is everyone's responsibility, not just the       │   │
│  │   security team's. DevSecOps embeds security into the       │   │
│  │   DNA of software development."                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Traditional:  Dev ──▶ Ops ──▶ Security (end)                      │
│  DevSecOps:    Dev + Sec + Ops (integrated throughout)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The Shift-Left Approach

### What is Shift-Left?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SHIFT-LEFT SECURITY                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Traditional Security (Shift-Right):                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Plan → Code → Build → Test → Deploy → [SECURITY] → Operate │   │
│  │                                            ↑                │   │
│  │                                     Late & Expensive        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Shift-Left Security:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ [SEC] → [SEC] → [SEC] → [SEC] → [SEC] → [SEC] → [SEC]      │   │
│  │  ↓        ↓       ↓       ↓       ↓       ↓       ↓        │   │
│  │ Plan → Code → Build → Test → Deploy → Release → Operate    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Benefits:                                                          │
│  • Find vulnerabilities early (cheaper to fix)                     │
│  • Prevent security debt                                           │
│  • Faster release cycles                                           │
│  • Better security culture                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Cost of Fixing Vulnerabilities

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COST TO FIX BY PHASE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Cost ($)                                                           │
│    ▲                                                                │
│    │                                               ████████        │
│    │                                               ████████        │
│    │                                    ████       ████████        │
│    │                                    ████       ████████        │
│    │                         ████       ████       ████████        │
│    │              ████       ████       ████       ████████        │
│    │    ████      ████       ████       ████       ████████        │
│    │    ████      ████       ████       ████       ████████        │
│    └────────────────────────────────────────────────────────▶      │
│         Design    Code      Test      Release    Production        │
│          $1       $10       $100       $1000      $10000           │
│                                                                     │
│  Finding a bug in production costs 100x more than in design!       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## DevSecOps Lifecycle

### Security in Each Phase

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS LIFECYCLE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                         ┌─────────────┐                            │
│                    ┌────│    PLAN     │────┐                       │
│                    │    │ • Threat    │    │                       │
│                    │    │   Modeling  │    │                       │
│                    │    │ • Security  │    │                       │
│              ┌─────┴──┐ │   Require.  │ ┌──┴─────┐                 │
│              │MONITOR │ └─────────────┘ │  CODE  │                 │
│              │        │                 │        │                 │
│              │• SIEM  │                 │• SAST  │                 │
│              │• Alerts│    DevSecOps    │• Linter│                 │
│              │• Audit │      Cycle      │• IDE   │                 │
│              └─────┬──┘                 └──┬─────┘                 │
│                    │    ┌─────────────┐    │                       │
│                    │    │   DEPLOY    │    │                       │
│                    │    │ • IaC Scan  │    │                       │
│                    │    │ • Config    │    │                       │
│                    └────│   Audit     │────┘                       │
│                         └──────┬──────┘                            │
│                                │                                    │
│                    ┌───────────┴───────────┐                       │
│                    │                       │                       │
│              ┌─────┴─────┐          ┌──────┴────┐                  │
│              │  RELEASE  │          │   BUILD   │                  │
│              │           │          │           │                  │
│              │• Pen Test │          │• SAST     │                  │
│              │• Sign     │          │• SCA      │                  │
│              │• Approve  │          │• Container│                  │
│              └───────────┘          └───────────┘                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Security Activities by Phase

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY BY PHASE                                 │
├─────────────────┬───────────────────────────────────────────────────┤
│ Phase           │ Security Activities                               │
├─────────────────┼───────────────────────────────────────────────────┤
│ PLAN            │ • Threat modeling                                 │
│                 │ • Security requirements                           │
│                 │ • Risk assessment                                 │
│                 │ • Security user stories                           │
├─────────────────┼───────────────────────────────────────────────────┤
│ CODE            │ • Secure coding training                          │
│                 │ • IDE security plugins                            │
│                 │ • Pre-commit hooks                                │
│                 │ • Peer code review                                │
├─────────────────┼───────────────────────────────────────────────────┤
│ BUILD           │ • SAST (Static Analysis)                          │
│                 │ • SCA (Dependency Scanning)                       │
│                 │ • Container image scanning                        │
│                 │ • License compliance                              │
├─────────────────┼───────────────────────────────────────────────────┤
│ TEST            │ • DAST (Dynamic Analysis)                         │
│                 │ • IAST (Interactive Analysis)                     │
│                 │ • Security unit tests                             │
│                 │ • API security testing                            │
├─────────────────┼───────────────────────────────────────────────────┤
│ RELEASE         │ • Penetration testing                             │
│                 │ • Security sign-off                               │
│                 │ • Artifact signing                                │
│                 │ • Compliance validation                           │
├─────────────────┼───────────────────────────────────────────────────┤
│ DEPLOY          │ • IaC security scanning                           │
│                 │ • Configuration audit                             │
│                 │ • Secrets management                              │
│                 │ • Admission control                               │
├─────────────────┼───────────────────────────────────────────────────┤
│ OPERATE         │ • Runtime protection                              │
│                 │ • Log monitoring                                  │
│                 │ • Vulnerability management                        │
│                 │ • Incident response                               │
├─────────────────┼───────────────────────────────────────────────────┤
│ MONITOR         │ • SIEM/SOC                                        │
│                 │ • Anomaly detection                               │
│                 │ • Compliance monitoring                           │
│                 │ • Security metrics                                │
└─────────────────┴───────────────────────────────────────────────────┘
```

---

## DevSecOps Principles

### Core Principles

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS PRINCIPLES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. AUTOMATION FIRST                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Automate security testing in CI/CD                        │   │
│  │ • Policy-as-code                                            │   │
│  │ • Automated remediation where possible                      │   │
│  │ • Reduce manual security gates                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  2. SECURITY AS CODE                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Security policies in version control                      │   │
│  │ • Infrastructure security as code                           │   │
│  │ • Compliance as code                                        │   │
│  │ • Auditable and repeatable                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  3. SHARED RESPONSIBILITY                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Everyone owns security                                    │   │
│  │ • Security champions in teams                               │   │
│  │ • Break down silos                                          │   │
│  │ • Collaborative culture                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  4. CONTINUOUS IMPROVEMENT                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Learn from incidents                                      │   │
│  │ • Regular security assessments                              │   │
│  │ • Metrics-driven decisions                                  │   │
│  │ • Evolve with threats                                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## DevSecOps Culture

### Building a Security Culture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY CULTURE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Traditional                      DevSecOps                         │
│  ───────────                      ─────────                         │
│                                                                     │
│  "Security is the                 "Security is                      │
│   security team's job"             everyone's job"                  │
│                                                                     │
│  Security as gatekeeper  ───▶    Security as enabler               │
│  Security blocks releases ───▶   Security enables safe releases    │
│  Blame culture           ───▶    Learning culture                  │
│  Compliance checkbox     ───▶    Risk-based approach               │
│  Manual reviews          ───▶    Automated checks                  │
│  Reactive                ───▶    Proactive                         │
│                                                                     │
│  Key Cultural Elements:                                             │
│  • Blameless post-mortems                                          │
│  • Security training for all                                       │
│  • Celebrate security wins                                         │
│  • Make security easy and frictionless                             │
│  • Gamification (bug bounties, CTFs)                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Security Champions Program

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY CHAMPIONS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Security Champion = Developer with security expertise              │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                     Security Team                             │ │
│  │                          │                                    │ │
│  │           ┌──────────────┼──────────────┐                    │ │
│  │           ▼              ▼              ▼                    │ │
│  │      ┌────────┐    ┌────────┐    ┌────────┐                 │ │
│  │      │Champion│    │Champion│    │Champion│                 │ │
│  │      │ Team A │    │ Team B │    │ Team C │                 │ │
│  │      └───┬────┘    └───┬────┘    └───┬────┘                 │ │
│  │          │             │             │                       │ │
│  │      ┌───┴───┐     ┌───┴───┐     ┌───┴───┐                  │ │
│  │      │Team A │     │Team B │     │Team C │                  │ │
│  │      │Devs   │     │Devs   │     │Devs   │                  │ │
│  │      └───────┘     └───────┘     └───────┘                  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Champion Responsibilities:                                         │
│  • First point of contact for security questions                   │
│  • Review security-sensitive code                                  │
│  • Advocate for security best practices                            │
│  • Triage security findings                                        │
│  • Bridge between security team and developers                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## DevSecOps Tools Landscape

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS TOOLCHAIN                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PLAN & CODE                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Threat Modeling: OWASP Threat Dragon, Microsoft TMT         │   │
│  │ IDE Security: SonarLint, Snyk IDE, Checkmarx                │   │
│  │ Pre-commit: git-secrets, detect-secrets, pre-commit hooks   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  BUILD & TEST                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ SAST: SonarQube, Semgrep, CodeQL, Checkmarx                 │   │
│  │ SCA: Snyk, Dependabot, OWASP Dependency-Check               │   │
│  │ DAST: OWASP ZAP, Burp Suite, Nuclei                         │   │
│  │ Container: Trivy, Grype, Clair, Anchore                     │   │
│  │ IaC: Checkov, tfsec, KICS, Terrascan                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  DEPLOY & OPERATE                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Secrets: HashiCorp Vault, AWS Secrets Manager               │   │
│  │ K8s Security: OPA/Gatekeeper, Kyverno, Falco               │   │
│  │ Runtime: Falco, Sysdig, Aqua, Prisma Cloud                  │   │
│  │ SIEM: Splunk, ELK, Datadog Security                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## DevSecOps Pipeline

### Security-Integrated Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS PIPELINE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                        CI PIPELINE                            │  │
│  │                                                               │  │
│  │  ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐ │  │
│  │  │Code │──▶│Build│──▶│SAST │──▶│ SCA │──▶│Image│──▶│Unit │ │  │
│  │  │Pull │   │     │   │Scan │   │Scan │   │Scan │   │Tests│ │  │
│  │  └─────┘   └─────┘   └─────┘   └─────┘   └─────┘   └─────┘ │  │
│  │                         │         │         │               │  │
│  │                         ▼         ▼         ▼               │  │
│  │                    ┌─────────────────────────────┐          │  │
│  │                    │    Security Gate            │          │  │
│  │                    │  • Critical vulns: FAIL     │          │  │
│  │                    │  • High vulns: WARN         │          │  │
│  │                    │  • Policy violations: FAIL  │          │  │
│  │                    └─────────────────────────────┘          │  │
│  └───────────────────────────────┬──────────────────────────────┘  │
│                                  │                                  │
│  ┌───────────────────────────────▼──────────────────────────────┐  │
│  │                        CD PIPELINE                            │  │
│  │                                                               │  │
│  │  ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐ │  │
│  │  │ IaC │──▶│DAST │──▶│Sign │──▶│Deploy│──▶│Smoke│──▶│Prod │ │  │
│  │  │Scan │   │Scan │   │Artif│   │Stage │   │Test │   │     │ │  │
│  │  └─────┘   └─────┘   └─────┘   └─────┘   └─────┘   └─────┘ │  │
│  │                                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Example Pipeline Configuration

```yaml
# .gitlab-ci.yml or GitHub Actions workflow
stages:
  - build
  - security-scan
  - test
  - deploy

sast:
  stage: security-scan
  script:
    - semgrep --config auto --json -o sast-results.json .
  artifacts:
    reports:
      sast: sast-results.json
  allow_failure: false

dependency-scan:
  stage: security-scan
  script:
    - snyk test --json > snyk-results.json
  artifacts:
    reports:
      dependency_scanning: snyk-results.json

container-scan:
  stage: security-scan
  script:
    - trivy image --severity CRITICAL,HIGH --exit-code 1 $IMAGE

iac-scan:
  stage: security-scan
  script:
    - checkov -d ./terraform --output json > checkov-results.json

dast:
  stage: test
  script:
    - zap-baseline.py -t $STAGING_URL -r dast-report.html
```

---

## Metrics and KPIs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS METRICS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Vulnerability Metrics:                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Mean Time to Detect (MTTD)                                │   │
│  │ • Mean Time to Remediate (MTTR)                             │   │
│  │ • Vulnerability density (vulns per 1000 LOC)                │   │
│  │ • Open vulnerability count by severity                      │   │
│  │ • Vulnerability aging (days open)                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Process Metrics:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • % of pipelines with security scans                        │   │
│  │ • Security scan coverage                                    │   │
│  │ • False positive rate                                       │   │
│  │ • Pipeline failure rate due to security                     │   │
│  │ • Time to security sign-off                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Culture Metrics:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Security training completion rate                         │   │
│  │ • Security champion coverage                                │   │
│  │ • Security bugs found by developers vs security team        │   │
│  │ • Participation in security activities                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
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
│  DevSecOps Key Concepts:                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Security integrated into every SDLC phase                 │   │
│  │ • Shift-left: Find vulnerabilities early                    │   │
│  │ • Automation: Security testing in CI/CD                     │   │
│  │ • Shared responsibility: Everyone owns security             │   │
│  │ • Continuous improvement: Learn and evolve                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Activities:                                                    │
│  • Threat modeling in planning                                     │
│  • SAST/SCA in build                                               │
│  • DAST in testing                                                 │
│  • Container/IaC scanning                                          │
│  • Runtime protection                                              │
│  • Continuous monitoring                                           │
│                                                                     │
│  Success Factors:                                                   │
│  • Executive support                                               │
│  • Security champions program                                      │
│  • Automated security gates                                        │
│  • Metrics-driven improvement                                      │
│                                                                     │
│  Next: Learn about threat modeling                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
