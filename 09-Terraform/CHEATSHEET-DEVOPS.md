# Terraform Cheat Sheet for DevOps Engineers

## Quick Reference Commands

### Basic Workflow
```bash
terraform init                      # Initialize working directory
terraform plan                      # Preview changes
terraform apply                     # Apply changes
terraform destroy                   # Destroy infrastructure

# Common flags
terraform plan -out=plan.tfplan     # Save plan to file
terraform apply plan.tfplan         # Apply saved plan
terraform apply -auto-approve       # Skip confirmation
terraform apply -var="env=prod"     # Pass variable
terraform apply -var-file="prod.tfvars"
terraform plan -target=aws_instance.web  # Target specific resource
```

### State Management
```bash
terraform state list                # List resources in state
terraform state show aws_instance.web    # Show resource details
terraform state mv old_name new_name     # Rename resource
terraform state rm aws_instance.web      # Remove from state
terraform state pull                # Pull remote state
terraform state push                # Push local state
terraform refresh                   # Sync state with real infrastructure
```

### Workspace Management
```bash
terraform workspace list            # List workspaces
terraform workspace new dev         # Create workspace
terraform workspace select prod     # Switch workspace
terraform workspace show            # Show current workspace
terraform workspace delete dev      # Delete workspace
```

### Other Commands
```bash
terraform fmt                       # Format code
terraform fmt -check               # Check formatting
terraform validate                  # Validate configuration
terraform output                    # Show outputs
terraform output -json              # JSON format
terraform graph | dot -Tpng > graph.png  # Visualize
terraform import aws_instance.web i-1234  # Import existing resource
terraform taint aws_instance.web    # Mark for recreation
terraform untaint aws_instance.web  # Remove taint
```

---

## Configuration Syntax

### Basic Structure
```hcl
# Provider configuration
provider "aws" {
  region = "us-east-1"
}

# Resource definition
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = "WebServer"
  }
}

# Data source (read existing resources)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Output
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

### Variables
```hcl
# variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

variable "allowed_ports" {
  type    = list(number)
  default = [80, 443]
}

variable "tags" {
  type = map(string)
  default = {
    Project = "MyApp"
  }
}

# Complex type
variable "server_config" {
  type = object({
    name     = string
    size     = string
    replicas = number
  })
}

# Using variables
resource "aws_instance" "web" {
  count         = var.instance_count
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
}
```

### terraform.tfvars
```hcl
environment    = "production"
instance_count = 3
allowed_ports  = [80, 443, 8080]

tags = {
  Project     = "MyApp"
  Environment = "Production"
}
```

### Locals
```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  name_prefix = "${var.project_name}-${var.environment}"
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}
```

---

## Advanced Features

### Count and For Each
```hcl
# Count
resource "aws_instance" "web" {
  count         = 3
  ami           = var.ami_id
  instance_type = "t2.micro"

  tags = {
    Name = "web-${count.index}"
  }
}

# For Each with list
resource "aws_instance" "web" {
  for_each      = toset(["web1", "web2", "web3"])
  ami           = var.ami_id
  instance_type = "t2.micro"

  tags = {
    Name = each.key
  }
}

# For Each with map
variable "instances" {
  default = {
    web  = "t2.micro"
    api  = "t2.small"
    db   = "t2.medium"
  }
}

resource "aws_instance" "servers" {
  for_each      = var.instances
  ami           = var.ami_id
  instance_type = each.value

  tags = {
    Name = each.key
  }
}
```

### Dynamic Blocks
```hcl
resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### Conditionals
```hcl
resource "aws_instance" "web" {
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"

  # Conditional resource creation
  count = var.create_instance ? 1 : 0
}
```

### Modules
```hcl
# Using module
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr     = "10.0.0.0/16"
  environment  = var.environment
}

# Module outputs
output "vpc_id" {
  value = module.vpc.vpc_id
}

# Remote module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

---

## Backend Configuration

### S3 Backend (AWS)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Azure Storage Backend
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstatestorage"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

---

## Interview Q&A

### Q1: What is Terraform state and why is it important?
**A:** State is a JSON file that maps real-world resources to configuration. It:
- Tracks resource metadata
- Improves performance (caches attributes)
- Enables collaboration (remote state)
- Determines what changes are needed

### Q2: What is the difference between terraform plan and apply?
**A:**
- **plan**: Shows what changes will be made (dry run)
- **apply**: Actually creates/modifies/destroys resources

### Q3: How do you handle secrets in Terraform?
**A:**
- Mark variables as sensitive: `sensitive = true`
- Use environment variables: `TF_VAR_db_password`
- Use secret managers (Vault, AWS Secrets Manager)
- Never commit secrets to version control
- Use encrypted state backend

### Q4: What is a Terraform module?
**A:** Reusable, self-contained package of Terraform configurations. Benefits:
- Code reuse
- Abstraction
- Organization
- Versioning

### Q5: What is the difference between count and for_each?
**A:**
- **count**: Creates numbered instances, referenced by index
- **for_each**: Creates named instances, referenced by key
Use for_each when resources need stable identifiers.

### Q6: How do you import existing resources into Terraform?
**A:**
```bash
# 1. Write resource configuration
# 2. Import the resource
terraform import aws_instance.web i-1234567890abcdef0

# 3. Run plan to verify
terraform plan
```

### Q7: What is remote state and why use it?
**A:** State stored in a shared location (S3, Azure Storage). Benefits:
- Team collaboration
- State locking (prevents conflicts)
- Secure storage (encryption)
- Backup/versioning

### Q8: How do you manage multiple environments?
**A:**
- **Workspaces**: Same config, different state
- **Directory structure**: Separate directories per environment
- **tfvars files**: Environment-specific variables
- **Modules**: Reusable components across environments

### Q9: What is terraform taint?
**A:** Marks a resource for recreation on next apply. Useful when resource is corrupted or needs to be rebuilt.
```bash
terraform taint aws_instance.web
```

### Q10: How do you handle dependencies between resources?
**A:**
- **Implicit**: Reference another resource's attribute
- **Explicit**: Use `depends_on` meta-argument
```hcl
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id  # Implicit
  depends_on = [aws_internet_gateway.gw]  # Explicit
}
```

---

## Best Practices

### File Structure
```
terraform/
├── main.tf              # Main resources
├── variables.tf         # Variable declarations
├── outputs.tf           # Output values
├── providers.tf         # Provider configuration
├── terraform.tfvars     # Variable values
├── backend.tf           # Backend configuration
└── modules/
    └── vpc/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Code Quality
1. **Use consistent formatting**: `terraform fmt`
2. **Validate before apply**: `terraform validate`
3. **Use meaningful names**: Descriptive resource names
4. **Tag everything**: Include common tags
5. **Use modules**: Don't repeat yourself
6. **Version lock providers**: Prevent breaking changes
7. **Use remote state**: Enable collaboration
8. **Implement state locking**: Prevent concurrent modifications

### Security
```hcl
# Mark sensitive variables
variable "db_password" {
  type      = string
  sensitive = true
}

# Don't output sensitive values
output "password" {
  value     = var.db_password
  sensitive = true
}
```

---

## Provider Lock
```hcl
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}
```
