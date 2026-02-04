# Terraform Cloud

## What is Terraform Cloud?

Terraform Cloud is HashiCorp's managed service for Terraform that provides collaboration, governance, and automation features for teams.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM CLOUD                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Web Interface                      │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │Workspaces│ │  Runs   │ │ Teams   │ │Registry │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│  ┌────────────────────────┼────────────────────────────┐   │
│  │                        ▼                            │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              Remote Execution               │   │   │
│  │  │                                             │   │   │
│  │  │  • Runs Terraform plans/applies             │   │   │
│  │  │  • Manages state files                      │   │   │
│  │  │  • Provides locking                         │   │   │
│  │  │  • Stores variables and secrets             │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Features:                                                  │
│  • Remote state management                                  │
│  • Remote runs (plan/apply)                                 │
│  • VCS integration (GitHub, GitLab, etc.)                  │
│  • Private module registry                                  │
│  • Policy as code (Sentinel)                               │
│  • Cost estimation                                          │
│  • Team management & RBAC                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Terraform Cloud vs Enterprise

| Feature | Cloud (Free) | Cloud (Team) | Enterprise |
|---------|--------------|--------------|------------|
| **Remote State** | ✓ | ✓ | ✓ |
| **Remote Runs** | ✓ | ✓ | ✓ |
| **VCS Integration** | ✓ | ✓ | ✓ |
| **Private Registry** | ✓ | ✓ | ✓ |
| **Team Management** | Limited | ✓ | ✓ |
| **SSO/SAML** | ✗ | ✓ | ✓ |
| **Sentinel Policies** | ✗ | ✓ | ✓ |
| **Audit Logging** | ✗ | ✓ | ✓ |
| **Self-Hosted** | ✗ | ✗ | ✓ |

---

## Getting Started

### 1. Create Account

1. Go to [app.terraform.io](https://app.terraform.io)
2. Sign up for a free account
3. Create an organization

### 2. Configure CLI

```bash
# Login to Terraform Cloud
terraform login

# Opens browser for authentication
# Token saved to ~/.terraform.d/credentials.tfrc.json
```

### 3. Configure Backend

```hcl
terraform {
  cloud {
    organization = "my-organization"

    workspaces {
      name = "my-workspace"
    }
  }
}

# Or with tags for multiple workspaces
terraform {
  cloud {
    organization = "my-organization"

    workspaces {
      tags = ["app:myapp"]
    }
  }
}
```

### Legacy Backend Configuration

```hcl
# Still supported but cloud block is preferred
terraform {
  backend "remote" {
    organization = "my-organization"

    workspaces {
      name = "my-workspace"
    }
  }
}
```

---

## Workspaces

### Creating Workspaces

**Via UI:**
1. Go to Workspaces → New Workspace
2. Choose workflow type
3. Configure VCS or CLI-driven

**Via CLI:**
```bash
# With cloud block configured
terraform init
# Workspace created automatically if it doesn't exist
```

### Workspace Types

| Type | Description |
|------|-------------|
| **VCS-driven** | Triggered by Git commits |
| **CLI-driven** | Triggered by terraform commands |
| **API-driven** | Triggered via API |

### VCS-Driven Workspace

```hcl
# Configuration in Terraform Cloud UI:
# - VCS Provider: GitHub
# - Repository: myorg/infrastructure
# - Working Directory: terraform/production
# - Auto-apply: Enabled/Disabled
```

### CLI-Driven Workspace

```bash
# Run locally, execute remotely
terraform init
terraform plan   # Runs in Terraform Cloud
terraform apply  # Runs in Terraform Cloud
```

---

## Variables

### Setting Variables

**In UI:**
- Workspace → Variables
- Add Terraform variables or Environment variables

**Variable Types:**

```hcl
# Terraform Variables (terraform.tfvars equivalent)
# Key: instance_type
# Value: t3.micro
# Category: Terraform variable

# Environment Variables
# Key: AWS_ACCESS_KEY_ID
# Value: AKIAIOSFODNN7EXAMPLE
# Category: Environment variable
# Sensitive: Yes
```

### Variable Sets

Share variables across workspaces.

```
Variable Set: "AWS Credentials"
├── AWS_ACCESS_KEY_ID (sensitive)
├── AWS_SECRET_ACCESS_KEY (sensitive)
└── AWS_REGION

Applied to:
├── production-infrastructure
├── staging-infrastructure
└── dev-infrastructure
```

### Sensitive Variables

```hcl
# Mark as sensitive in UI
# Key: db_password
# Value: ••••••••
# Sensitive: ✓ (checked)

# Cannot be viewed after save
# Shown as (sensitive) in logs
```

---

## Runs

### Run Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                    RUN WORKFLOW                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Pending                                                 │
│     └── Waiting in queue                                    │
│                                                             │
│  2. Planning                                                │
│     └── Running terraform plan                              │
│                                                             │
│  3. Cost Estimation (if enabled)                            │
│     └── Estimating infrastructure costs                     │
│                                                             │
│  4. Policy Check (if Sentinel configured)                   │
│     └── Evaluating policies                                 │
│                                                             │
│  5. Planned                                                 │
│     └── Waiting for confirmation                            │
│                                                             │
│  6. Applying                                                │
│     └── Running terraform apply                             │
│                                                             │
│  7. Applied / Errored                                       │
│     └── Run complete                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Speculative Plans

Plans that don't apply changes - useful for pull requests.

```bash
# Speculative plan (PR preview)
terraform plan  # With VCS integration on PRs
```

### Run Triggers

Automatically trigger runs when dependent workspaces change.

```
Workspace: networking
    │
    │ triggers
    ▼
Workspace: compute
    │
    │ triggers
    ▼
Workspace: application
```

---

## VCS Integration

### Supported Providers

- GitHub (and GitHub Enterprise)
- GitLab (and GitLab Self-Managed)
- Bitbucket Cloud (and Server)
- Azure DevOps

### Configuration

1. **Connect VCS Provider:**
   - Settings → VCS Providers → Add VCS Provider
   - Follow OAuth setup

2. **Configure Workspace:**
   ```
   VCS Repository: myorg/infrastructure
   Working Directory: terraform/
   VCS Branch: main
   Automatic Speculative Plans: Enabled
   ```

### Workflow

```
Developer                    Terraform Cloud
    │                              │
    │ Push to branch              │
    │ ─────────────────────────►  │
    │                              │ Speculative Plan
    │                              │
    │ Open Pull Request           │
    │ ─────────────────────────►  │
    │                              │ Plan posted as PR comment
    │                              │
    │ Merge to main               │
    │ ─────────────────────────►  │
    │                              │ Run triggered
    │                              │ Plan → Apply
```

---

## Private Module Registry

### Publishing Modules

**From VCS:**
1. Registry → Publish Module
2. Select VCS provider
3. Choose repository (must follow naming: `terraform-<PROVIDER>-<NAME>`)

**Module Naming:**
```
terraform-aws-vpc
terraform-azure-network
terraform-google-compute
```

### Using Private Modules

```hcl
module "vpc" {
  source  = "app.terraform.io/my-organization/vpc/aws"
  version = "1.0.0"

  cidr_block = "10.0.0.0/16"
}
```

---

## Sentinel Policies

Policy as Code for governance.

### Policy Example

```python
# require-tags.sentinel
import "tfplan/v2" as tfplan

# Get all AWS instances
aws_instances = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_instance" and
    rc.mode is "managed" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

# Check for required tags
required_tags = ["Environment", "Owner", "Project"]

check_tags = rule {
    all aws_instances as _, instance {
        all required_tags as tag {
            instance.change.after.tags contains tag
        }
    }
}

main = rule {
    check_tags
}
```

### Policy Sets

```
Policy Set: "AWS Standards"
├── require-tags.sentinel
├── restrict-instance-types.sentinel
└── enforce-encryption.sentinel

Enforcement Level: hard-mandatory
Applied to: All AWS workspaces
```

### Enforcement Levels

| Level | Behavior |
|-------|----------|
| `advisory` | Warn but allow apply |
| `soft-mandatory` | Require override to apply |
| `hard-mandatory` | Block apply entirely |

---

## Cost Estimation

Automatic cost estimation for supported resources.

```
Run Cost Estimation
───────────────────
Resources: 5 of 12 estimated

Monthly Cost: $156.24
Hourly Cost: $0.21

Changes:
  + aws_instance.web: $24.00/month
  + aws_rds_instance.db: $125.00/month
  ~ aws_instance.app: $0.00 (no change)
```

---

## API and Automation

### API Token

```bash
# Create API token in UI:
# User Settings → Tokens → Create API token

# Use in automation
export TF_TOKEN_app_terraform_io="your-token-here"
```

### API Example

```bash
# List workspaces
curl \
  --header "Authorization: Bearer $TF_TOKEN" \
  https://app.terraform.io/api/v2/organizations/my-org/workspaces

# Trigger run
curl \
  --header "Authorization: Bearer $TF_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data '{"data":{"type":"runs","relationships":{"workspace":{"data":{"type":"workspaces","id":"ws-xxxxx"}}}}}' \
  https://app.terraform.io/api/v2/runs
```

### CI/CD Integration

```yaml
# GitHub Actions
name: Terraform

on:
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: hashicorp/setup-terraform@v2
        with:
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan
```

---

## Teams and Permissions

### Team Permissions

| Permission | Description |
|------------|-------------|
| **Read** | View workspace, state, runs |
| **Plan** | Queue plans |
| **Write** | Apply runs, lock/unlock |
| **Admin** | Full workspace control |

### Organization Permissions

| Role | Capabilities |
|------|-------------|
| **Owners** | Full organization access |
| **Members** | Access based on team membership |

### Example Team Structure

```
Organization: ACME Corp
├── Team: Platform
│   ├── Permission: Admin on infrastructure-*
│   └── Members: alice, bob
├── Team: Developers
│   ├── Permission: Plan on app-*
│   └── Members: charlie, dave
└── Team: Security
    ├── Permission: Read on *
    └── Members: eve
```

---

## Best Practices

### 1. Workspace Naming

```
# Pattern: <app>-<environment>
web-app-dev
web-app-staging
web-app-production

# Or with component
networking-production
compute-production
database-production
```

### 2. Variable Management

```
# Use Variable Sets for shared credentials
Variable Set: "AWS Production"
├── AWS_ACCESS_KEY_ID
├── AWS_SECRET_ACCESS_KEY
└── AWS_REGION

# Workspace-specific in workspace variables
instance_type = "t3.large"
```

### 3. State Management

```hcl
# Always use cloud block
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-workspace"
    }
  }
}
```

### 4. Run Triggers for Dependencies

```
networking (VPC, subnets)
    │
    └── triggers → compute (EC2, ASG)
                       │
                       └── triggers → application (deployments)
```

---

## Summary

| Feature | Description |
|---------|-------------|
| **Remote State** | Secure, shared state storage |
| **Remote Runs** | Execute Terraform in cloud |
| **VCS Integration** | GitOps workflow |
| **Private Registry** | Share modules internally |
| **Sentinel** | Policy as code |
| **Cost Estimation** | Predict infrastructure costs |
| **Teams** | RBAC and collaboration |

### Configuration Block

```hcl
terraform {
  cloud {
    organization = "org-name"

    workspaces {
      name = "workspace-name"
      # OR
      tags = ["app:myapp", "env:prod"]
    }
  }
}
```
