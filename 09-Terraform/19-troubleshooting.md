# Terraform Troubleshooting

## Debug Logging

### TF_LOG Environment Variable

```bash
# Set log level
export TF_LOG=DEBUG

# Log levels (most to least verbose):
# TRACE - Most detailed
# DEBUG - Debugging information
# INFO  - General information
# WARN  - Warnings
# ERROR - Errors only

# Run with debug logging
TF_LOG=DEBUG terraform plan
```

### Log to File

```bash
# Set log file path
export TF_LOG=DEBUG
export TF_LOG_PATH="./terraform.log"

# Run command
terraform plan

# View logs
cat terraform.log
```

### Provider-Specific Logging

```bash
# Log only provider operations
export TF_LOG_PROVIDER=DEBUG

# Log core Terraform operations
export TF_LOG_CORE=DEBUG
```

### Debug Example

```bash
$ TF_LOG=DEBUG terraform plan 2>&1 | head -50

2024-01-15T10:30:00.000Z [DEBUG] Adding temp file log sink: /tmp/terraform-log123
2024-01-15T10:30:00.001Z [INFO]  Terraform version: 1.7.0
2024-01-15T10:30:00.002Z [DEBUG] Provider registry.terraform.io/hashicorp/aws
2024-01-15T10:30:00.003Z [DEBUG] checking for provider in "."
...
```

---

## terraform console

Interactive expression testing.

```bash
# Start console
terraform console

# Test expressions
> var.instance_type
"t2.micro"

> local.common_tags
{
  "Environment" = "production"
  "ManagedBy" = "Terraform"
}

> length(var.subnets)
3

> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> aws_instance.web.public_ip
"54.123.45.67"

# Exit
> exit
```

### Console with State

```bash
# Access state values
terraform console

> aws_instance.web.id
"i-abc123def456"

> aws_vpc.main.cidr_block
"10.0.0.0/16"

> module.vpc.public_subnet_ids
[
  "subnet-abc123",
  "subnet-def456",
]
```

---

## terraform validate

Check configuration syntax.

```bash
# Validate configuration
terraform validate

# Success output:
# Success! The configuration is valid.

# Error output:
# Error: Missing required argument
#   on main.tf line 5, in resource "aws_instance" "web":
#    5: resource "aws_instance" "web" {
# The argument "ami" is required, but no definition was found.
```

### Validate JSON Output

```bash
terraform validate -json

# Output:
{
  "valid": false,
  "error_count": 1,
  "warning_count": 0,
  "diagnostics": [
    {
      "severity": "error",
      "summary": "Missing required argument",
      "detail": "The argument \"ami\" is required..."
    }
  ]
}
```

---

## terraform fmt

Format and check code style.

```bash
# Format all files
terraform fmt

# Check formatting only (don't modify)
terraform fmt -check

# Show diff
terraform fmt -diff

# Recursive (all subdirectories)
terraform fmt -recursive
```

---

## Common Errors and Solutions

### Error: State Lock

```
Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        abc123-def456
  Path:      terraform.tfstate
  Operation: OperationTypePlan
  Who:       user@hostname
  Version:   1.7.0
  Created:   2024-01-15 10:30:00.000000000 +0000 UTC
```

**Solutions:**

```bash
# Wait for other operation to complete

# Force unlock (if stuck)
terraform force-unlock abc123-def456

# Check who has the lock
terraform state pull | jq '.lineage'
```

### Error: Provider Configuration

```
Error: No valid credential sources found

  with provider["registry.terraform.io/hashicorp/aws"],
  on providers.tf line 1, in provider "aws":
   1: provider "aws" {
```

**Solutions:**

```bash
# Set environment variables
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_REGION="us-east-1"

# Or use AWS CLI profile
export AWS_PROFILE="my-profile"

# Or configure in provider
provider "aws" {
  region  = "us-east-1"
  profile = "my-profile"
}
```

### Error: Resource Not Found

```
Error: Error reading resource: ResourceNotFoundException

  The resource aws_instance.web with ID i-abc123 was not found.
```

**Solutions:**

```bash
# Remove from state if resource was deleted externally
terraform state rm aws_instance.web

# Or refresh state
terraform refresh

# Or import if resource exists with different ID
terraform import aws_instance.web i-new-id
```

### Error: Dependency Cycle

```
Error: Cycle: aws_security_group.a, aws_security_group.b
```

**Solutions:**

```hcl
# Break the cycle using separate rules

# Instead of inline rules causing cycles:
resource "aws_security_group" "a" {
  ingress {
    security_groups = [aws_security_group.b.id]  # Cycle!
  }
}

# Use separate security group rules:
resource "aws_security_group" "a" {
  name = "sg-a"
}

resource "aws_security_group_rule" "a_from_b" {
  type                     = "ingress"
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
}
```

### Error: Invalid Value

```
Error: Invalid value for variable

  on variables.tf line 5:
   5: variable "instance_type" {

The given value is not valid for variable "instance_type":
Invalid instance type.
```

**Solutions:**

```hcl
# Check validation rules
variable "instance_type" {
  validation {
    condition     = contains(["t2.micro", "t2.small"], var.instance_type)
    error_message = "Must be t2.micro or t2.small."
  }
}

# Use valid value
terraform apply -var="instance_type=t2.micro"
```

### Error: Backend Initialization

```
Error: Backend initialization required, please run "terraform init"

Reason: Initial configuration of the requested backend "s3"
```

**Solutions:**

```bash
# Initialize backend
terraform init

# Reconfigure backend
terraform init -reconfigure

# Migrate state to new backend
terraform init -migrate-state
```

### Error: Provider Version Constraint

```
Error: Unsatisfied provider version constraint

  on versions.tf line 5:
   5:       version = "~> 4.0"

Provider registry.terraform.io/hashicorp/aws v5.0.0 does not match
constraint ~> 4.0.
```

**Solutions:**

```hcl
# Update version constraint
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Updated
    }
  }
}

# Then reinitialize
terraform init -upgrade
```

---

## State Issues

### Corrupted State

```bash
# Pull current state
terraform state pull > backup.tfstate

# Check if valid JSON
cat backup.tfstate | jq .

# If corrupted, restore from backup
# S3 versioning example:
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix terraform.tfstate

# Restore previous version
aws s3api get-object \
  --bucket my-terraform-state \
  --key terraform.tfstate \
  --version-id "version-id" \
  restored.tfstate

terraform state push restored.tfstate
```

### State Drift

```bash
# Detect drift
terraform plan

# Shows differences between state and reality

# Refresh state to match reality
terraform apply -refresh-only

# Or update resources to match state
terraform apply
```

### Missing Resources in State

```bash
# Import missing resource
terraform import aws_instance.web i-abc123

# Remove from state if no longer needed
terraform state rm aws_instance.old
```

---

## Plan Analysis

### Reading Plan Output

```bash
$ terraform plan

# Symbols:
#   + create
#   - destroy
#   ~ update in-place
#   -/+ destroy and recreate (replacement)
#   <= read (data source)

# Example:
  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
        id            = "i-abc123"
      ~ instance_type = "t2.micro" -> "t2.small"
        tags          = {
            "Name" = "web"
        }
    }
```

### Save Plan for Review

```bash
# Save plan to file
terraform plan -out=tfplan

# Show saved plan
terraform show tfplan

# Show as JSON
terraform show -json tfplan | jq .

# Apply saved plan
terraform apply tfplan
```

---

## Debugging Tips

### 1. Isolate the Problem

```bash
# Test specific resource
terraform plan -target=aws_instance.web

# Test specific module
terraform plan -target=module.vpc
```

### 2. Check Provider Documentation

```bash
# View module documentation
terraform providers schema -json | jq '.provider_schemas["registry.terraform.io/hashicorp/aws"]'

# View resource documentation
terraform providers schema -json | jq '.provider_schemas["registry.terraform.io/hashicorp/aws"].resource_schemas.aws_instance'
```

### 3. Verify Variables

```bash
# Console to test variables
terraform console

> var.instance_type
> local.config
> data.aws_ami.ubuntu.id
```

### 4. Check Dependencies

```bash
# Generate dependency graph
terraform graph | dot -Tpng > graph.png

# View as text
terraform graph
```

### 5. Clean Slate

```bash
# Remove cached providers/modules
rm -rf .terraform/

# Reinitialize
terraform init
```

---

## Performance Issues

### Slow Plans

```bash
# Refresh specific resources only
terraform plan -refresh=false -target=aws_instance.web

# Use parallelism
terraform apply -parallelism=20

# Enable provider caching
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
```

### Large State Files

```bash
# Check state size
terraform state pull | wc -c

# Consider splitting state into modules
# Each module can have its own state

# Use -target for partial operations
terraform plan -target=module.specific
```

---

## Summary

| Tool | Purpose |
|------|---------|
| `TF_LOG` | Debug logging |
| `terraform console` | Test expressions |
| `terraform validate` | Check syntax |
| `terraform fmt` | Format code |
| `terraform graph` | View dependencies |
| `-target` | Isolate resources |
| `terraform show` | View state/plan |

### Debug Checklist

1. ✓ Enable debug logging (`TF_LOG=DEBUG`)
2. ✓ Validate configuration (`terraform validate`)
3. ✓ Check provider credentials
4. ✓ Verify variable values (`terraform console`)
5. ✓ Review state (`terraform state list`)
6. ✓ Check for cycles (`terraform graph`)
7. ✓ Test specific resources (`-target`)
8. ✓ Review provider documentation
