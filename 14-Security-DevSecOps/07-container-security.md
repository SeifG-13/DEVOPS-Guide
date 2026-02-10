# Container Security

## Container Security Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CONTAINER SECURITY LAYERS                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. Base Image Security                                      │   │
│  │    • Use minimal, trusted images                            │   │
│  │    • Scan for vulnerabilities                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 2. Build Security                                           │   │
│  │    • Secure Dockerfile practices                            │   │
│  │    • Multi-stage builds                                     │   │
│  │    • No secrets in images                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 3. Registry Security                                        │   │
│  │    • Private registries                                     │   │
│  │    • Image signing                                          │   │
│  │    • Access controls                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 4. Runtime Security                                         │   │
│  │    • Non-root users                                         │   │
│  │    • Read-only filesystems                                  │   │
│  │    • Security contexts                                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Image Scanning Tools

### Trivy

```bash
# Install
brew install trivy
# or
apt-get install trivy

# Scan image
trivy image nginx:latest

# Scan with severity filter
trivy image --severity CRITICAL,HIGH nginx:latest

# Ignore unfixed vulnerabilities
trivy image --ignore-unfixed nginx:latest

# Output formats
trivy image --format json -o results.json nginx:latest
trivy image --format sarif -o results.sarif nginx:latest

# Scan local image
trivy image --input myimage.tar

# Exit code on findings
trivy image --exit-code 1 --severity CRITICAL nginx:latest
```

### Grype

```bash
# Install
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# Scan image
grype nginx:latest

# Scan with severity threshold
grype nginx:latest --fail-on high

# Output formats
grype nginx:latest -o json > results.json
grype nginx:latest -o sarif > results.sarif

# Scan SBOM
grype sbom:./sbom.json
```

### CI/CD Integration

```yaml
# GitHub Actions - Trivy
name: Container Security

on:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## Dockerfile Best Practices

### Secure Dockerfile

```dockerfile
# Use specific version, not latest
FROM python:3.11-slim-bookworm

# Add labels for tracking
LABEL maintainer="security@example.com"
LABEL version="1.0"

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Set working directory
WORKDIR /app

# Copy only requirements first (caching)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY --chown=appuser:appgroup . .

# Remove unnecessary packages
RUN apt-get purge -y --auto-remove && \
    rm -rf /var/lib/apt/lists/*

# Switch to non-root user
USER appuser

# Set read-only root filesystem
# (handled at runtime in K8s)

# Don't expose unnecessary ports
EXPOSE 8080

# Use exec form for proper signal handling
CMD ["python", "app.py"]
```

### Multi-Stage Build

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# Runtime stage - minimal image
FROM scratch

# Import CA certs for HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary only
COPY --from=builder /build/app /app

# Non-root user (numeric for scratch)
USER 65534

ENTRYPOINT ["/app"]
```

### Dockerfile Linting (Hadolint)

```bash
# Install
brew install hadolint
# or
docker run --rm -i hadolint/hadolint < Dockerfile

# Run lint
hadolint Dockerfile

# Ignore specific rules
hadolint --ignore DL3008 --ignore DL3009 Dockerfile

# CI/CD integration
# GitHub Actions
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
    failure-threshold: error
```

---

## Base Image Security

### Choosing Base Images

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BASE IMAGE SELECTION                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Image Type          │ Size    │ Packages │ Security               │
│  ────────────────────┼─────────┼──────────┼────────────────────────│
│  scratch             │ 0 MB    │ None     │ Best (minimal surface) │
│  distroless          │ ~2 MB   │ Minimal  │ Excellent              │
│  alpine              │ ~5 MB   │ Basic    │ Very Good              │
│  slim (debian/ubuntu)│ ~80 MB  │ Reduced  │ Good                   │
│  full (debian/ubuntu)│ ~200 MB │ Full     │ More vulnerabilities   │
│                                                                     │
│  Recommendations:                                                   │
│  • Go/Rust: Use scratch or distroless                             │
│  • Python/Node: Use slim variants                                  │
│  • Need shell debugging: Use alpine                                │
│  • Avoid: latest tag, full images                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Distroless Images

```dockerfile
# Python distroless
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt -t /app/deps

FROM gcr.io/distroless/python3
WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY . .
ENV PYTHONPATH=/app/deps
CMD ["app.py"]
```

```dockerfile
# Java distroless
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

FROM gcr.io/distroless/java17-debian11
COPY --from=builder /app/target/app.jar /app.jar
CMD ["app.jar"]
```

---

## Image Signing

### Cosign (Sigstore)

```bash
# Install cosign
brew install cosign

# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key myregistry/myimage:tag

# Verify image
cosign verify --key cosign.pub myregistry/myimage:tag

# Keyless signing (uses OIDC)
COSIGN_EXPERIMENTAL=1 cosign sign myregistry/myimage:tag

# Verify keyless
COSIGN_EXPERIMENTAL=1 cosign verify myregistry/myimage:tag
```

### CI/CD Signing

```yaml
# GitHub Actions - Sign with Cosign
- name: Sign image with Cosign
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
  run: |
    cosign sign --key env://COSIGN_PRIVATE_KEY \
      ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

---

## Runtime Security

### Kubernetes Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
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
      volumeMounts:
        - name: tmp
          mountPath: /tmp

  volumes:
    - name: tmp
      emptyDir: {}
```

### Pod Security Standards

```yaml
# Namespace with restricted policy
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

---

## Container Security Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CONTAINER SECURITY CHECKLIST                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Build Time:                                                        │
│  □ Use minimal base images (distroless, alpine, slim)              │
│  □ Pin specific image versions (no :latest)                        │
│  □ Multi-stage builds to reduce size                               │
│  □ No secrets in Dockerfile or image                               │
│  □ Lint Dockerfile with Hadolint                                   │
│  □ Scan image for vulnerabilities                                  │
│  □ Sign images with Cosign                                         │
│                                                                     │
│  Runtime:                                                           │
│  □ Run as non-root user                                            │
│  □ Read-only root filesystem                                       │
│  □ Drop all capabilities                                           │
│  □ No privilege escalation                                         │
│  □ Resource limits set                                             │
│  □ Seccomp profile enabled                                         │
│                                                                     │
│  Registry:                                                          │
│  □ Use private registry                                            │
│  □ Enable vulnerability scanning                                   │
│  □ Implement access controls                                       │
│  □ Enforce image signing                                           │
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
│  Scanning Tools:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Trivy      │ Fast, comprehensive, multiple formats          │   │
│  │ Grype      │ Anchore's scanner, SBOM support               │   │
│  │ Clair      │ Open source, API-based                        │   │
│  │ Snyk       │ Commercial, developer-friendly                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Practices:                                                     │
│  • Use minimal base images                                         │
│  • Scan in CI/CD pipeline                                          │
│  • Run as non-root                                                 │
│  • Sign and verify images                                          │
│  • Apply Pod Security Standards                                    │
│                                                                     │
│  Next: Learn about Kubernetes security                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
