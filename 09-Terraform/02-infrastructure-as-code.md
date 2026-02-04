# Infrastructure as Code (IaC)

## What is Infrastructure as Code?

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure through machine-readable configuration files rather than manual processes or interactive tools.

```
┌─────────────────────────────────────────────────────────────┐
│              TRADITIONAL vs IaC APPROACH                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TRADITIONAL (Manual)           IaC (Automated)             │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │ Click in        │           │ Write code      │         │
│  │ AWS Console     │           │ (main.tf)       │         │
│  └────────┬────────┘           └────────┬────────┘         │
│           │                             │                   │
│           ▼                             ▼                   │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │ Document steps  │           │ Version control │         │
│  │ (maybe)         │           │ (Git)           │         │
│  └────────┬────────┘           └────────┬────────┘         │
│           │                             │                   │
│           ▼                             ▼                   │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │ Repeat manually │           │ terraform apply │         │
│  │ for each env    │           │ (automated)     │         │
│  └─────────────────┘           └─────────────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Declarative vs Imperative

### Declarative (Terraform)

Define **what** you want - the system figures out **how**.

```hcl
# Declarative: Define desired state
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Terraform handles:
# - Creating instances if they don't exist
# - Updating if configuration changed
# - Determining order of operations
```

### Imperative (Scripts)

Define **how** to achieve the result step by step.

```bash
#!/bin/bash
# Imperative: Define each step

# Check if instance exists
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=web" --query 'Reservations[].Instances[].InstanceId' --output text)

if [ -z "$INSTANCE_ID" ]; then
    # Create if not exists
    aws ec2 run-instances --image-id ami-0c55b159 --instance-type t2.micro
else
    # Update logic here
    echo "Instance already exists"
fi
```

### Comparison

| Aspect | Declarative | Imperative |
|--------|-------------|------------|
| **Focus** | What (end state) | How (steps) |
| **Idempotency** | Built-in | Must implement |
| **Readability** | High | Varies |
| **Error Handling** | Automatic | Manual |
| **Learning Curve** | Lower | Higher |
| **Flexibility** | Less | More |

---

## Mutable vs Immutable Infrastructure

### Mutable Infrastructure (Traditional)

Servers are updated in place - like maintaining a pet.

```
┌─────────────────────────────────────────────────────────────┐
│                 MUTABLE INFRASTRUCTURE                       │
│                    (Pets Approach)                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Server v1.0                                                │
│  ┌─────────────────┐                                       │
│  │   Web Server    │                                       │
│  │   Nginx 1.18    │                                       │
│  │   App v1.0      │                                       │
│  └────────┬────────┘                                       │
│           │                                                 │
│           │ SSH + Update                                    │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │   Web Server    │  Same server, modified                │
│  │   Nginx 1.20    │  (configuration drift possible)       │
│  │   App v1.1      │                                       │
│  └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Servers are long-lived
- Updates applied in place
- Each server can become unique (snowflake)
- Configuration drift over time
- Debugging can be difficult

### Immutable Infrastructure (Modern)

Servers are never modified - replace with new version.

```
┌─────────────────────────────────────────────────────────────┐
│                IMMUTABLE INFRASTRUCTURE                      │
│                   (Cattle Approach)                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Server v1.0              Server v1.1                       │
│  ┌─────────────────┐      ┌─────────────────┐              │
│  │   Web Server    │      │   Web Server    │              │
│  │   Nginx 1.18    │      │   Nginx 1.20    │              │
│  │   App v1.0      │      │   App v1.1      │              │
│  └────────┬────────┘      └────────┬────────┘              │
│           │                        │                        │
│           │ Destroy        Deploy  │                        │
│           ▼                        ▼                        │
│       ┌───────┐            ┌─────────────────┐             │
│       │  🗑️   │            │  New Instance   │             │
│       │Delete │            │  (from image)   │             │
│       └───────┘            └─────────────────┘             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Servers are disposable
- Never updated, only replaced
- All servers are identical
- No configuration drift
- Easy rollback (deploy previous version)

### Pets vs Cattle

| Pets (Mutable) | Cattle (Immutable) |
|----------------|-------------------|
| Named servers (db-master-01) | Numbered servers (web-001, web-002) |
| Manually configured | Automated from image |
| Unique, irreplaceable | Identical, disposable |
| Fixed when sick | Replaced when sick |
| Vertical scaling | Horizontal scaling |

### Terraform and Immutable Infrastructure

```hcl
# Immutable pattern with Terraform
resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  image_id      = var.ami_id  # New AMI = new instances
  instance_type = "t2.micro"

  # When AMI changes, new instances are created
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "web" {
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  min_size = 2
  max_size = 10

  # Rolling update replaces instances
  instance_refresh {
    strategy = "Rolling"
  }
}
```

### When to Use Each

| Use Mutable When | Use Immutable When |
|------------------|-------------------|
| Legacy applications | Cloud-native applications |
| Database servers | Stateless web servers |
| Quick hotfixes needed | Consistent deployments needed |
| Limited automation | Full automation pipeline |
| Cost constraints | Scale and reliability priority |

---

## IaC Benefits

### 1. Version Control

```bash
# Track all infrastructure changes
git log --oneline
a1b2c3d Add production database
e4f5g6h Update instance type to t3.large
i7j8k9l Add auto-scaling group
```

### 2. Reproducibility

```hcl
# Same code = Same infrastructure
# Development
terraform apply -var="environment=dev"

# Staging
terraform apply -var="environment=staging"

# Production
terraform apply -var="environment=prod"
```

### 3. Documentation

```hcl
# Code IS documentation
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP and HTTPS traffic"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from anywhere"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from anywhere"
  }
}
```

### 4. Automation

```yaml
# CI/CD Pipeline
name: Infrastructure Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        run: terraform apply tfplan
```

### 5. Collaboration

```hcl
# Code review for infrastructure
# Pull Request: Add new database instance

resource "aws_db_instance" "new_db" {
  # Reviewer can see exactly what will be created
  identifier     = "app-database"
  engine         = "postgres"
  engine_version = "14.7"
  instance_class = "db.t3.medium"
  # ...
}
```

---

## IaC Workflow

### Standard Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                    IaC WORKFLOW                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. WRITE                                                   │
│     ┌─────────────────┐                                    │
│     │ Edit .tf files  │                                    │
│     │ in IDE          │                                    │
│     └────────┬────────┘                                    │
│              │                                              │
│  2. INIT     ▼                                              │
│     ┌─────────────────┐                                    │
│     │ terraform init  │ Download providers                 │
│     └────────┬────────┘                                    │
│              │                                              │
│  3. VALIDATE ▼                                              │
│     ┌─────────────────┐                                    │
│     │terraform validate│ Check syntax                      │
│     └────────┬────────┘                                    │
│              │                                              │
│  4. PLAN     ▼                                              │
│     ┌─────────────────┐                                    │
│     │ terraform plan  │ Preview changes                    │
│     └────────┬────────┘                                    │
│              │                                              │
│  5. APPLY    ▼                                              │
│     ┌─────────────────┐                                    │
│     │ terraform apply │ Execute changes                    │
│     └────────┬────────┘                                    │
│              │                                              │
│  6. DESTROY  ▼ (when needed)                               │
│     ┌─────────────────┐                                    │
│     │terraform destroy│ Remove resources                   │
│     └─────────────────┘                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Commands

```bash
# Initialize working directory
terraform init

# Validate configuration
terraform validate

# Format code
terraform fmt

# Preview changes
terraform plan

# Apply changes
terraform apply

# Destroy infrastructure
terraform destroy
```

---

## GitOps with IaC

### GitOps Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                    GITOPS WORKFLOW                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Developer                     CI/CD                        │
│  ┌─────────┐                  ┌─────────────────────────┐  │
│  │ 1. Edit │                  │                         │  │
│  │   code  │                  │  4. terraform plan      │  │
│  └────┬────┘                  │     (comment on PR)     │  │
│       │                       │                         │  │
│       ▼                       │  5. terraform apply     │  │
│  ┌─────────┐                  │     (on merge)          │  │
│  │ 2. Push │─────────────────►│                         │  │
│  │   to Git│                  └─────────────────────────┘  │
│  └────┬────┘                              │                 │
│       │                                   │                 │
│       ▼                                   ▼                 │
│  ┌─────────┐                  ┌─────────────────────────┐  │
│  │ 3. Open │                  │  Cloud Infrastructure   │  │
│  │   PR    │                  │                         │  │
│  └─────────┘                  └─────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Benefits of GitOps

| Benefit | Description |
|---------|-------------|
| **Audit Trail** | Git history shows who changed what and when |
| **Code Review** | PRs enable review of infrastructure changes |
| **Rollback** | Git revert to previous infrastructure state |
| **Consistency** | Single source of truth in Git |
| **Automation** | CI/CD applies changes automatically |

---

## IaC Tools Comparison

| Tool | Type | Language | State | Best For |
|------|------|----------|-------|----------|
| **Terraform** | Provisioning | HCL | Yes | Multi-cloud infrastructure |
| **CloudFormation** | Provisioning | YAML/JSON | AWS-managed | AWS-only shops |
| **Pulumi** | Provisioning | Python/TS/Go | Yes | Developers who prefer code |
| **Ansible** | Config Mgmt | YAML | No | Server configuration |
| **Chef** | Config Mgmt | Ruby | No | Complex configurations |
| **Puppet** | Config Mgmt | Puppet DSL | No | Large-scale config mgmt |

---

## Summary

| Concept | Description |
|---------|-------------|
| **IaC** | Managing infrastructure through code |
| **Declarative** | Define what, not how |
| **Imperative** | Define step-by-step how |
| **Mutable** | Update servers in place |
| **Immutable** | Replace servers, never update |
| **Pets** | Named, unique servers |
| **Cattle** | Numbered, disposable servers |
| **GitOps** | Git as source of truth for infra |

### Key Takeaways

1. **IaC enables** version control, automation, and reproducibility
2. **Declarative** approach is generally preferred for infrastructure
3. **Immutable infrastructure** reduces drift and improves reliability
4. **GitOps** brings software development practices to infrastructure
5. **Choose tools** based on your cloud strategy and team skills
