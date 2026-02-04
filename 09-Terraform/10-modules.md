# Terraform Modules

## What are Modules?

Modules are containers for multiple resources that are used together. They allow you to create reusable, shareable, and maintainable infrastructure components.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM MODULES                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Root Module (Your Configuration)                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  main.tf                                            │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │ module "vpc" {                              │   │   │
│  │  │   source = "./modules/vpc"                  │   │   │
│  │  │   cidr   = "10.0.0.0/16"                   │   │   │
│  │  │ }                                           │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  Child Module (./modules/vpc)                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  variables.tf  │  main.tf  │  outputs.tf           │   │
│  │       │              │            │                 │   │
│  │       ▼              ▼            ▼                 │   │
│  │   Inputs ──► Resources ──► Outputs                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Module Structure

### Standard Layout

```
module-name/
├── main.tf           # Main resources
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── versions.tf       # Provider/Terraform versions
├── README.md         # Documentation
├── examples/         # Usage examples
│   └── basic/
│       └── main.tf
└── tests/            # Module tests
```

### Minimal Module

```hcl
# modules/s3-bucket/variables.tf
variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

# modules/s3-bucket/main.tf
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  tags = {
    Name        = var.bucket_name
    Environment = var.environment
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

# modules/s3-bucket/outputs.tf
output "bucket_id" {
  description = "ID of the S3 bucket"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.this.arn
}
```

---

## Using Modules

### Local Modules

```hcl
module "bucket" {
  source = "./modules/s3-bucket"

  bucket_name = "my-app-data"
  environment = "production"
}

# Access module outputs
resource "aws_iam_policy" "bucket_access" {
  name = "bucket-access"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:*"]
      Resource = [module.bucket.bucket_arn]
    }]
  })
}
```

### Registry Modules

```hcl
# From Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Environment = "production"
  }
}
```

### Git Modules

```hcl
# From Git repository
module "app" {
  source = "git::https://github.com/org/terraform-modules.git//app?ref=v1.0.0"

  app_name = "myapp"
}

# SSH URL
module "app" {
  source = "git::git@github.com:org/terraform-modules.git//app?ref=v1.0.0"

  app_name = "myapp"
}

# Specific branch or tag
module "app" {
  source = "git::https://github.com/org/modules.git//app?ref=develop"
}
```

### Other Sources

```hcl
# S3 bucket
module "app" {
  source = "s3::https://s3-us-east-1.amazonaws.com/bucket/module.zip"
}

# HTTP URL
module "app" {
  source = "https://example.com/modules/app.zip"
}

# GitHub shorthand
module "app" {
  source = "github.com/org/repo//modules/app?ref=v1.0.0"
}

# Bitbucket shorthand
module "app" {
  source = "bitbucket.org/org/repo//modules/app"
}
```

---

## Module Inputs

### Passing Variables

```hcl
module "web" {
  source = "./modules/ec2-instance"

  # Required inputs
  ami_id        = "ami-12345678"
  instance_type = "t2.micro"

  # Optional inputs
  instance_count = 3
  enable_monitoring = true

  # Complex inputs
  security_group_ids = [
    aws_security_group.web.id,
    aws_security_group.ssh.id
  ]

  tags = {
    Environment = "production"
    Team        = "web"
  }
}
```

### Module Variable Definition

```hcl
# modules/ec2-instance/variables.tf

variable "ami_id" {
  description = "AMI ID for the instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1
}

variable "security_group_ids" {
  description = "List of security group IDs"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Tags to apply"
  type        = map(string)
  default     = {}
}
```

---

## Module Outputs

### Exposing Values

```hcl
# modules/ec2-instance/outputs.tf

output "instance_ids" {
  description = "IDs of created instances"
  value       = aws_instance.this[*].id
}

output "public_ips" {
  description = "Public IPs of instances"
  value       = aws_instance.this[*].public_ip
}

output "private_ips" {
  description = "Private IPs of instances"
  value       = aws_instance.this[*].private_ip
}
```

### Using Module Outputs

```hcl
module "web" {
  source = "./modules/ec2-instance"
  # ... inputs
}

# Reference outputs
resource "aws_lb_target_group_attachment" "web" {
  count            = length(module.web.instance_ids)
  target_group_arn = aws_lb_target_group.web.arn
  target_id        = module.web.instance_ids[count.index]
}

# In outputs.tf
output "web_server_ips" {
  value = module.web.public_ips
}
```

---

## Module Versioning

### Version Constraints

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"

  # Exact version
  version = "5.0.0"

  # Minimum version
  version = ">= 5.0.0"

  # Range
  version = ">= 5.0.0, < 6.0.0"

  # Pessimistic constraint
  version = "~> 5.0"    # >= 5.0.0, < 6.0.0
  version = "~> 5.0.0"  # >= 5.0.0, < 5.1.0
}
```

### Git Refs

```hcl
# Tag
module "app" {
  source = "git::https://github.com/org/repo.git?ref=v1.0.0"
}

# Branch
module "app" {
  source = "git::https://github.com/org/repo.git?ref=develop"
}

# Commit SHA
module "app" {
  source = "git::https://github.com/org/repo.git?ref=abc1234"
}
```

---

## Multiple Module Instances

### Using count

```hcl
variable "environments" {
  default = ["dev", "staging", "prod"]
}

module "vpc" {
  source   = "./modules/vpc"
  count    = length(var.environments)

  name = "vpc-${var.environments[count.index]}"
  cidr = "10.${count.index}.0.0/16"
}

# Access: module.vpc[0], module.vpc[1], etc.
output "vpc_ids" {
  value = module.vpc[*].vpc_id
}
```

### Using for_each

```hcl
variable "projects" {
  default = {
    web = {
      cidr = "10.0.0.0/16"
      env  = "production"
    }
    api = {
      cidr = "10.1.0.0/16"
      env  = "production"
    }
  }
}

module "vpc" {
  source   = "./modules/vpc"
  for_each = var.projects

  name = "vpc-${each.key}"
  cidr = each.value.cidr
  environment = each.value.env
}

# Access: module.vpc["web"], module.vpc["api"]
output "vpc_ids" {
  value = { for k, v in module.vpc : k => v.vpc_id }
}
```

---

## Module Composition

### Nested Modules

```hcl
# Root module calls environment module
module "production" {
  source = "./modules/environment"

  environment = "production"
  region      = "us-east-1"
}

# modules/environment/main.tf
# Environment module calls VPC and compute modules
module "vpc" {
  source = "../vpc"

  name = "${var.environment}-vpc"
  cidr = "10.0.0.0/16"
}

module "compute" {
  source = "../compute"

  vpc_id    = module.vpc.vpc_id
  subnet_id = module.vpc.public_subnets[0]
}
```

### Passing Between Modules

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "security" {
  source = "./modules/security"
  vpc_id = module.vpc.vpc_id  # Pass VPC ID
}

module "compute" {
  source = "./modules/compute"

  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.public_subnet_ids
  security_group_ids = module.security.security_group_ids
}
```

---

## Terraform Registry

### Finding Modules

- Browse: [registry.terraform.io](https://registry.terraform.io)
- Search for specific functionality
- Check verified badge for official modules

### Popular Modules

| Module | Description |
|--------|-------------|
| terraform-aws-modules/vpc/aws | AWS VPC |
| terraform-aws-modules/eks/aws | AWS EKS cluster |
| terraform-aws-modules/rds/aws | AWS RDS database |
| terraform-azurerm-modules/vnet/azurerm | Azure VNet |
| terraform-google-modules/network/google | GCP Network |

### Using Registry Modules

```hcl
# Registry source format: namespace/name/provider
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.0.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
}
```

---

## Publishing Modules

### Module Requirements

```
module-name/
├── main.tf           # Required
├── variables.tf      # Required (even if empty)
├── outputs.tf        # Required (even if empty)
├── README.md         # Required
└── LICENSE           # Recommended
```

### README Template

```markdown
# Module Name

Brief description.

## Usage

```hcl
module "example" {
  source = "namespace/name/provider"
  version = "1.0.0"

  input1 = "value"
}
```

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.0.0 |
| aws | >= 5.0.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| input1 | Description | string | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| output1 | Description |
```

---

## Best Practices

### 1. Keep Modules Focused

```hcl
# Good - Single responsibility
modules/
├── vpc/          # Only VPC resources
├── security/     # Only security groups
└── compute/      # Only compute resources

# Avoid - Too broad
modules/
└── infrastructure/  # Everything!
```

### 2. Use Semantic Versioning

```
v1.0.0 - Initial release
v1.1.0 - New feature (backward compatible)
v1.1.1 - Bug fix
v2.0.0 - Breaking change
```

### 3. Document Everything

```hcl
variable "instance_type" {
  description = "EC2 instance type. See AWS documentation for available types."
  type        = string
  default     = "t3.micro"

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Instance type must be t2 or t3 series."
  }
}
```

### 4. Provide Examples

```
modules/vpc/
├── examples/
│   ├── simple/
│   │   └── main.tf
│   └── complete/
│       └── main.tf
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Module** | Container for related resources |
| **Source** | Where to find module code |
| **Inputs** | Variables passed to module |
| **Outputs** | Values exposed by module |
| **Version** | Pin module to specific version |

### Module Sources

| Source | Example |
|--------|---------|
| Local | `./modules/vpc` |
| Registry | `terraform-aws-modules/vpc/aws` |
| GitHub | `github.com/org/repo//path` |
| Git | `git::https://github.com/org/repo.git` |
| S3 | `s3::https://bucket.s3.amazonaws.com/module.zip` |
