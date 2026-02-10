# Infrastructure Security

## Infrastructure Security Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE SECURITY                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What is Infrastructure Security?                                   │
│  • Securing cloud resources and configurations                     │
│  • Infrastructure as Code (IaC) security                           │
│  • Cloud Security Posture Management (CSPM)                        │
│  • Network security and segmentation                               │
│                                                                     │
│  Security Layers:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Network      │ Firewalls, VPCs, Security Groups             │   │
│  │ Compute      │ Hardening, patching, access control          │   │
│  │ Storage      │ Encryption, access policies                  │   │
│  │ Identity     │ IAM, RBAC, federation                        │   │
│  │ Data         │ Encryption, classification, DLP              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## IaC Security Scanning

### Checkov

```bash
# Install Checkov
pip install checkov

# Scan Terraform
checkov -d ./terraform/

# Scan specific file
checkov -f main.tf

# Scan with specific checks
checkov -d . --check CKV_AWS_1,CKV_AWS_2

# Output formats
checkov -d . -o json > results.json
checkov -d . -o sarif > results.sarif

# Skip checks
checkov -d . --skip-check CKV_AWS_21

# Scan Kubernetes manifests
checkov -d ./k8s/

# Scan Helm charts
checkov -d ./helm/mychart/
```

### Checkov Configuration

```yaml
# .checkov.yaml
compact: true
directory:
  - ./terraform
framework:
  - terraform
  - kubernetes
skip-check:
  - CKV_AWS_21  # S3 versioning
soft-fail: false
output:
  - cli
  - sarif
```

### tfsec

```bash
# Install tfsec
brew install tfsec

# Basic scan
tfsec .

# Output formats
tfsec . --format json > results.json
tfsec . --format sarif > results.sarif

# Exclude specific checks
tfsec . --exclude aws-s3-enable-bucket-logging

# Severity threshold
tfsec . --minimum-severity HIGH

# Soft fail (exit 0)
tfsec . --soft-fail
```

### tfsec Configuration

```yaml
# .tfsec.yml
severity_overrides:
  AWS001: LOW

exclude:
  - aws-s3-enable-bucket-logging

include:
  - "**/*.tf"
```

### Terrascan

```bash
# Install Terrascan
brew install terrascan

# Scan Terraform
terrascan scan -t aws

# Scan Kubernetes
terrascan scan -i k8s -f deployment.yaml

# Scan Helm
terrascan scan -i helm -d ./mychart/

# Output formats
terrascan scan -o json
terrascan scan -o sarif

# Use specific policy
terrascan scan -p /path/to/policies
```

---

## Cloud Security Posture Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CSPM OVERVIEW                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What CSPM Does:                                                    │
│  • Continuous monitoring of cloud configurations                   │
│  • Compliance checking against frameworks                          │
│  • Risk assessment and prioritization                              │
│  • Automated remediation                                           │
│                                                                     │
│  Common Frameworks:                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ CIS Benchmarks   │ Industry-standard security configs       │   │
│  │ SOC 2            │ Service organization controls            │   │
│  │ PCI DSS          │ Payment card industry standards          │   │
│  │ HIPAA            │ Healthcare data protection               │   │
│  │ GDPR             │ European data protection                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### AWS Security Hub

```bash
# Enable Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Get findings
aws securityhub get-findings \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}'

# Enable specific standard
aws securityhub batch-enable-standards \
  --standards-subscription-requests \
  StandardsArn=arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0
```

### Prowler (AWS/Azure/GCP)

```bash
# Install Prowler
pip install prowler

# Scan AWS
prowler aws

# Scan specific checks
prowler aws --checks s3_bucket_public_access

# Compliance check
prowler aws --compliance cis_3.0_aws

# Output formats
prowler aws -M json-asff -o ./output/

# Scan Azure
prowler azure

# Scan GCP
prowler gcp
```

---

## Network Security

### AWS VPC Security

```hcl
# Terraform - Secure VPC Configuration
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "secure-vpc"
  }
}

# Enable VPC Flow Logs
resource "aws_flow_log" "main" {
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
}

# Security Group - Restrictive
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.main.id

  # HTTPS only
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # No SSH from internet
  # Use SSM Session Manager instead

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Network ACL
resource "aws_network_acl" "main" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = [aws_subnet.public.id]

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }
}
```

### Kubernetes Network Policies

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## IAM Security

### AWS IAM Best Practices

```hcl
# Terraform - Least Privilege IAM Role
resource "aws_iam_role" "app" {
  name = "app-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
      Condition = {
        StringEquals = {
          "aws:SourceAccount" = data.aws_caller_identity.current.account_id
        }
      }
    }]
  })
}

# Scoped policy
resource "aws_iam_role_policy" "app" {
  name = "app-policy"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = [
          "arn:aws:s3:::mybucket/app/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          "arn:aws:secretsmanager:*:*:secret:app/*"
        ]
      }
    ]
  })
}

# Enforce MFA for sensitive operations
resource "aws_iam_policy" "mfa_required" {
  name = "mfa-required"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Deny"
        Action = "*"
        Resource = "*"
        Condition = {
          BoolIfExists = {
            "aws:MultiFactorAuthPresent" = "false"
          }
        }
      }
    ]
  })
}
```

### Service Control Policies (SCPs)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootAccount",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    },
    {
      "Sid": "RequireIMDSv2",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:MetadataHttpTokens": "required"
        }
      }
    },
    {
      "Sid": "DenyPublicS3",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:DeletePublicAccessBlock"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Storage Security

### S3 Security Configuration

```hcl
# Terraform - Secure S3 Bucket
resource "aws_s3_bucket" "secure" {
  bucket = "my-secure-bucket"
}

# Block public access
resource "aws_s3_bucket_public_access_block" "secure" {
  bucket = aws_s3_bucket.secure.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Enable versioning
resource "aws_s3_bucket_versioning" "secure" {
  bucket = aws_s3_bucket.secure.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "secure" {
  bucket = aws_s3_bucket.secure.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.s3.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

# Enable logging
resource "aws_s3_bucket_logging" "secure" {
  bucket = aws_s3_bucket.secure.id

  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "s3-access-logs/"
}

# Bucket policy - enforce TLS
resource "aws_s3_bucket_policy" "secure" {
  bucket = aws_s3_bucket.secure.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "EnforceTLS"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.secure.arn,
          "${aws_s3_bucket.secure.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}
```

---

## CI/CD Integration

### GitHub Actions - Infrastructure Security

```yaml
name: Infrastructure Security Scan

on:
  pull_request:
    paths:
      - 'terraform/**'
      - 'kubernetes/**'

jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: terraform/
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: checkov.sarif

  tfsec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: terraform/
          format: sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: results.sarif.json

  terrascan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Terrascan
        uses: tenable/terrascan-action@main
        with:
          iac_type: 'terraform'
          iac_version: 'v14'
          policy_type: 'aws'
          sarif_upload: true
```

### GitLab CI - Infrastructure Security

```yaml
# .gitlab-ci.yml
stages:
  - security

checkov:
  stage: security
  image: bridgecrew/checkov:latest
  script:
    - checkov -d terraform/ -o junitxml > checkov.xml
  artifacts:
    reports:
      junit: checkov.xml
  rules:
    - changes:
        - terraform/**/*

tfsec:
  stage: security
  image: aquasec/tfsec:latest
  script:
    - tfsec terraform/ --format junit > tfsec.xml
  artifacts:
    reports:
      junit: tfsec.xml
  rules:
    - changes:
        - terraform/**/*
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                INFRASTRUCTURE SECURITY BEST PRACTICES               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  IaC Security:                                                      │
│  ✓ Scan all IaC before deployment                                  │
│  ✓ Use policy-as-code (OPA, Sentinel)                              │
│  ✓ Version control all infrastructure                              │
│  ✓ Review changes through PRs                                      │
│                                                                     │
│  Network:                                                           │
│  ✓ Implement network segmentation                                  │
│  ✓ Use private subnets for workloads                               │
│  ✓ Enable flow logs and monitoring                                 │
│  ✓ Default deny, explicit allow                                    │
│                                                                     │
│  Identity:                                                          │
│  ✓ Implement least privilege                                       │
│  ✓ Use temporary credentials                                       │
│  ✓ Enable MFA everywhere                                           │
│  ✓ Regular access reviews                                          │
│                                                                     │
│  Data:                                                              │
│  ✓ Encrypt at rest and in transit                                  │
│  ✓ Use customer-managed keys                                       │
│  ✓ Enable logging and auditing                                     │
│  ✓ Implement data classification                                   │
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
│  IaC Security Tools:                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Checkov      │ Multi-framework, extensive checks            │   │
│  │ tfsec        │ Terraform-focused, fast                      │   │
│  │ Terrascan    │ Policy-driven, multiple IaC types           │   │
│  │ KICS         │ Keeping IaC Secure, multi-platform          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  CSPM Tools:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Prowler      │ Open source, AWS/Azure/GCP                   │   │
│  │ Security Hub │ AWS native, aggregates findings              │   │
│  │ Defender     │ Azure native CSPM                            │   │
│  │ SCC          │ GCP Security Command Center                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key Principles:                                                    │
│  • Scan IaC in CI/CD pipelines                                     │
│  • Implement defense in depth                                      │
│  • Monitor continuously                                            │
│  • Automate compliance checks                                      │
│                                                                     │
│  Next: Learn about CI/CD pipeline security                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
