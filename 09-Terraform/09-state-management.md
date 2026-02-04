# Terraform State Management

## What is Terraform State?

Terraform state is a JSON file that maps your configuration to real-world resources. It's essential for Terraform to track, plan, and manage infrastructure.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM STATE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Configuration        State File         Real Infrastructure│
│  (.tf files)          (.tfstate)         (Cloud Resources)  │
│                                                             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
│  │ resource    │     │ {           │     │   EC2       │   │
│  │ "aws_       │◄───►│  "id":      │◄───►│   Instance  │   │
│  │ instance"   │     │  "i-123..." │     │   i-123...  │   │
│  │ "web" {...} │     │ }           │     │             │   │
│  └─────────────┘     └─────────────┘     └─────────────┘   │
│                                                             │
│  Desired State        Recorded State     Actual State       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Why State is Important

### 1. Mapping to Real Resources

```json
{
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",
            "ami": "ami-12345678",
            "instance_type": "t2.micro"
          }
        }
      ]
    }
  ]
}
```

### 2. Tracking Metadata

- Resource dependencies
- Provider configurations
- Version information

### 3. Performance Optimization

- Caches attribute values
- Reduces API calls
- Faster plan generation

### 4. Collaboration

- Team members share state
- Prevents conflicting changes
- Provides locking mechanism

---

## State File Structure

### terraform.tfstate

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 42,
  "lineage": "abc123-def456-...",
  "outputs": {
    "instance_ip": {
      "value": "54.123.45.67",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-0abc123def456",
            "ami": "ami-12345678",
            "instance_type": "t2.micro",
            "public_ip": "54.123.45.67",
            "tags": {
              "Name": "web-server"
            }
          }
        }
      ]
    }
  ]
}
```

### Key Fields

| Field | Description |
|-------|-------------|
| `version` | State file format version |
| `terraform_version` | Terraform version used |
| `serial` | Incremented on each state change |
| `lineage` | Unique ID for state history |
| `outputs` | Output values |
| `resources` | Managed resources |

---

## Local State

### Default Behavior

```bash
# State stored locally by default
$ ls -la
terraform.tfstate       # Current state
terraform.tfstate.backup # Previous state
```

### Limitations of Local State

| Issue | Description |
|-------|-------------|
| **No Sharing** | Can't collaborate with team |
| **No Locking** | Risk of concurrent modifications |
| **Data Loss** | Local file can be deleted/corrupted |
| **Secrets Exposed** | Sensitive data in plain text |

---

## Remote State

### Benefits

| Benefit | Description |
|---------|-------------|
| **Collaboration** | Team shares single state |
| **Locking** | Prevents concurrent changes |
| **Security** | Encrypted storage, access control |
| **Versioning** | State history and recovery |

### Common Backends

| Backend | Provider | Locking |
|---------|----------|---------|
| `s3` | AWS | DynamoDB |
| `azurerm` | Azure | Built-in |
| `gcs` | Google Cloud | Built-in |
| `consul` | HashiCorp | Built-in |
| `http` | Any | Depends |
| `remote` | Terraform Cloud | Built-in |

### S3 Backend Example

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # For locking
  }
}
```

---

## State Locking

### Why Locking Matters

```
┌─────────────────────────────────────────────────────────────┐
│                WITHOUT STATE LOCKING                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User A                              User B                 │
│  ┌─────────────┐                    ┌─────────────┐        │
│  │ Read state  │                    │ Read state  │        │
│  └──────┬──────┘                    └──────┬──────┘        │
│         │                                  │                │
│         ▼                                  ▼                │
│  ┌─────────────┐                    ┌─────────────┐        │
│  │ Plan change │                    │ Plan change │        │
│  │ (add VM)    │                    │ (modify VM) │        │
│  └──────┬──────┘                    └──────┬──────┘        │
│         │                                  │                │
│         ▼                                  ▼                │
│  ┌─────────────┐                    ┌─────────────┐        │
│  │ Write state │                    │ Write state │        │
│  └─────────────┘                    └─────────────┘        │
│                                                             │
│  CONFLICT! One user's changes may be overwritten!          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### With Locking

```
┌─────────────────────────────────────────────────────────────┐
│                  WITH STATE LOCKING                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User A                              User B                 │
│  ┌─────────────┐                    ┌─────────────┐        │
│  │ Acquire lock│                    │ Try to lock │        │
│  │ ✓ Success   │                    │ ✗ Blocked   │        │
│  └──────┬──────┘                    └──────┬──────┘        │
│         │                                  │                │
│         ▼                                  │ Wait...        │
│  ┌─────────────┐                           │                │
│  │ Make changes│                           │                │
│  └──────┬──────┘                           │                │
│         │                                  │                │
│         ▼                                  │                │
│  ┌─────────────┐                           │                │
│  │ Release lock│                           │                │
│  └──────┬──────┘                           │                │
│         │                                  ▼                │
│         │                           ┌─────────────┐        │
│         │                           │ Acquire lock│        │
│         │                           │ ✓ Success   │        │
│         │                           └─────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### DynamoDB for S3 Locking

```hcl
# Create DynamoDB table for locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Terraform State Lock Table"
  }
}

# Use in backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```

### Force Unlock

```bash
# If lock is stuck (use with caution!)
terraform force-unlock LOCK_ID

# Example
terraform force-unlock abc123-def456-789...
```

---

## State Commands

### terraform state list

```bash
# List all resources in state
terraform state list

# Output:
# aws_instance.web
# aws_security_group.web_sg
# aws_vpc.main

# Filter by resource type
terraform state list aws_instance.*

# Filter by module
terraform state list module.vpc.*
```

### terraform state show

```bash
# Show details of a resource
terraform state show aws_instance.web

# Output:
# resource "aws_instance" "web" {
#     id            = "i-0abc123def456"
#     ami           = "ami-12345678"
#     instance_type = "t2.micro"
#     ...
# }
```

### terraform state mv

```bash
# Rename a resource
terraform state mv aws_instance.web aws_instance.webserver

# Move to a module
terraform state mv aws_instance.web module.compute.aws_instance.web

# Move between modules
terraform state mv module.old.aws_instance.web module.new.aws_instance.web
```

### terraform state rm

```bash
# Remove from state (doesn't destroy resource!)
terraform state rm aws_instance.web

# Resource will no longer be managed by Terraform
# You'll need to import it or delete manually
```

### terraform state pull

```bash
# Download remote state to stdout
terraform state pull > backup.tfstate

# Useful for backup or inspection
terraform state pull | jq '.resources'
```

### terraform state push

```bash
# Upload local state to remote (dangerous!)
terraform state push backup.tfstate

# Use with extreme caution
# Can overwrite remote state
```

---

## State Refresh

### terraform refresh (deprecated)

```bash
# Sync state with real infrastructure
terraform refresh  # Deprecated in favor of:

# Use apply with refresh-only
terraform apply -refresh-only
```

### Refresh During Plan/Apply

```bash
# Disable refresh
terraform plan -refresh=false
terraform apply -refresh=false

# Normal behavior (refresh enabled)
terraform plan  # Refreshes state automatically
```

---

## Sensitive Data in State

### What's Stored

State files contain sensitive information:
- Database passwords
- API keys
- Private keys
- Connection strings

### Protection Strategies

```hcl
# 1. Enable encryption (S3 backend)
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "terraform.tfstate"
    region  = "us-east-1"
    encrypt = true  # Server-side encryption

    # Optional: Use KMS key
    kms_key_id = "arn:aws:kms:us-east-1:123456789:key/abc-123"
  }
}

# 2. Restrict access (IAM policy)
# Limit who can read state bucket

# 3. Mark sensitive outputs
output "db_password" {
  value     = random_password.db.result
  sensitive = true
}

# 4. Use secrets manager for sensitive values
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "database-password"
}
```

---

## State File Best Practices

### 1. Always Use Remote State

```hcl
terraform {
  backend "s3" {
    bucket = "company-terraform-state"
    key    = "${var.environment}/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### 2. Enable Locking

```hcl
terraform {
  backend "s3" {
    # ... other config
    dynamodb_table = "terraform-locks"
  }
}
```

### 3. Enable Versioning

```hcl
# Enable S3 versioning for state bucket
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### 4. Never Edit State Manually

```bash
# Use state commands instead
terraform state mv ...
terraform state rm ...

# Don't:
# vim terraform.tfstate  # NEVER!
```

### 5. Backup Before Dangerous Operations

```bash
# Before state manipulation
terraform state pull > backup-$(date +%Y%m%d).tfstate
```

---

## Summary

| Command | Purpose |
|---------|---------|
| `terraform state list` | List resources |
| `terraform state show` | Show resource details |
| `terraform state mv` | Rename/move resource |
| `terraform state rm` | Remove from state |
| `terraform state pull` | Download state |
| `terraform state push` | Upload state |
| `terraform force-unlock` | Release stuck lock |

### State Best Practices

1. **Use remote state** for collaboration
2. **Enable locking** to prevent conflicts
3. **Encrypt state** at rest
4. **Version state** bucket for recovery
5. **Restrict access** with IAM
6. **Never edit** state manually
7. **Backup** before dangerous operations
