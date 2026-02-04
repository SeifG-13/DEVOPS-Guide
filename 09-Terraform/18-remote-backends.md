# Terraform Remote Backends

## What are Backends?

Backends determine where Terraform stores its state data. Remote backends enable team collaboration and provide features like locking and encryption.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM BACKENDS                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Local Backend (Default)        Remote Backend              │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │ terraform.tfstate│           │   S3 / Azure /  │         │
│  │ (local file)    │           │   GCS / Consul  │         │
│  └─────────────────┘           └─────────────────┘         │
│                                                             │
│  Limitations:                   Benefits:                   │
│  • No team sharing             • Team collaboration         │
│  • No locking                  • State locking              │
│  • No encryption               • Encryption at rest         │
│  • Risk of data loss           • Versioning/backup          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Backend Types

| Backend | Provider | Locking | Description |
|---------|----------|---------|-------------|
| `local` | - | No | Default, stores locally |
| `s3` | AWS | DynamoDB | S3 bucket with optional locking |
| `azurerm` | Azure | Built-in | Azure Blob Storage |
| `gcs` | GCP | Built-in | Google Cloud Storage |
| `consul` | HashiCorp | Built-in | Consul KV store |
| `http` | Any | Optional | HTTP endpoints |
| `remote` | TFC | Built-in | Terraform Cloud |
| `kubernetes` | K8s | Built-in | Kubernetes secrets |
| `pg` | PostgreSQL | Built-in | PostgreSQL database |

---

## S3 Backend (AWS)

### Basic Configuration

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### With DynamoDB Locking

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

### Complete S3 Backend Setup

```hcl
# 1. Create S3 bucket for state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-${data.aws_caller_identity.current.account_id}"

  lifecycle {
    prevent_destroy = true
  }
}

# 2. Enable versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# 3. Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

# 4. Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# 5. Create DynamoDB table for locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  lifecycle {
    prevent_destroy = true
  }
}

# 6. KMS key for encryption
resource "aws_kms_key" "terraform_state" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```

### S3 Backend Options

```hcl
terraform {
  backend "s3" {
    # Required
    bucket = "my-terraform-state"
    key    = "path/to/terraform.tfstate"
    region = "us-east-1"

    # Encryption
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/abc-123"

    # Locking
    dynamodb_table = "terraform-locks"

    # Authentication
    profile        = "terraform"  # AWS profile
    role_arn       = "arn:aws:iam::123456789:role/TerraformRole"

    # Workspace prefix
    workspace_key_prefix = "workspaces"
  }
}
```

---

## Azure Backend (azurerm)

### Basic Configuration

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### Setup Azure Backend Resources

```hcl
# Create resource group
resource "azurerm_resource_group" "terraform_state" {
  name     = "terraform-state-rg"
  location = "East US"
}

# Create storage account
resource "azurerm_storage_account" "terraform_state" {
  name                     = "tfstate${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.terraform_state.name
  location                 = azurerm_resource_group.terraform_state.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  blob_properties {
    versioning_enabled = true
  }
}

# Create container
resource "azurerm_storage_container" "terraform_state" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.terraform_state.name
  container_access_type = "private"
}
```

### Azure Backend Options

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"

    # Authentication options
    use_azuread_auth     = true  # Use Azure AD
    subscription_id      = "xxx-xxx-xxx"
    tenant_id            = "xxx-xxx-xxx"
    client_id            = "xxx-xxx-xxx"
    client_secret        = "xxx"  # Or use env var ARM_CLIENT_SECRET
  }
}
```

---

## GCS Backend (Google Cloud)

### Basic Configuration

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "terraform/state"
  }
}
```

### Setup GCS Backend

```hcl
# Create bucket
resource "google_storage_bucket" "terraform_state" {
  name          = "my-terraform-state-${var.project_id}"
  location      = "US"
  force_destroy = false

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      num_newer_versions = 5
    }
    action {
      type = "Delete"
    }
  }

  uniform_bucket_level_access = true
}
```

### GCS Backend Options

```hcl
terraform {
  backend "gcs" {
    bucket      = "my-terraform-state"
    prefix      = "prod"
    credentials = "service-account.json"  # Or use GOOGLE_CREDENTIALS

    # Encryption
    encryption_key = "base64-encoded-key"  # Customer-supplied
  }
}
```

---

## Consul Backend

```hcl
terraform {
  backend "consul" {
    address = "consul.example.com:8500"
    scheme  = "https"
    path    = "terraform/state"

    # Authentication
    access_token = var.consul_token  # Or CONSUL_HTTP_TOKEN env var
  }
}
```

---

## HTTP Backend

```hcl
terraform {
  backend "http" {
    address        = "https://mybackend.example.com/state"
    lock_address   = "https://mybackend.example.com/lock"
    unlock_address = "https://mybackend.example.com/unlock"

    username       = "terraform"
    password       = var.backend_password
  }
}
```

---

## PostgreSQL Backend

```hcl
terraform {
  backend "pg" {
    conn_str    = "postgres://user:pass@db.example.com/terraform_state"
    schema_name = "terraform_remote_state"
  }
}
```

---

## Backend Configuration

### Partial Configuration

Store sensitive values outside the configuration.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "terraform.tfstate"
    region = "us-east-1"
    # Other values provided at init
  }
}
```

```bash
# Provide at init time
terraform init \
  -backend-config="dynamodb_table=terraform-locks" \
  -backend-config="encrypt=true"

# Or use a file
terraform init -backend-config=backend.hcl
```

```hcl
# backend.hcl
dynamodb_table = "terraform-locks"
encrypt        = true
role_arn       = "arn:aws:iam::123456789:role/TerraformRole"
```

---

## Backend Migration

### Migrate Local to Remote

```bash
# 1. Add backend configuration
# backend.tf
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}

# 2. Initialize with migration
terraform init -migrate-state

# Terraform will ask:
# Do you want to copy existing state to the new backend?
# Enter "yes"
```

### Migrate Between Backends

```bash
# 1. Update backend configuration
# Change from s3 to azurerm, for example

# 2. Reinitialize
terraform init -migrate-state

# 3. Verify
terraform state list
```

### Reconfigure Backend

```bash
# Reinitialize without migrating state
terraform init -reconfigure
```

---

## State Encryption

### S3 with KMS

```hcl
terraform {
  backend "s3" {
    bucket     = "my-terraform-state"
    key        = "terraform.tfstate"
    region     = "us-east-1"
    encrypt    = true
    kms_key_id = "arn:aws:kms:us-east-1:123456789:key/abc-123"
  }
}
```

### Azure with Customer-Managed Key

```hcl
# Storage account with CMK
resource "azurerm_storage_account" "terraform_state" {
  name                     = "tfstateaccount"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  identity {
    type = "SystemAssigned"
  }

  customer_managed_key {
    key_vault_key_id          = azurerm_key_vault_key.terraform.id
    user_assigned_identity_id = azurerm_user_assigned_identity.terraform.id
  }
}
```

---

## State Locking

### How Locking Works

```
┌─────────────────────────────────────────────────────────────┐
│                    STATE LOCKING                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User A: terraform apply                                    │
│      │                                                      │
│      ▼                                                      │
│  ┌─────────────────┐                                       │
│  │ Acquire Lock    │  ✓ Success                            │
│  └─────────────────┘                                       │
│      │                                                      │
│      │    User B: terraform apply                          │
│      │        │                                            │
│      │        ▼                                            │
│      │    ┌─────────────────┐                              │
│      │    │ Acquire Lock    │  ✗ Failed (locked)          │
│      │    └─────────────────┘                              │
│      │        │                                            │
│      │        ▼                                            │
│      │    Error: state locked                              │
│      │                                                      │
│      ▼                                                      │
│  ┌─────────────────┐                                       │
│  │ Make changes    │                                       │
│  └─────────────────┘                                       │
│      │                                                      │
│      ▼                                                      │
│  ┌─────────────────┐                                       │
│  │ Release Lock    │                                       │
│  └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Force Unlock

```bash
# If lock is stuck
terraform force-unlock LOCK_ID

# Get lock ID from error message:
# Error: Error acquiring the state lock
# Lock Info:
#   ID:        abc123-def456-...
#   Path:      terraform.tfstate
#   Operation: OperationTypePlan
```

### Disable Locking (Not Recommended)

```bash
# Skip locking for this operation
terraform apply -lock=false

# Skip lock timeout
terraform apply -lock-timeout=5m
```

---

## Workspaces with Remote Backends

### S3 Workspace Paths

```hcl
terraform {
  backend "s3" {
    bucket               = "my-terraform-state"
    key                  = "terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "workspaces"
  }
}

# State file paths:
# default:    s3://my-terraform-state/terraform.tfstate
# dev:        s3://my-terraform-state/workspaces/dev/terraform.tfstate
# production: s3://my-terraform-state/workspaces/production/terraform.tfstate
```

---

## Best Practices

### 1. Always Use Remote State in Teams

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "project/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### 2. Enable Versioning

```hcl
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### 3. Encrypt State

```hcl
backend "s3" {
  encrypt    = true
  kms_key_id = aws_kms_key.terraform.arn
}
```

### 4. Restrict Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-terraform-state/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/Team": "DevOps"
        }
      }
    }
  ]
}
```

### 5. Use Separate State per Environment

```
s3://terraform-state/
├── dev/terraform.tfstate
├── staging/terraform.tfstate
└── production/terraform.tfstate
```

---

## Summary

| Backend | Locking | Encryption | Best For |
|---------|---------|------------|----------|
| `s3` | DynamoDB | KMS | AWS teams |
| `azurerm` | Built-in | CMK | Azure teams |
| `gcs` | Built-in | CMEK | GCP teams |
| `remote` | Built-in | Built-in | Terraform Cloud |
| `consul` | Built-in | TLS | Multi-cloud |

### Backend Configuration Pattern

```hcl
terraform {
  backend "<type>" {
    # Storage location
    # Encryption settings
    # Locking configuration
    # Authentication
  }
}
```
