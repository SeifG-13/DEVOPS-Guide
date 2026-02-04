# Terraform Workspaces

## What are Workspaces?

Workspaces allow you to manage multiple distinct sets of infrastructure resources with the same Terraform configuration. Each workspace has its own state file.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM WORKSPACES                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Same Configuration (.tf files)                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  main.tf  │  variables.tf  │  outputs.tf           │   │
│  └─────────────────────────────────────────────────────┘   │
│           │               │               │                 │
│           ▼               ▼               ▼                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │  Workspace  │ │  Workspace  │ │  Workspace  │          │
│  │  "default"  │ │    "dev"    │ │   "prod"    │          │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘          │
│         │               │               │                   │
│         ▼               ▼               ▼                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ State File  │ │ State File  │ │ State File  │          │
│  │  (default)  │ │   (dev)     │ │   (prod)    │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Default Workspace

Every Terraform configuration has at least one workspace called `default`.

```bash
# Check current workspace
terraform workspace show
# Output: default

# List all workspaces
terraform workspace list
# Output:
# * default
```

---

## Workspace Commands

### Creating Workspaces

```bash
# Create new workspace
terraform workspace new dev
# Created and switched to workspace "dev"!

terraform workspace new staging
terraform workspace new production

# List workspaces (* indicates current)
terraform workspace list
#   default
#   dev
# * staging
#   production
```

### Switching Workspaces

```bash
# Switch to existing workspace
terraform workspace select dev
# Switched to workspace "dev"

terraform workspace select production
# Switched to workspace "production"
```

### Deleting Workspaces

```bash
# Delete workspace (must not be current)
terraform workspace select default
terraform workspace delete dev
# Deleted workspace "dev"!

# Force delete (even if state is not empty)
terraform workspace delete -force staging
```

### Show Current Workspace

```bash
terraform workspace show
# Output: production
```

---

## Using Workspace Name

### terraform.workspace Variable

```hcl
# Access current workspace name
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}
```

### Conditional Configuration

```hcl
# Different settings per workspace
locals {
  environment_config = {
    default = {
      instance_type = "t3.micro"
      instance_count = 1
    }
    dev = {
      instance_type = "t3.micro"
      instance_count = 1
    }
    staging = {
      instance_type = "t3.small"
      instance_count = 2
    }
    production = {
      instance_type = "t3.large"
      instance_count = 3
    }
  }

  config = local.environment_config[terraform.workspace]
}

resource "aws_instance" "web" {
  count         = local.config.instance_count
  ami           = var.ami_id
  instance_type = local.config.instance_type

  tags = {
    Name        = "web-${terraform.workspace}-${count.index}"
    Environment = terraform.workspace
  }
}
```

### Dynamic Resource Names

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "myapp-data-${terraform.workspace}"

  tags = {
    Environment = terraform.workspace
  }
}

resource "aws_db_instance" "main" {
  identifier = "myapp-db-${terraform.workspace}"
  # ...
}
```

---

## State File Location

### Local Backend

```bash
# With local backend, states are in:
terraform.tfstate.d/
├── dev/
│   └── terraform.tfstate
├── staging/
│   └── terraform.tfstate
└── production/
    └── terraform.tfstate

# Default workspace uses:
terraform.tfstate
```

### Remote Backend (S3)

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "myapp/terraform.tfstate"
    region = "us-east-1"
  }
}

# States stored as:
# s3://my-terraform-state/env:/dev/myapp/terraform.tfstate
# s3://my-terraform-state/env:/staging/myapp/terraform.tfstate
# s3://my-terraform-state/env:/production/myapp/terraform.tfstate
```

### Custom Key per Workspace

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "myapp/${terraform.workspace}/terraform.tfstate"  # Won't work!
    region = "us-east-1"
  }
}

# Note: You can't use terraform.workspace in backend config
# Terraform handles this automatically with "env:" prefix
```

---

## Workspace-Specific Variables

### Using tfvars Files

```bash
# Create workspace-specific variable files
# dev.tfvars
# staging.tfvars
# production.tfvars

# Apply with workspace-specific vars
terraform workspace select dev
terraform apply -var-file="${terraform.workspace}.tfvars"

# Or use a script
WORKSPACE=$(terraform workspace show)
terraform apply -var-file="${WORKSPACE}.tfvars"
```

### Variable Files Structure

```
project/
├── main.tf
├── variables.tf
├── dev.tfvars
├── staging.tfvars
└── production.tfvars
```

```hcl
# dev.tfvars
instance_type  = "t3.micro"
instance_count = 1
db_instance_class = "db.t3.micro"

# production.tfvars
instance_type  = "t3.large"
instance_count = 5
db_instance_class = "db.r5.large"
```

---

## Workspaces vs Directories

### Directory Structure Approach

```
infrastructure/
├── dev/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
└── production/
    ├── main.tf
    ├── variables.tf
    └── terraform.tfvars
```

### Comparison

| Aspect | Workspaces | Directories |
|--------|------------|-------------|
| **Code Duplication** | None | Potential duplication |
| **State Isolation** | Automatic | Manual |
| **Configuration Differences** | Limited (via workspace name) | Full flexibility |
| **Visibility** | Less obvious | Clear separation |
| **CI/CD** | More complex | Simpler |
| **Module Versions** | Same | Can differ |

### When to Use Workspaces

- Same infrastructure, different environments
- Quick environment switching
- Temporary/testing environments
- Simple environment differences

### When to Use Directories

- Significantly different configurations
- Different module versions per environment
- Separate teams managing environments
- Different approval processes

---

## Best Practices

### 1. Name Resources with Workspace

```hcl
resource "aws_instance" "web" {
  tags = {
    Name        = "${var.project}-${terraform.workspace}-web"
    Environment = terraform.workspace
    Project     = var.project
  }
}
```

### 2. Use Locals for Configuration

```hcl
locals {
  env_config = {
    dev = {
      instance_type = "t3.micro"
      min_size      = 1
      max_size      = 2
    }
    staging = {
      instance_type = "t3.small"
      min_size      = 2
      max_size      = 4
    }
    production = {
      instance_type = "t3.large"
      min_size      = 3
      max_size      = 10
    }
  }

  # Default to dev if workspace not found
  config = lookup(local.env_config, terraform.workspace, local.env_config.dev)
}
```

### 3. Validate Workspace

```hcl
locals {
  valid_workspaces = ["dev", "staging", "production"]
}

resource "null_resource" "validate_workspace" {
  count = contains(local.valid_workspaces, terraform.workspace) ? 0 : 1

  provisioner "local-exec" {
    command = "echo 'Invalid workspace: ${terraform.workspace}' && exit 1"
  }
}
```

### 4. Document Workspace Requirements

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region (set via workspace-specific .tfvars)"
  type        = string
}

# README.md
# Workspaces
# - dev: Development environment (dev.tfvars)
# - staging: Staging environment (staging.tfvars)
# - production: Production environment (production.tfvars)
#
# Usage:
# terraform workspace select dev
# terraform apply -var-file=dev.tfvars
```

---

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Terraform

on:
  push:
    branches: [main, develop]

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        workspace: [dev, staging, production]

    steps:
      - uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        run: terraform init

      - name: Select Workspace
        run: terraform workspace select ${{ matrix.workspace }} || terraform workspace new ${{ matrix.workspace }}

      - name: Terraform Plan
        run: terraform plan -var-file="${{ matrix.workspace }}.tfvars"

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve -var-file="${{ matrix.workspace }}.tfvars"
```

### Makefile Example

```makefile
WORKSPACE ?= dev

.PHONY: init plan apply destroy

init:
	terraform init
	terraform workspace select $(WORKSPACE) || terraform workspace new $(WORKSPACE)

plan: init
	terraform plan -var-file=$(WORKSPACE).tfvars

apply: init
	terraform apply -var-file=$(WORKSPACE).tfvars

destroy: init
	terraform destroy -var-file=$(WORKSPACE).tfvars

# Usage: make plan WORKSPACE=production
```

---

## Complete Example

```hcl
# main.tf
terraform {
  required_version = ">= 1.0.0"

  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "myapp/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}

locals {
  # Environment-specific configurations
  environments = {
    dev = {
      instance_type  = "t3.micro"
      instance_count = 1
      domain_prefix  = "dev"
    }
    staging = {
      instance_type  = "t3.small"
      instance_count = 2
      domain_prefix  = "staging"
    }
    production = {
      instance_type  = "t3.large"
      instance_count = 3
      domain_prefix  = "www"
    }
  }

  env = local.environments[terraform.workspace]

  common_tags = {
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
    Project     = var.project_name
  }
}

resource "aws_instance" "web" {
  count = local.env.instance_count

  ami           = var.ami_id
  instance_type = local.env.instance_type

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${terraform.workspace}-web-${count.index + 1}"
  })
}

output "instance_ips" {
  value = aws_instance.web[*].public_ip
}

output "workspace" {
  value = terraform.workspace
}
```

---

## Summary

| Command | Description |
|---------|-------------|
| `terraform workspace list` | List all workspaces |
| `terraform workspace show` | Show current workspace |
| `terraform workspace new <name>` | Create new workspace |
| `terraform workspace select <name>` | Switch workspace |
| `terraform workspace delete <name>` | Delete workspace |

### Key Points

1. **Each workspace** has its own state file
2. **Use `terraform.workspace`** to access workspace name
3. **Workspaces are NOT** for completely different configurations
4. **Consider directories** for significantly different environments
5. **Name resources** with workspace for clarity
6. **Use tfvars files** for workspace-specific variables
