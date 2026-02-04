# Terraform Import, Taint, and State Commands

## State Commands Overview

Terraform provides commands to inspect and manipulate state directly. Use these carefully as improper use can corrupt your state.

```
┌─────────────────────────────────────────────────────────────┐
│                   STATE COMMANDS                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  terraform state <subcommand>                               │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │    list     │  │    show     │  │     mv      │        │
│  │ List all    │  │ Show detail │  │ Move/rename │        │
│  │ resources   │  │ of resource │  │ resources   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │     rm      │  │    pull     │  │    push     │        │
│  │ Remove from │  │ Download    │  │ Upload      │        │
│  │ state       │  │ state       │  │ state       │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## terraform state list

List all resources in the state.

```bash
# List all resources
terraform state list

# Output:
# aws_instance.web
# aws_security_group.web_sg
# aws_vpc.main
# aws_subnet.public[0]
# aws_subnet.public[1]
# module.database.aws_db_instance.main

# Filter by resource type
terraform state list aws_instance.*

# Filter by module
terraform state list module.database.*

# List with specific state file
terraform state list -state=backup.tfstate
```

---

## terraform state show

Show detailed information about a resource.

```bash
# Show resource details
terraform state show aws_instance.web

# Output:
# resource "aws_instance" "web" {
#     ami                          = "ami-12345678"
#     arn                          = "arn:aws:ec2:us-east-1:123456789:instance/i-abc123"
#     id                           = "i-abc123"
#     instance_type                = "t2.micro"
#     private_ip                   = "10.0.1.50"
#     public_ip                    = "54.123.45.67"
#     tags                         = {
#         "Name" = "web-server"
#     }
#     ...
# }

# Show indexed resource
terraform state show 'aws_subnet.public[0]'

# Show resource in module
terraform state show module.database.aws_db_instance.main
```

---

## terraform state mv

Move or rename resources in state.

### Rename Resource

```bash
# Rename a resource
terraform state mv aws_instance.web aws_instance.webserver

# Before: resource "aws_instance" "web" { ... }
# After:  resource "aws_instance" "webserver" { ... }

# Update your .tf file to match!
```

### Move to Module

```bash
# Move resource into a module
terraform state mv aws_instance.web module.compute.aws_instance.web

# Now referenced as: module.compute.aws_instance.web
```

### Move Between Modules

```bash
# Move from one module to another
terraform state mv \
  module.old_module.aws_instance.web \
  module.new_module.aws_instance.web
```

### Move Indexed Resources

```bash
# Move specific index
terraform state mv 'aws_instance.web[0]' 'aws_instance.web["primary"]'

# Move all to new name
terraform state mv aws_instance.old aws_instance.new
```

### Dry Run

```bash
# Preview move without making changes
terraform state mv -dry-run aws_instance.web aws_instance.webserver
```

---

## terraform state rm

Remove resources from state without destroying them.

```bash
# Remove single resource
terraform state rm aws_instance.web

# Resource still exists in cloud but Terraform won't manage it!

# Remove indexed resource
terraform state rm 'aws_subnet.public[0]'

# Remove entire module
terraform state rm module.database

# Dry run
terraform state rm -dry-run aws_instance.web
```

### Use Case: Handoff to Different Config

```bash
# Remove from this config
terraform state rm aws_instance.web

# Import in another config
cd ../other-project
terraform import aws_instance.web i-abc123
```

---

## terraform state pull

Download and display the current state.

```bash
# Pull state to stdout
terraform state pull

# Save to file
terraform state pull > backup.tfstate

# Pretty print with jq
terraform state pull | jq '.resources'

# Get specific value
terraform state pull | jq '.resources[] | select(.type == "aws_instance")'
```

---

## terraform state push

Upload a local state file to remote backend.

```bash
# Push state (DANGEROUS!)
terraform state push backup.tfstate

# Force push (even more dangerous!)
terraform state push -force backup.tfstate
```

**Warning:** This can overwrite remote state and cause data loss!

---

## terraform import

Import existing infrastructure into Terraform state.

### Basic Import

```bash
# Syntax
terraform import <resource_address> <resource_id>

# Import EC2 instance
terraform import aws_instance.web i-abc123def456

# Import S3 bucket
terraform import aws_s3_bucket.data my-bucket-name

# Import security group
terraform import aws_security_group.web sg-12345678
```

### Import Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                   IMPORT WORKFLOW                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Write Resource Block (empty or minimal)                 │
│     ┌─────────────────────────────────────────────────┐    │
│     │ resource "aws_instance" "web" {                 │    │
│     │   # Configuration will be filled in            │    │
│     │ }                                               │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
│  2. Run Import Command                                      │
│     terraform import aws_instance.web i-abc123              │
│                                                             │
│  3. Run Plan to See Current State                           │
│     terraform plan                                          │
│     # Shows attributes that need to be added               │
│                                                             │
│  4. Update Configuration to Match                           │
│     ┌─────────────────────────────────────────────────┐    │
│     │ resource "aws_instance" "web" {                 │    │
│     │   ami           = "ami-12345678"                │    │
│     │   instance_type = "t2.micro"                    │    │
│     │   # ... other attributes from plan output      │    │
│     │ }                                               │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
│  5. Verify with Plan                                        │
│     terraform plan                                          │
│     # Should show: No changes                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Import Examples

```bash
# VPC
terraform import aws_vpc.main vpc-abc123

# Subnet
terraform import aws_subnet.public subnet-abc123

# Security Group
terraform import aws_security_group.web sg-abc123

# IAM User
terraform import aws_iam_user.admin admin-user

# RDS Instance
terraform import aws_db_instance.main mydb-instance

# Route53 Zone
terraform import aws_route53_zone.main Z1234567890ABC

# Azure Resource Group
terraform import azurerm_resource_group.main /subscriptions/xxx/resourceGroups/mygroup

# GCP Instance
terraform import google_compute_instance.vm projects/myproject/zones/us-central1-a/instances/myvm
```

---

## Import Block (Terraform 1.5+)

Declarative import using configuration.

### Basic Import Block

```hcl
# Import existing instance
import {
  to = aws_instance.web
  id = "i-abc123def456"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  # ... other attributes
}
```

### Run Import

```bash
# Plan with import
terraform plan

# Apply imports
terraform apply
```

### Generate Configuration

```bash
# Generate config for imported resources (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf

# Creates generated.tf with resource configurations
```

### Multiple Imports

```hcl
import {
  to = aws_instance.web
  id = "i-abc123"
}

import {
  to = aws_security_group.web
  id = "sg-def456"
}

import {
  to = aws_vpc.main
  id = "vpc-789xyz"
}
```

---

## terraform taint (Deprecated)

Mark a resource for recreation on next apply.

```bash
# Taint resource (deprecated in Terraform 0.15.2+)
terraform taint aws_instance.web

# Resource will be destroyed and recreated on next apply
```

### Modern Replacement: -replace Flag

```bash
# Replace specific resource
terraform apply -replace="aws_instance.web"

# Replace multiple resources
terraform apply \
  -replace="aws_instance.web" \
  -replace="aws_instance.api"

# Plan with replacement
terraform plan -replace="aws_instance.web"
```

---

## terraform untaint

Remove taint from a resource.

```bash
# Untaint resource
terraform untaint aws_instance.web

# Resource will no longer be recreated
```

---

## Practical Scenarios

### Scenario 1: Rename Resource

```bash
# 1. Rename in state
terraform state mv aws_instance.old_name aws_instance.new_name

# 2. Update your .tf file
# Change: resource "aws_instance" "old_name"
# To:     resource "aws_instance" "new_name"

# 3. Verify
terraform plan
# Should show: No changes
```

### Scenario 2: Move to Module

```bash
# 1. Create module structure
mkdir -p modules/compute

# 2. Move resource definition to module
# Move resource block to modules/compute/main.tf

# 3. Add module block in root
# module "compute" {
#   source = "./modules/compute"
# }

# 4. Move state
terraform state mv aws_instance.web module.compute.aws_instance.web

# 5. Verify
terraform plan
```

### Scenario 3: Import Existing Infrastructure

```bash
# 1. Write minimal resource block
cat >> main.tf << 'EOF'
resource "aws_instance" "existing" {
}
EOF

# 2. Import
terraform import aws_instance.existing i-abc123

# 3. Get current configuration
terraform state show aws_instance.existing

# 4. Update main.tf with actual values

# 5. Verify
terraform plan
# Should show: No changes
```

### Scenario 4: Split State

```bash
# Move resources to new state file
terraform state mv -state-out=new.tfstate aws_instance.web

# In new project, use new.tfstate as starting point
cd new-project
terraform init
terraform state push new.tfstate
```

---

## State Command Safety

### Always Backup First

```bash
# Before any state manipulation
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate
```

### Use Dry Run

```bash
# Preview changes
terraform state mv -dry-run aws_instance.old aws_instance.new
terraform state rm -dry-run aws_instance.web
```

### Lock State During Changes

```bash
# State operations automatically lock
# Force unlock if needed (use with caution!)
terraform force-unlock LOCK_ID
```

---

## Summary

| Command | Purpose |
|---------|---------|
| `state list` | List resources in state |
| `state show` | Show resource details |
| `state mv` | Move/rename resources |
| `state rm` | Remove from state |
| `state pull` | Download state |
| `state push` | Upload state |
| `import` | Import existing resource |
| `taint` | Mark for recreation (deprecated) |
| `apply -replace` | Replace resource |

### Import Methods

| Method | Version | Description |
|--------|---------|-------------|
| `terraform import` | All | CLI command |
| `import {}` block | 1.5+ | Declarative import |
| `-generate-config-out` | 1.5+ | Auto-generate config |

### Best Practices

1. **Always backup** state before manipulation
2. **Use -dry-run** to preview changes
3. **Update .tf files** to match state changes
4. **Run plan** after changes to verify
5. **Use import blocks** over CLI import when possible
6. **Use -replace** instead of taint
