# Terraform Best Practices

## Project Structure

### Recommended Layout

```
terraform-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   └── ...
│   └── production/
│       └── ...
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   └── database/
└── global/
    └── iam/
```

### Single Environment Layout

```
project/
├── main.tf           # Main resources
├── variables.tf      # Variable declarations
├── outputs.tf        # Output declarations
├── providers.tf      # Provider configuration
├── versions.tf       # Version constraints
├── locals.tf         # Local values
├── data.tf           # Data sources
├── terraform.tfvars  # Variable values
└── backend.tf        # Backend configuration
```

---

## Naming Conventions

### Resources

```hcl
# Use descriptive, lowercase names with underscores
resource "aws_instance" "web_server" { }      # Good
resource "aws_instance" "WebServer" { }       # Avoid
resource "aws_instance" "ws1" { }             # Avoid

# Include environment/purpose in name
resource "aws_s3_bucket" "app_logs_production" { }

# Use consistent prefixes
resource "aws_security_group" "web_sg" { }
resource "aws_security_group" "api_sg" { }
resource "aws_security_group" "db_sg" { }
```

### Variables

```hcl
# Use lowercase with underscores
variable "instance_type" { }    # Good
variable "instanceType" { }     # Avoid
variable "InstanceType" { }     # Avoid

# Be descriptive
variable "web_server_instance_type" { }
variable "database_backup_retention_days" { }

# Prefix related variables
variable "vpc_cidr" { }
variable "vpc_name" { }
variable "vpc_enable_dns" { }
```

### Files

```
main.tf              # Primary resources
variables.tf         # All variable declarations
outputs.tf           # All outputs
providers.tf         # Provider configuration
versions.tf          # Version constraints
locals.tf            # Local values
data.tf              # Data sources
<resource-type>.tf   # Grouped by resource type (optional)
```

---

## State Management

### Always Use Remote State

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "project/environment/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Enable Locking

```hcl
# DynamoDB for S3
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### State Per Environment

```
s3://terraform-state/
├── dev/terraform.tfstate
├── staging/terraform.tfstate
└── production/terraform.tfstate
```

### Never Edit State Manually

```bash
# Use terraform commands
terraform state mv
terraform state rm
terraform import

# Never:
# vim terraform.tfstate
```

---

## Version Pinning

### Pin Terraform Version

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"
}
```

### Pin Provider Versions

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5.0"
    }
  }
}
```

### Pin Module Versions

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"  # Exact version
}
```

### Commit Lock File

```bash
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars        # If contains secrets
crash.log

# DO commit
.terraform.lock.hcl  # Provider lock file
```

---

## Module Design

### Keep Modules Focused

```hcl
# Good: Single responsibility
modules/
├── vpc/           # Only VPC resources
├── security/      # Only security groups
├── compute/       # Only compute resources
└── database/      # Only database resources

# Avoid: Kitchen sink modules
modules/
└── infrastructure/  # Everything!
```

### Use defaults/ for Configurable Options

```hcl
# modules/web-server/variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"  # Sensible default
}
```

### Document Modules

```markdown
# modules/vpc/README.md

## VPC Module

Creates a VPC with public and private subnets.

### Usage

```hcl
module "vpc" {
  source = "./modules/vpc"

  cidr_block = "10.0.0.0/16"
  name       = "production"
}
```

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| cidr_block | VPC CIDR block | string | n/a | yes |

### Outputs

| Name | Description |
|------|-------------|
| vpc_id | ID of the VPC |
```

---

## Security Best Practices

### Never Hardcode Secrets

```hcl
# NEVER do this
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"  # NO!
  secret_key = "wJalrXUtnFEMI..."       # NO!
}

# Use environment variables
# export AWS_ACCESS_KEY_ID="..."
# export AWS_SECRET_ACCESS_KEY="..."

# Or use IAM roles (best)
provider "aws" {
  region = "us-east-1"
  # Uses instance role or CLI credentials
}
```

### Mark Sensitive Variables

```hcl
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

output "db_connection_string" {
  value     = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}"
  sensitive = true
}
```

### Use Secrets Manager

```hcl
# Fetch secrets from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/database/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

### Encrypt State

```hcl
terraform {
  backend "s3" {
    bucket     = "terraform-state"
    key        = "terraform.tfstate"
    encrypt    = true
    kms_key_id = "arn:aws:kms:us-east-1:123456789:key/abc-123"
  }
}
```

---

## Code Quality

### Use terraform fmt

```bash
# Format all files
terraform fmt -recursive

# Check in CI/CD
terraform fmt -check -recursive
```

### Use terraform validate

```bash
# Validate syntax
terraform validate
```

### Use Linting Tools

```bash
# Install tflint
brew install tflint

# Run linter
tflint

# With specific ruleset
tflint --enable-rule=terraform_naming_convention
```

### Use Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs
```

---

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Terraform

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.7.0

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

### Plan Review Workflow

```yaml
# On PR: Plan only, post as comment
- name: Terraform Plan
  id: plan
  run: terraform plan -no-color
  continue-on-error: true

- name: Comment Plan on PR
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: '```terraform\n${{ steps.plan.outputs.stdout }}\n```'
      })
```

---

## Immutable Infrastructure Patterns

### Use AMIs Instead of Provisioners

```hcl
# Good: Pre-built AMI
resource "aws_instance" "web" {
  ami = data.aws_ami.app.id  # AMI built with Packer
}

# Avoid: Provisioners
resource "aws_instance" "web" {
  ami = "ami-base"

  provisioner "remote-exec" {
    inline = ["apt-get install...", "configure..."]
  }
}
```

### Use Launch Templates

```hcl
resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "web" {
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
  }
}
```

---

## Documentation

### Comment Complex Logic

```hcl
# Calculate subnet CIDR blocks based on VPC CIDR
# Splits /16 into /24 subnets across availability zones
locals {
  subnet_cidrs = [
    for idx, az in var.availability_zones :
    cidrsubnet(var.vpc_cidr, 8, idx)
  ]
}
```

### Use Descriptions

```hcl
variable "instance_type" {
  description = "EC2 instance type. Use t3.micro for dev, t3.large for production."
  type        = string
  default     = "t3.micro"
}

output "load_balancer_dns" {
  description = "DNS name of the load balancer for the web application"
  value       = aws_lb.web.dns_name
}
```

### Generate Documentation

```bash
# Using terraform-docs
terraform-docs markdown table . > README.md

# Auto-generate on pre-commit
- id: terraform_docs
```

---

## Summary Checklist

### Project Setup
- [ ] Use consistent directory structure
- [ ] Separate environments
- [ ] Use modules for reusability
- [ ] Commit .terraform.lock.hcl

### State Management
- [ ] Use remote state
- [ ] Enable locking
- [ ] Enable encryption
- [ ] State per environment

### Security
- [ ] No hardcoded secrets
- [ ] Mark sensitive variables
- [ ] Use secrets manager
- [ ] Encrypt state at rest

### Code Quality
- [ ] Run terraform fmt
- [ ] Run terraform validate
- [ ] Use tflint
- [ ] Use pre-commit hooks

### CI/CD
- [ ] Automate plan/apply
- [ ] Require plan review on PRs
- [ ] Protect production branch
- [ ] Use separate credentials

### Documentation
- [ ] Document modules
- [ ] Use descriptions
- [ ] Comment complex logic
- [ ] Generate README with terraform-docs

---

## Quick Reference

### Do's

```hcl
✓ Use remote state with locking
✓ Pin versions (Terraform, providers, modules)
✓ Use modules for reusability
✓ Mark sensitive variables
✓ Use consistent naming conventions
✓ Review plans before applying
✓ Use workspaces or directories for environments
```

### Don'ts

```hcl
✗ Don't hardcode secrets
✗ Don't edit state manually
✗ Don't use local state in teams
✗ Don't skip version pinning
✗ Don't apply without reviewing plan
✗ Don't use provisioners when alternatives exist
✗ Don't commit .tfvars with secrets
```
