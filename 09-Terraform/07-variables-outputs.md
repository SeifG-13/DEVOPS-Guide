# Terraform Variables and Outputs

## Input Variables

Input variables allow you to parameterize your Terraform configurations, making them reusable and flexible.

```
┌─────────────────────────────────────────────────────────────┐
│                    INPUT VARIABLES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Sources (Priority: Low → High)                             │
│                                                             │
│  1. Default value in variable block                         │
│  2. terraform.tfvars file                                   │
│  3. *.auto.tfvars files                                     │
│  4. -var-file flag                                          │
│  5. -var flag                                               │
│  6. TF_VAR_* environment variables                          │
│                                                             │
│            ▼                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  variable "instance_type" { ... }                   │   │
│  └─────────────────────────────────────────────────────┘   │
│            ▼                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  resource "aws_instance" "web" {                    │   │
│  │    instance_type = var.instance_type                │   │
│  │  }                                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Variable Declaration

### Basic Syntax

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}
```

### Complete Variable Block

```hcl
variable "instance_type" {
  description = "The type of EC2 instance to launch"
  type        = string
  default     = "t2.micro"

  # Validation rules
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }

  # Mark as sensitive (won't show in logs)
  sensitive = false

  # Allow null values
  nullable = false
}
```

---

## Variable Types

### Primitive Types

```hcl
# String
variable "environment" {
  type    = string
  default = "production"
}

# Number
variable "instance_count" {
  type    = number
  default = 3
}

# Boolean
variable "enable_monitoring" {
  type    = bool
  default = true
}
```

### Collection Types

```hcl
# List of strings
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# List of numbers
variable "ports" {
  type    = list(number)
  default = [80, 443, 8080]
}

# Set (unique values, unordered)
variable "allowed_ips" {
  type    = set(string)
  default = ["10.0.0.1", "10.0.0.2"]
}

# Map of strings
variable "tags" {
  type = map(string)
  default = {
    Environment = "production"
    Team        = "devops"
  }
}

# Map of numbers
variable "port_mapping" {
  type = map(number)
  default = {
    http  = 80
    https = 443
  }
}
```

### Structural Types

```hcl
# Object (specific structure)
variable "instance_config" {
  type = object({
    ami           = string
    instance_type = string
    disk_size     = number
    encrypted     = bool
  })
  default = {
    ami           = "ami-12345678"
    instance_type = "t2.micro"
    disk_size     = 20
    encrypted     = true
  }
}

# Tuple (fixed-length, mixed types)
variable "mixed_config" {
  type    = tuple([string, number, bool])
  default = ["web", 80, true]
}

# List of objects
variable "users" {
  type = list(object({
    name  = string
    email = string
    admin = bool
  }))
  default = [
    { name = "alice", email = "alice@example.com", admin = true },
    { name = "bob", email = "bob@example.com", admin = false }
  ]
}
```

### Optional Attributes (Terraform 1.3+)

```hcl
variable "server_config" {
  type = object({
    name     = string
    size     = optional(string, "t2.micro")  # Default if not provided
    enabled  = optional(bool)                 # Defaults to null
    tags     = optional(map(string), {})
  })
}

# Usage
server_config = {
  name = "web-server"
  # size defaults to "t2.micro"
  # enabled defaults to null
  # tags defaults to {}
}
```

---

## Setting Variable Values

### Default Values

```hcl
variable "region" {
  default = "us-east-1"
}
```

### terraform.tfvars

```hcl
# terraform.tfvars (auto-loaded)
region        = "us-west-2"
instance_type = "t3.micro"
environment   = "staging"

tags = {
  Project = "web-app"
  Owner   = "devops"
}

availability_zones = [
  "us-west-2a",
  "us-west-2b"
]
```

### Auto-loaded Files

```
# Files auto-loaded in order:
terraform.tfvars
terraform.tfvars.json
*.auto.tfvars
*.auto.tfvars.json

# Example: production.auto.tfvars
environment = "production"
instance_count = 5
```

### Command Line

```bash
# Single variable
terraform apply -var="instance_type=t3.large"

# Multiple variables
terraform apply \
  -var="instance_type=t3.large" \
  -var="environment=production"

# Variable file
terraform apply -var-file="production.tfvars"

# Multiple files
terraform apply \
  -var-file="common.tfvars" \
  -var-file="production.tfvars"
```

### Environment Variables

```bash
# Format: TF_VAR_<variable_name>
export TF_VAR_instance_type="t3.large"
export TF_VAR_environment="production"
export TF_VAR_region="us-west-2"

# For complex types, use JSON
export TF_VAR_tags='{"Environment":"production","Team":"devops"}'
```

---

## Variable Validation

### Custom Validation Rules

```hcl
variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Instance type must be t2.* or t3.* series."
  }
}

variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "cidr_block" {
  type = string

  validation {
    condition     = can(cidrnetmask(var.cidr_block))
    error_message = "Must be a valid CIDR block."
  }
}

# Multiple validations
variable "bucket_name" {
  type = string

  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "Bucket name must be between 3 and 63 characters."
  }

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9.-]*[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must start and end with alphanumeric character."
  }
}
```

---

## Sensitive Variables

### Marking as Sensitive

```hcl
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

variable "api_key" {
  description = "API key for external service"
  type        = string
  sensitive   = true
}
```

### Behavior

```bash
# Sensitive values are hidden in output
$ terraform plan
  + resource "aws_db_instance" "main" {
      + password = (sensitive value)
    }

# But stored in state file (encrypt your state!)
```

### Setting Sensitive Values

```bash
# Don't put in .tfvars file (committed to git)
# Use environment variables
export TF_VAR_db_password="supersecret"

# Or prompt at runtime (no default)
variable "db_password" {
  type      = string
  sensitive = true
  # No default = prompt required
}
```

---

## Using Variables

### In Resources

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  count         = var.instance_count

  tags = var.common_tags
}
```

### In Expressions

```hcl
# String interpolation
resource "aws_instance" "web" {
  tags = {
    Name = "${var.project_name}-${var.environment}-web"
  }
}

# Conditional
resource "aws_instance" "web" {
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"
}

# Lookup in map
resource "aws_instance" "web" {
  instance_type = var.instance_types[var.environment]
}

# Index in list
resource "aws_instance" "web" {
  availability_zone = var.availability_zones[0]
}
```

---

## Output Values

Outputs expose values from your configuration for use by other configurations or for display.

### Basic Output

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}
```

### Output Options

```hcl
output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.endpoint}/${var.db_name}"

  # Hide from console output
  sensitive = true
}

output "instance_ips" {
  description = "List of instance IPs"
  value       = aws_instance.web[*].public_ip

  # Add precondition
  precondition {
    condition     = length(aws_instance.web) > 0
    error_message = "No instances were created."
  }
}
```

### Viewing Outputs

```bash
# Show all outputs
terraform output

# Show specific output
terraform output instance_id

# Show as JSON
terraform output -json

# Show raw value (no quotes)
terraform output -raw instance_public_ip
```

### Using Outputs from Other Configurations

```hcl
# In another configuration, use remote state
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Access outputs
resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.network.outputs.public_subnet_id
}
```

---

## Local Values

Locals are computed values for use within a module.

### Basic Locals

```hcl
locals {
  environment = "production"
  project     = "web-app"

  # Computed values
  name_prefix = "${local.project}-${local.environment}"

  # Complex values
  common_tags = {
    Environment = local.environment
    Project     = local.project
    ManagedBy   = "Terraform"
  }
}
```

### Using Locals

```hcl
resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-instance"
  })
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.name_prefix}-data"
  tags   = local.common_tags
}
```

### Locals vs Variables

| Aspect | Variables | Locals |
|--------|-----------|--------|
| **Source** | External input | Internal computation |
| **Changeable** | Yes (at runtime) | No (fixed in code) |
| **Use Case** | User configuration | DRY, computed values |
| **Reference** | `var.name` | `local.name` |

---

## File Organization

### Recommended Structure

```
project/
├── variables.tf      # Variable declarations
├── terraform.tfvars  # Default values (git-ignored for secrets)
├── outputs.tf        # Output declarations
├── locals.tf         # Local values
└── main.tf           # Resources
```

### Example Files

```hcl
# variables.tf
variable "region" {
  description = "AWS region"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Invalid environment."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

# terraform.tfvars
region      = "us-east-1"
environment = "production"

# locals.tf
locals {
  name_prefix = "myapp-${var.environment}"
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# outputs.tf
output "instance_id" {
  value = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **variable** | Input parameter declaration |
| **default** | Default value if not provided |
| **type** | Data type constraint |
| **validation** | Custom validation rules |
| **sensitive** | Hide value in output |
| **output** | Export values |
| **local** | Computed internal values |

### Variable Reference

```hcl
var.variable_name        # Input variable
local.local_name         # Local value
module.name.output_name  # Module output
```

### Priority (Low to High)

1. Default in variable block
2. `terraform.tfvars`
3. `*.auto.tfvars`
4. `-var-file` flag
5. `-var` flag
6. `TF_VAR_*` environment variables
