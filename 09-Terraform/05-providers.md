# Terraform Providers

## What are Providers?

Providers are plugins that enable Terraform to interact with cloud platforms, SaaS providers, and other APIs. Each provider adds a set of resource types and data sources.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM PROVIDERS                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Terraform Core                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    terraform                         │   │
│  │                                                      │   │
│  │  Configuration ──► Provider Plugins ──► APIs        │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│           ┌───────────────┼───────────────┐                │
│           ▼               ▼               ▼                │
│     ┌──────────┐   ┌──────────┐   ┌──────────┐           │
│     │   AWS    │   │  Azure   │   │   GCP    │           │
│     │ Provider │   │ Provider │   │ Provider │           │
│     └────┬─────┘   └────┬─────┘   └────┬─────┘           │
│          │              │              │                   │
│          ▼              ▼              ▼                   │
│     ┌──────────┐   ┌──────────┐   ┌──────────┐           │
│     │  AWS API │   │Azure API │   │ GCP API  │           │
│     └──────────┘   └──────────┘   └──────────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Provider Configuration

### Basic Syntax

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Provider Source Address

```hcl
# Format: [hostname/]namespace/type
# Default hostname: registry.terraform.io

required_providers {
  # HashiCorp provider
  aws = {
    source = "hashicorp/aws"  # registry.terraform.io/hashicorp/aws
  }

  # Third-party provider
  datadog = {
    source = "DataDog/datadog"
  }

  # Private registry
  internal = {
    source = "my-registry.example.com/myorg/internal"
  }
}
```

---

## Version Constraints

### Version Operators

```hcl
required_providers {
  aws = {
    source  = "hashicorp/aws"

    # Exact version
    version = "5.0.0"

    # Minimum version
    version = ">= 5.0.0"

    # Maximum version
    version = "< 6.0.0"

    # Range
    version = ">= 5.0.0, < 6.0.0"

    # Pessimistic constraint (allows 5.0.x but not 5.1.0)
    version = "~> 5.0.0"

    # Pessimistic (allows 5.x.x but not 6.0.0)
    version = "~> 5.0"
  }
}
```

### Version Constraint Examples

| Constraint | Meaning |
|------------|---------|
| `= 5.0.0` | Exactly 5.0.0 |
| `>= 5.0.0` | 5.0.0 or higher |
| `<= 5.0.0` | 5.0.0 or lower |
| `~> 5.0.0` | >= 5.0.0 and < 5.1.0 |
| `~> 5.0` | >= 5.0.0 and < 6.0.0 |
| `>= 5.0, < 6.0` | Range: 5.x.x versions |

---

## Major Cloud Providers

### AWS Provider

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  # Authentication options:

  # Option 1: Environment variables (recommended)
  # AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY

  # Option 2: Shared credentials file
  shared_credentials_files = ["~/.aws/credentials"]
  profile                  = "default"

  # Option 3: Static credentials (not recommended)
  # access_key = "AKIAIOSFODNN7EXAMPLE"
  # secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

  # Option 4: IAM role (for EC2, ECS, Lambda)
  # Automatically uses instance/task role

  # Default tags for all resources
  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "Terraform"
    }
  }
}
```

### Azure Provider

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}

  # Authentication options:

  # Option 1: Azure CLI (az login)
  # Automatically uses logged-in session

  # Option 2: Service Principal with Client Secret
  subscription_id = "00000000-0000-0000-0000-000000000000"
  tenant_id       = "00000000-0000-0000-0000-000000000000"
  client_id       = "00000000-0000-0000-0000-000000000000"
  client_secret   = var.arm_client_secret

  # Option 3: Environment variables
  # ARM_SUBSCRIPTION_ID, ARM_TENANT_ID, ARM_CLIENT_ID, ARM_CLIENT_SECRET

  # Option 4: Managed Identity (for Azure VMs, App Service)
  # use_msi = true
}
```

### GCP Provider

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "my-project-id"
  region  = "us-central1"
  zone    = "us-central1-a"

  # Authentication options:

  # Option 1: Application Default Credentials
  # gcloud auth application-default login

  # Option 2: Service Account Key File
  credentials = file("service-account.json")

  # Option 3: Environment variable
  # GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json
}
```

---

## Multiple Provider Configurations

### Provider Aliases

```hcl
# Default provider (us-east-1)
provider "aws" {
  region = "us-east-1"
}

# Aliased provider (us-west-2)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Aliased provider (eu-west-1)
provider "aws" {
  alias  = "europe"
  region = "eu-west-1"
}

# Using default provider
resource "aws_instance" "east_server" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Using aliased provider
resource "aws_instance" "west_server" {
  provider      = aws.west
  ami           = "ami-87654321"
  instance_type = "t2.micro"
}

resource "aws_instance" "europe_server" {
  provider      = aws.europe
  ami           = "ami-11111111"
  instance_type = "t2.micro"
}
```

### Multi-Cloud Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
}

provider "google" {
  project = "my-project"
  region  = "us-central1"
}

# AWS resource
resource "aws_s3_bucket" "backup" {
  bucket = "my-backup-bucket"
}

# Azure resource
resource "azurerm_resource_group" "main" {
  name     = "my-resource-group"
  location = "East US"
}

# GCP resource
resource "google_storage_bucket" "archive" {
  name     = "my-archive-bucket"
  location = "US"
}
```

---

## Provider Authentication

### Environment Variables

```bash
# AWS
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_REGION="us-east-1"

# Azure
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="your-client-secret"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"

# GCP
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
export GOOGLE_PROJECT="my-project-id"
```

### Using Variables (Secure)

```hcl
# variables.tf
variable "aws_access_key" {
  description = "AWS access key"
  type        = string
  sensitive   = true
}

variable "aws_secret_key" {
  description = "AWS secret key"
  type        = string
  sensitive   = true
}

# providers.tf
provider "aws" {
  region     = "us-east-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}

# terraform.tfvars (add to .gitignore!)
aws_access_key = "AKIAIOSFODNN7EXAMPLE"
aws_secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

### Assume Role (AWS)

```hcl
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::ACCOUNT_ID:role/TerraformRole"
    session_name = "TerraformSession"
    external_id  = "my-external-id"
  }
}
```

---

## Provider Lock File

### .terraform.lock.hcl

Terraform automatically creates a lock file to ensure consistent provider versions.

```hcl
# .terraform.lock.hcl (auto-generated)
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=",
    "zh:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  ]
}
```

### Lock File Commands

```bash
# Update lock file for current platform
terraform init -upgrade

# Update lock file for multiple platforms
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64

# View provider versions
terraform providers
```

---

## Common Providers

### Infrastructure Providers

| Provider | Description |
|----------|-------------|
| `hashicorp/aws` | Amazon Web Services |
| `hashicorp/azurerm` | Microsoft Azure |
| `hashicorp/google` | Google Cloud Platform |
| `digitalocean/digitalocean` | DigitalOcean |
| `linode/linode` | Linode |
| `hetznercloud/hcloud` | Hetzner Cloud |

### Kubernetes & Container

| Provider | Description |
|----------|-------------|
| `hashicorp/kubernetes` | Kubernetes resources |
| `hashicorp/helm` | Helm chart deployment |
| `kreuzwerker/docker` | Docker containers |

### DNS & CDN

| Provider | Description |
|----------|-------------|
| `hashicorp/dns` | DNS records |
| `cloudflare/cloudflare` | Cloudflare |
| `fastly/fastly` | Fastly CDN |

### Monitoring & Logging

| Provider | Description |
|----------|-------------|
| `DataDog/datadog` | Datadog |
| `newrelic/newrelic` | New Relic |
| `grafana/grafana` | Grafana |
| `PagerDuty/pagerduty` | PagerDuty |

### Database

| Provider | Description |
|----------|-------------|
| `cyrilgdn/postgresql` | PostgreSQL |
| `petoju/mysql` | MySQL |
| `mongodb/mongodbatlas` | MongoDB Atlas |

### Version Control

| Provider | Description |
|----------|-------------|
| `integrations/github` | GitHub |
| `gitlabhq/gitlab` | GitLab |
| `hashicorp/tfe` | Terraform Cloud/Enterprise |

---

## Provider Configuration Best Practices

### Separate Provider File

```hcl
# providers.tf
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

### Version Pinning

```hcl
# Pin to specific minor version for stability
required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.31.0"  # Allows 5.31.x
  }
}

# Or pin exactly for reproducibility
required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "= 5.31.0"  # Exactly 5.31.0
  }
}
```

### Commit Lock File

```bash
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars

# DO commit the lock file
# .terraform.lock.hcl should be committed
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Provider** | Plugin for API interaction |
| **Source** | Registry address (namespace/type) |
| **Version** | Version constraint |
| **Alias** | Multiple configurations of same provider |
| **Lock File** | Ensures consistent versions |
| **Authentication** | Credentials for provider access |

### Provider Block Structure

```hcl
terraform {
  required_providers {
    <name> = {
      source  = "<hostname>/<namespace>/<type>"
      version = "<constraint>"
    }
  }
}

provider "<name>" {
  alias = "optional_alias"
  # Provider-specific configuration
}
```
