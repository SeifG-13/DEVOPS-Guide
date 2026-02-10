# DevSecOps Cheat Sheet for DevOps Engineers

## Quick Reference - Security in CI/CD

### DevSecOps Pipeline Stages
```
Code → SAST → Build → SCA → Container Scan → DAST → Deploy → Runtime Security
  │      │       │      │          │           │        │           │
 Lint  Static  Compile Dependency Image      Dynamic  Prod     Monitoring
      Analysis        Scanning   Scanning   Testing          & Response
```

---

## Static Application Security Testing (SAST)

### SonarQube
```bash
# Run SonarQube analysis
sonar-scanner \
  -Dsonar.projectKey=myproject \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=token
```

### Semgrep
```bash
# Install
pip install semgrep

# Run scan
semgrep --config auto .
semgrep --config p/security-audit .
semgrep --config p/owasp-top-ten .

# CI integration
semgrep ci --config auto
```

### GitHub Actions SAST
```yaml
- name: Run Semgrep
  uses: returntocorp/semgrep-action@v1
  with:
    config: p/security-audit

- name: SonarCloud Scan
  uses: SonarSource/sonarcloud-github-action@master
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## Software Composition Analysis (SCA)

### Dependency Scanning

#### Snyk
```bash
# Install
npm install -g snyk

# Authenticate
snyk auth

# Test for vulnerabilities
snyk test
snyk test --all-projects

# Monitor (add to dashboard)
snyk monitor

# Fix vulnerabilities
snyk fix
```

#### OWASP Dependency-Check
```bash
dependency-check --project MyProject --scan ./src --format HTML
```

#### Trivy (Dependencies)
```bash
# Scan filesystem for vulnerabilities
trivy fs --security-checks vuln,config .
```

### GitHub Actions SCA
```yaml
- name: Snyk Security Scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

---

## Container Security

### Image Scanning

#### Trivy
```bash
# Scan image
trivy image myapp:latest
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan and fail on vulnerabilities
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Scan Dockerfile
trivy config ./Dockerfile
```

#### Grype
```bash
# Install
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s

# Scan image
grype myapp:latest
grype myapp:latest --fail-on high
```

### Docker Security Best Practices
```dockerfile
# Use specific version, not latest
FROM node:18-alpine3.18

# Run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Don't store secrets in image
# Use runtime secrets injection

# Minimize attack surface
RUN apk add --no-cache <only-needed-packages>

# Use multi-stage builds
FROM node:18 AS builder
# ... build steps
FROM node:18-alpine
COPY --from=builder /app/dist ./
```

### GitHub Actions Container Scan
```yaml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

---

## Kubernetes Security

### Pod Security Standards
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:latest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        limits:
          memory: "128Mi"
          cpu: "500m"
```

### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress
      ports:
        - protocol: TCP
          port: 8080
```

### RBAC
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: myapp
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: myapp
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: myapp
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```

### Kube-bench (CIS Benchmarks)
```bash
# Run CIS benchmark checks
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
```

---

## Secret Management

### HashiCorp Vault
```bash
# Start Vault
vault server -dev

# Store secret
vault kv put secret/myapp/db password=secret123

# Get secret
vault kv get secret/myapp/db

# Kubernetes auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

### Sealed Secrets
```bash
# Install
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Create sealed secret
kubectl create secret generic my-secret --from-literal=password=secret -o yaml \
  | kubeseal --format yaml > sealed-secret.yaml

# Apply (safe to commit to Git)
kubectl apply -f sealed-secret.yaml
```

---

## Interview Q&A

### Q1: What is DevSecOps?
**A:** Integrating security practices into DevOps:
- Shift left (security earlier in pipeline)
- Automate security testing
- Continuous security monitoring
- Security as code

### Q2: What is the difference between SAST and DAST?
**A:**
- **SAST (Static)**: Analyzes source code without running it. Finds coding errors, SQL injection patterns
- **DAST (Dynamic)**: Tests running application. Finds runtime vulnerabilities, configuration issues

### Q3: What is SCA (Software Composition Analysis)?
**A:** Scanning third-party dependencies for:
- Known vulnerabilities (CVEs)
- License compliance
- Outdated packages
Tools: Snyk, OWASP Dependency-Check, npm audit

### Q4: How do you secure container images?
**A:**
1. Use minimal base images (Alpine, distroless)
2. Scan for vulnerabilities (Trivy, Grype)
3. Run as non-root user
4. Don't store secrets in images
5. Use multi-stage builds
6. Sign images (Cosign, Notary)

### Q5: What are the OWASP Top 10?
**A:** Most critical web application security risks:
1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable Components
7. Authentication Failures
8. Software/Data Integrity Failures
9. Logging/Monitoring Failures
10. SSRF

### Q6: How do you implement secret management?
**A:**
- **Never in code/Git**: Use secret managers
- **HashiCorp Vault**: Centralized secrets
- **Cloud providers**: AWS Secrets Manager, Azure Key Vault
- **Kubernetes**: External Secrets, Sealed Secrets
- **Rotate regularly**: Automated rotation

### Q7: What is a security scan in CI/CD?
**A:**
```yaml
stages:
  - sast       # Code vulnerabilities
  - sca        # Dependency vulnerabilities
  - build      # Create artifact
  - container  # Image vulnerabilities
  - dast       # Runtime vulnerabilities
  - deploy     # Production deployment
```

### Q8: How do you secure Kubernetes?
**A:**
- Enable RBAC
- Use Network Policies
- Pod Security Standards
- Scan images before deployment
- Use service mesh (mTLS)
- Audit logging
- Regular updates

### Q9: What is supply chain security?
**A:** Securing the entire software supply chain:
- Verify dependencies (SCA)
- Sign artifacts (Sigstore, Cosign)
- SBOM (Software Bill of Materials)
- Secure build pipelines
- Verify image provenance

### Q10: What is the principle of least privilege?
**A:** Grant minimum permissions needed:
- Users get only required access
- Services use dedicated service accounts
- Containers run as non-root
- RBAC with minimal permissions

---

## CI/CD Security Pipeline

```yaml
name: Secure CI/CD

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      # SAST
      - uses: actions/checkout@v4
      - name: SAST - Semgrep
        uses: returntocorp/semgrep-action@v1

      # SCA
      - name: SCA - Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      # Secret detection
      - name: Detect secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./

      # Build
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      # Container scan
      - name: Container scan - Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

---

## Best Practices

1. **Shift left** - Security testing early in pipeline
2. **Automate** - Integrate security scans in CI/CD
3. **Fail fast** - Break builds on critical vulnerabilities
4. **Least privilege** - Minimal permissions everywhere
5. **Secrets management** - Never hardcode secrets
6. **Keep updated** - Patch dependencies and base images
7. **Monitor runtime** - Detect anomalies in production
8. **Security as code** - Version control policies
