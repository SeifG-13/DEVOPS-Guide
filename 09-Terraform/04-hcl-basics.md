# HCL Basics (HashiCorp Configuration Language)

## What is HCL?

HCL (HashiCorp Configuration Language) is a structured configuration language designed to be both human-readable and machine-friendly. It's used by Terraform and other HashiCorp tools.

```hcl
# HCL is designed to be readable
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name        = "WebServer"
    Environment = "production"
  }
}
```

---

## Basic Syntax

### Arguments and Blocks

```hcl
# Block syntax
block_type "label1" "label2" {
  # Arguments (key = value)
  argument1 = "value1"
  argument2 = 123

  # Nested block
  nested_block {
    nested_argument = "nested_value"
  }
}
```

### Examples

```hcl
# Resource block (two labels)
resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Provider block (one label)
provider "aws" {
  region = "us-east-1"
}

# Variable block (one label)
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

# Output block (one label)
output "instance_ip" {
  value = aws_instance.example.public_ip
}

# Terraform block (no labels)
terraform {
  required_version = ">= 1.0.0"
}
```

---

## Comments

```hcl
# Single-line comment (hash)

// Single-line comment (double slash)

/*
  Multi-line
  comment
  block
*/

# Comments are ignored by Terraform
resource "aws_instance" "web" {
  ami           = "ami-12345678"  # Inline comment
  instance_type = "t2.micro"      // Also works
}
```

---

## Data Types

### Primitive Types

```hcl
# String
name = "web-server"
name = "Hello, ${var.name}!"  # Interpolation

# Number
count = 5
ratio = 3.14

# Boolean
enabled = true
disabled = false

# Null
optional_value = null
```

### Complex Types

```hcl
# List (ordered collection of same type)
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

# Access by index
first_az = var.availability_zones[0]  # "us-east-1a"

# Map (key-value pairs)
tags = {
  Name        = "web-server"
  Environment = "production"
  Team        = "devops"
}

# Access by key
env = var.tags["Environment"]  # "production"
env = var.tags.Environment     # Alternative syntax

# Set (unordered unique values)
unique_ports = toset([80, 443, 8080])

# Object (structured type with named attributes)
server_config = {
  name          = "web-01"
  instance_type = "t2.micro"
  disk_size     = 100
}

# Tuple (fixed-length sequence with specific types)
mixed_data = ["string", 123, true]
```

### Type Declarations

```hcl
variable "string_var" {
  type = string
}

variable "number_var" {
  type = number
}

variable "bool_var" {
  type = bool
}

variable "list_var" {
  type = list(string)
}

variable "map_var" {
  type = map(string)
}

variable "set_var" {
  type = set(number)
}

variable "object_var" {
  type = object({
    name    = string
    age     = number
    enabled = bool
  })
}

variable "tuple_var" {
  type = tuple([string, number, bool])
}

# Any type (flexible)
variable "any_var" {
  type = any
}

# Optional attributes (Terraform 1.3+)
variable "config" {
  type = object({
    name     = string
    optional = optional(string, "default")
  })
}
```

---

## Block Types

### terraform Block

```hcl
terraform {
  # Required Terraform version
  required_version = ">= 1.0.0"

  # Required providers
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }

  # Backend configuration
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "state/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### provider Block

```hcl
# Default provider configuration
provider "aws" {
  region = "us-east-1"
}

# Aliased provider (for multi-region)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Using aliased provider
resource "aws_instance" "west_instance" {
  provider = aws.west
  # ...
}
```

### resource Block

```hcl
resource "resource_type" "resource_name" {
  # Required arguments
  argument1 = "value1"

  # Optional arguments
  argument2 = "value2"

  # Nested blocks
  nested_block {
    setting = "value"
  }

  # Meta-arguments
  count      = 3
  depends_on = [aws_vpc.main]

  lifecycle {
    create_before_destroy = true
  }

  # Tags (common pattern)
  tags = {
    Name = "example"
  }
}
```

### variable Block

```hcl
variable "instance_type" {
  description = "The type of EC2 instance"
  type        = string
  default     = "t2.micro"

  # Validation rule
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }

  # Mark as sensitive
  sensitive = false

  # Allow null value
  nullable = true
}
```

### output Block

```hcl
output "instance_ip" {
  description = "The public IP of the instance"
  value       = aws_instance.web.public_ip

  # Mark as sensitive (won't show in console)
  sensitive = false

  # Only output if condition is met
  precondition {
    condition     = aws_instance.web.public_ip != ""
    error_message = "Instance doesn't have a public IP."
  }
}
```

### data Block

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Use the data source
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}
```

### locals Block

```hcl
locals {
  # Simple values
  environment = "production"

  # Computed values
  instance_name = "${var.project}-${local.environment}-web"

  # Complex values
  common_tags = {
    Environment = local.environment
    Project     = var.project
    ManagedBy   = "Terraform"
  }
}

# Use locals
resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = local.instance_name
  })
}
```

### module Block

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  # Module inputs
  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Use module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnets[0]
}
```

---

## Expressions

### References

```hcl
# Variable reference
var.instance_type

# Local value reference
local.common_tags

# Resource attribute reference
aws_instance.web.public_ip
aws_instance.web.id

# Data source reference
data.aws_ami.ubuntu.id

# Module output reference
module.vpc.vpc_id

# Count index
aws_instance.web[0].id
aws_instance.web[count.index].id

# For_each key/value
aws_instance.web["web1"].id
each.key
each.value

# Path references
path.module   # Current module path
path.root     # Root module path
path.cwd      # Current working directory

# Terraform workspace
terraform.workspace
```

### Operators

```hcl
# Arithmetic
sum = 1 + 2        # Addition
diff = 5 - 3       # Subtraction
product = 2 * 3    # Multiplication
quotient = 10 / 2  # Division
remainder = 10 % 3 # Modulo

# Comparison
equal = 1 == 1         # true
not_equal = 1 != 2     # true
less_than = 1 < 2      # true
greater_than = 2 > 1   # true
less_or_equal = 1 <= 1 # true
greater_or_equal = 2 >= 2 # true

# Logical
and_result = true && false  # false
or_result = true || false   # true
not_result = !true          # false
```

### String Interpolation

```hcl
# Simple interpolation
name = "Hello, ${var.username}!"

# Expression interpolation
greeting = "You have ${var.count > 0 ? var.count : "no"} messages"

# Heredoc syntax
description = <<-EOT
  This is a multi-line string.
  It can span multiple lines.
  Variables work: ${var.name}
EOT

# Heredoc without interpolation
script = <<-'EOT'
  echo "No interpolation: ${this_stays_literal}"
EOT
```

---

## File Organization

### Recommended File Structure

```
project/
├── main.tf           # Main resources
├── variables.tf      # Variable declarations
├── outputs.tf        # Output declarations
├── providers.tf      # Provider configuration
├── terraform.tf      # Terraform settings
├── locals.tf         # Local values
├── data.tf           # Data sources
└── terraform.tfvars  # Variable values
```

### Example Files

```hcl
# terraform.tf
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# providers.tf
provider "aws" {
  region = var.region
}

# variables.tf
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

# main.tf
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  tags          = local.common_tags
}

# outputs.tf
output "instance_ip" {
  value = aws_instance.web.public_ip
}

# locals.tf
locals {
  common_tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}

# data.tf
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
}

# terraform.tfvars
region        = "us-west-2"
instance_type = "t3.micro"
```

---

## Formatting

### terraform fmt

```bash
# Format all files in current directory
terraform fmt

# Format recursively
terraform fmt -recursive

# Check formatting (don't modify)
terraform fmt -check

# Show diff of formatting changes
terraform fmt -diff
```

### Style Conventions

```hcl
# Use 2-space indentation
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Align equals signs in blocks
resource "aws_instance" "web" {
  ami                    = "ami-12345678"
  instance_type          = "t2.micro"
  monitoring             = true
  vpc_security_group_ids = [aws_security_group.web.id]
}

# Use blank lines to separate logical sections
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## Summary

| Element | Syntax |
|---------|--------|
| **Block** | `type "label" { }` |
| **Argument** | `name = value` |
| **Comment** | `#`, `//`, `/* */` |
| **String** | `"text"` or `"${var.x}"` |
| **Number** | `42`, `3.14` |
| **Boolean** | `true`, `false` |
| **List** | `["a", "b", "c"]` |
| **Map** | `{ key = "value" }` |
| **Reference** | `var.x`, `local.y`, `resource.attr` |

### Key Block Types

| Block | Purpose |
|-------|---------|
| `terraform` | Version and backend config |
| `provider` | Provider configuration |
| `resource` | Infrastructure objects |
| `variable` | Input parameters |
| `output` | Export values |
| `data` | Query existing resources |
| `locals` | Computed local values |
| `module` | Reusable configurations |
