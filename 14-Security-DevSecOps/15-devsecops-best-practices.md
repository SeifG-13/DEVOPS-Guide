# DevSecOps Best Practices

## DevSecOps Maturity Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS MATURITY LEVELS                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Level 1 - Initial:                                                 │
│  • Manual security testing                                         │
│  • Security as afterthought                                        │
│  • Reactive approach                                               │
│                                                                     │
│  Level 2 - Managed:                                                 │
│  • Basic automated scanning                                        │
│  • Security gates in pipeline                                      │
│  • Defined security policies                                       │
│                                                                     │
│  Level 3 - Defined:                                                 │
│  • Comprehensive automation                                        │
│  • Security integrated in SDLC                                     │
│  • Metrics and reporting                                           │
│                                                                     │
│  Level 4 - Measured:                                                │
│  • Continuous security monitoring                                  │
│  • Risk-based prioritization                                       │
│  • Security champions program                                      │
│                                                                     │
│  Level 5 - Optimized:                                               │
│  • Security by design                                              │
│  • Automated remediation                                           │
│  • Continuous improvement culture                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Security in the Development Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY AT EVERY STAGE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Plan           │ Threat modeling, security requirements           │
│  ────────────────────────────────────────────────────────────────   │
│  Code           │ Secure coding, IDE plugins, pre-commit hooks     │
│  ────────────────────────────────────────────────────────────────   │
│  Build          │ SAST, dependency scanning, secrets detection     │
│  ────────────────────────────────────────────────────────────────   │
│  Test           │ DAST, penetration testing, security unit tests   │
│  ────────────────────────────────────────────────────────────────   │
│  Release        │ Image signing, SBOM, compliance checks           │
│  ────────────────────────────────────────────────────────────────   │
│  Deploy         │ IaC scanning, admission control, GitOps          │
│  ────────────────────────────────────────────────────────────────   │
│  Operate        │ Runtime protection, monitoring, logging          │
│  ────────────────────────────────────────────────────────────────   │
│  Monitor        │ Threat detection, incident response              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Comprehensive Pipeline Security

### Complete Secure Pipeline

```yaml
# .github/workflows/secure-pipeline.yaml
name: Secure CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write
  packages: write
  id-token: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Stage 1: Secret Scanning
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          extra_args: --only-verified

  # Stage 2: SAST
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Semgrep Scan
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten

      - name: CodeQL Analysis
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, python

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

  # Stage 3: Dependency Scanning
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v3
        with:
          fail-on-severity: high

      - name: Snyk Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  # Stage 4: Build and Scan Image
  build:
    needs: [secret-scan, sast, dependency-scan]
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Image
        uses: docker/build-push-action@v5
        id: build
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy Results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  # Stage 5: Sign and Attest
  sign:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Sign Image
        run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Attest SBOM
        run: |
          cosign attest --yes --predicate sbom.spdx.json \
            --type spdxjson \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}

  # Stage 6: DAST (on staging)
  dast:
    needs: sign
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to Staging
        run: |
          # Deploy to staging environment
          kubectl set image deployment/app \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: OWASP ZAP Scan
        uses: zaproxy/action-full-scan@v0.7.0
        with:
          target: 'https://staging.example.com'
          rules_file_name: '.zap/rules.tsv'

  # Stage 7: Deploy to Production
  deploy:
    needs: [sign, dast]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4

      - name: Verify Image Signature
        run: |
          cosign verify \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}

      - name: Deploy to Production
        run: |
          kubectl set image deployment/app \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}
```

---

## Security Champions Program

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY CHAMPIONS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What is a Security Champion?                                       │
│  • Developer with security focus                                   │
│  • Bridge between dev and security teams                           │
│  • Advocates for security in their team                            │
│  • First responder for security questions                          │
│                                                                     │
│  Responsibilities:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Educate       │ Train team on secure coding                 │   │
│  │ Review        │ Security-focused code reviews               │   │
│  │ Advocate      │ Promote security best practices             │   │
│  │ Triage        │ Initial assessment of vulnerabilities       │   │
│  │ Communicate   │ Liaison with security team                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Benefits:                                                          │
│  • Scales security across organization                             │
│  • Faster security feedback                                        │
│  • Security becomes part of team culture                           │
│  • Reduced bottleneck on security team                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Security Metrics and KPIs

### Key Metrics

```yaml
# security-metrics.yaml
metrics:
  vulnerability_management:
    - name: Mean Time to Remediate (MTTR)
      description: Average time from vulnerability discovery to fix
      target: "< 7 days for critical, < 30 days for high"

    - name: Vulnerability Escape Rate
      description: Vulnerabilities found in production vs pre-production
      target: "< 5%"

    - name: Open Vulnerability Count
      description: Number of unresolved vulnerabilities by severity
      target: "0 critical, < 10 high"

  pipeline_security:
    - name: Security Gate Pass Rate
      description: Percentage of builds passing security checks
      target: "> 95%"

    - name: False Positive Rate
      description: Percentage of findings that are false positives
      target: "< 10%"

    - name: Security Scan Coverage
      description: Percentage of repos with automated scanning
      target: "100%"

  incident_response:
    - name: Mean Time to Detect (MTTD)
      description: Time from incident start to detection
      target: "< 1 hour"

    - name: Mean Time to Respond (MTTR)
      description: Time from detection to containment
      target: "< 4 hours"

    - name: Incidents per Quarter
      description: Number of security incidents
      target: "Trending down"

  compliance:
    - name: Policy Compliance Rate
      description: Resources compliant with security policies
      target: "> 99%"

    - name: Audit Finding Count
      description: Issues found in security audits
      target: "Trending down"
```

### Metrics Dashboard

```yaml
# Grafana dashboard for security metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-metrics-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "title": "DevSecOps Metrics",
      "panels": [
        {
          "title": "Vulnerability MTTR (Days)",
          "type": "stat",
          "targets": [{
            "expr": "avg(vulnerability_remediation_time_days) by (severity)"
          }]
        },
        {
          "title": "Open Vulnerabilities",
          "type": "piechart",
          "targets": [{
            "expr": "sum(open_vulnerabilities) by (severity)"
          }]
        },
        {
          "title": "Security Gate Pass Rate",
          "type": "gauge",
          "targets": [{
            "expr": "sum(security_gate_pass) / sum(security_gate_total) * 100"
          }]
        },
        {
          "title": "Vulnerability Trend",
          "type": "timeseries",
          "targets": [{
            "expr": "sum(open_vulnerabilities) by (severity)"
          }]
        },
        {
          "title": "MTTD/MTTR",
          "type": "stat",
          "targets": [
            {"expr": "avg(incident_detection_time_hours)"},
            {"expr": "avg(incident_response_time_hours)"}
          ]
        }
      ]
    }
```

---

## Tool Selection Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS TOOL STACK                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SAST (Static Analysis):                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Semgrep      │ Fast, customizable, good for most languages  │   │
│  │ CodeQL       │ Powerful queries, GitHub integration         │   │
│  │ SonarQube    │ Comprehensive, quality + security            │   │
│  │ Bandit       │ Python-specific                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Dependency Scanning:                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Snyk         │ Comprehensive, good remediation advice       │   │
│  │ Dependabot   │ GitHub native, automatic PRs                 │   │
│  │ Trivy        │ Fast, multi-purpose, free                    │   │
│  │ OWASP DC     │ Open source, NVD-based                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Container Security:                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Trivy        │ All-in-one, fast, accurate                   │   │
│  │ Grype        │ Fast, Syft integration                       │   │
│  │ Clair        │ CoreOS, good for registries                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  DAST (Dynamic Analysis):                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ OWASP ZAP    │ Comprehensive, free, API support             │   │
│  │ Nuclei       │ Template-based, fast                         │   │
│  │ Burp Suite   │ Professional, manual + automated             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Runtime Security:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Falco        │ Cloud native, eBPF-based                     │   │
│  │ Sysdig       │ Commercial, comprehensive                    │   │
│  │ Aqua         │ Full lifecycle security                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Secret Management:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Vault        │ Feature-rich, dynamic secrets                │   │
│  │ AWS SM       │ AWS native, simple                           │   │
│  │ External Sec │ Kubernetes native, multi-backend             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Culture and Process

### Security Culture Principles

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BUILDING SECURITY CULTURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Everyone's Responsibility:                                         │
│  • Security is not just the security team's job                    │
│  • Developers own the security of their code                       │
│  • Ops owns infrastructure security                                │
│  • Everyone is accountable                                         │
│                                                                     │
│  Blameless Culture:                                                 │
│  • Focus on systems, not individuals                               │
│  • Encourage reporting vulnerabilities                             │
│  • Learn from incidents                                            │
│  • Celebrate security wins                                         │
│                                                                     │
│  Continuous Learning:                                               │
│  • Regular security training                                       │
│  • Share security knowledge                                        │
│  • Learn from industry incidents                                   │
│  • Stay updated on threats                                         │
│                                                                     │
│  Enable, Don't Block:                                               │
│  • Security should enable velocity                                 │
│  • Automate security checks                                        │
│  • Provide clear guidance                                          │
│  • Make secure path the easy path                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Security Training Program

```yaml
# security-training-plan.yaml
training_program:
  onboarding:
    - name: Security Fundamentals
      duration: 2 hours
      topics:
        - OWASP Top 10
        - Secure coding basics
        - Company security policies
        - Tool overview

  ongoing:
    - name: Monthly Security Lunch & Learn
      frequency: Monthly
      format: Presentation + Discussion
      topics:
        - Recent vulnerabilities
        - Tool updates
        - Case studies

    - name: Secure Code Review Training
      frequency: Quarterly
      format: Workshop
      duration: 4 hours
      topics:
        - Code review checklist
        - Common vulnerability patterns
        - Hands-on exercises

    - name: CTF Events
      frequency: Semi-annual
      format: Competition
      duration: 1 day
      purpose: Practical security skills

  specialized:
    - name: Security Champion Training
      duration: 3 days
      topics:
        - Advanced secure coding
        - Threat modeling
        - Security testing
        - Incident response

    - name: Cloud Security
      duration: 1 day
      topics:
        - AWS/Azure/GCP security
        - IAM best practices
        - Cloud-specific threats
```

---

## Implementation Roadmap

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS IMPLEMENTATION ROADMAP                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1 - Foundation (Months 1-3):                                 │
│  □ Implement secret scanning in CI/CD                              │
│  □ Add dependency scanning                                         │
│  □ Enable basic SAST                                               │
│  □ Set up security training                                        │
│                                                                     │
│  Phase 2 - Integration (Months 4-6):                                │
│  □ Implement container scanning                                    │
│  □ Add IaC security scanning                                       │
│  □ Set up security gates                                           │
│  □ Launch security champions program                               │
│                                                                     │
│  Phase 3 - Automation (Months 7-9):                                 │
│  □ Implement DAST in staging                                       │
│  □ Add image signing and verification                              │
│  □ Set up compliance as code                                       │
│  □ Automate vulnerability ticketing                                │
│                                                                     │
│  Phase 4 - Runtime (Months 10-12):                                  │
│  □ Deploy runtime security (Falco)                                 │
│  □ Implement admission control                                     │
│  □ Set up security monitoring                                      │
│  □ Create incident response playbooks                              │
│                                                                     │
│  Phase 5 - Optimization (Ongoing):                                  │
│  □ Tune false positives                                            │
│  □ Track and improve metrics                                       │
│  □ Regular security assessments                                    │
│  □ Continuous improvement                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS BEST PRACTICES                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Code Security:                                                     │
│  ✓ Use pre-commit hooks for secrets                                │
│  ✓ Implement SAST in every PR                                      │
│  ✓ Dependency scanning with auto-updates                           │
│  ✓ Security-focused code reviews                                   │
│                                                                     │
│  Build Security:                                                    │
│  ✓ Pin dependencies to specific versions                           │
│  ✓ Use minimal base images                                         │
│  ✓ Scan images before pushing                                      │
│  ✓ Sign artifacts and images                                       │
│                                                                     │
│  Deploy Security:                                                   │
│  ✓ Use GitOps for deployments                                      │
│  ✓ Verify signatures before deploy                                 │
│  ✓ Admission control policies                                      │
│  ✓ Least privilege permissions                                     │
│                                                                     │
│  Runtime Security:                                                  │
│  ✓ Enable runtime threat detection                                 │
│  ✓ Implement network policies                                      │
│  ✓ Use read-only containers                                        │
│  ✓ Monitor and alert on anomalies                                  │
│                                                                     │
│  Process:                                                           │
│  ✓ Threat model new features                                       │
│  ✓ Regular penetration testing                                     │
│  ✓ Incident response drills                                        │
│  ✓ Measure and improve metrics                                     │
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
│  DevSecOps Pillars:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Automation    │ Security in every pipeline stage            │   │
│  │ Shift Left    │ Find issues early in development            │   │
│  │ Culture       │ Security is everyone's responsibility       │   │
│  │ Measurement   │ Track metrics, improve continuously         │   │
│  │ Defense Depth │ Multiple layers of security                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Success Factors:                                               │
│  • Executive support and investment                                │
│  • Security champions in each team                                 │
│  • Automated security in CI/CD                                     │
│  • Continuous training and learning                                │
│  • Metrics-driven improvement                                      │
│                                                                     │
│  Remember:                                                          │
│  • Start small, iterate quickly                                    │
│  • Focus on highest risk first                                     │
│  • Make security the easy path                                     │
│  • Build security culture, not just tools                          │
│  • Measure, learn, improve                                         │
│                                                                     │
│  This completes the Security-DevSecOps section!                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
