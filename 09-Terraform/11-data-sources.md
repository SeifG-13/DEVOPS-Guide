# Terraform Data Sources

## What are Data Sources?

Data sources allow Terraform to query and fetch information from providers without creating or managing resources. They read existing infrastructure or external data.

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA SOURCES                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Resources (Manage)              Data Sources (Query)       │
│  ┌─────────────────┐            ┌─────────────────┐        │
│  │ resource "aws_  │            │ data "aws_ami"  │        │
│  │ instance" "web" │            │ "ubuntu" {      │        │
│  │ {               │            │   most_recent   │        │
│  │   ami = ...     │            │   = true        │        │
│  │ }               │            │ }               │        │
│  └────────┬────────┘            └────────┬────────┘        │
│           │                              │                  │
│           ▼                              ▼                  │
│  ┌─────────────────┐            ┌─────────────────┐        │
│  │ CREATE/UPDATE/  │            │ READ ONLY       │        │
│  │ DELETE          │            │ (Query API)     │        │
│  └─────────────────┘            └─────────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Source Syntax

### Basic Structure

```hcl
data "provider_type" "name" {
  # Query parameters
  filter_argument = "value"
}

# Reference
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}
```

### Example

```hcl
# Find latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use the AMI
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}
```

---

## Common AWS Data Sources

### aws_ami

```hcl
# Latest Amazon Linux 2
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Specific AMI by ID
data "aws_ami" "specific" {
  filter {
    name   = "image-id"
    values = ["ami-12345678"]
  }
}
```

### aws_vpc

```hcl
# Find VPC by tag
data "aws_vpc" "main" {
  tags = {
    Name = "main-vpc"
  }
}

# Find default VPC
data "aws_vpc" "default" {
  default = true
}

# Use VPC data
resource "aws_subnet" "public" {
  vpc_id     = data.aws_vpc.main.id
  cidr_block = cidrsubnet(data.aws_vpc.main.cidr_block, 8, 1)
}
```

### aws_subnets

```hcl
# Find all subnets in VPC
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  tags = {
    Tier = "private"
  }
}

# Use subnet IDs
resource "aws_instance" "web" {
  count     = length(data.aws_subnets.private.ids)
  subnet_id = data.aws_subnets.private.ids[count.index]
  # ...
}
```

### aws_availability_zones

```hcl
# Get available AZs
data "aws_availability_zones" "available" {
  state = "available"

  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

# Use in resources
resource "aws_subnet" "public" {
  count             = length(data.aws_availability_zones.available.names)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### aws_caller_identity

```hcl
# Get current AWS account info
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "caller_arn" {
  value = data.aws_caller_identity.current.arn
}
```

### aws_region

```hcl
# Get current region
data "aws_region" "current" {}

output "region_name" {
  value = data.aws_region.current.name
}
```

### aws_iam_policy_document

```hcl
# Build IAM policy as data source
data "aws_iam_policy_document" "s3_read" {
  statement {
    sid    = "AllowS3Read"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]

    resources = [
      aws_s3_bucket.data.arn,
      "${aws_s3_bucket.data.arn}/*"
    ]
  }

  statement {
    sid    = "DenyDelete"
    effect = "Deny"

    actions = [
      "s3:DeleteObject"
    ]

    resources = ["*"]
  }
}

# Use the policy document
resource "aws_iam_policy" "s3_read" {
  name   = "s3-read-policy"
  policy = data.aws_iam_policy_document.s3_read.json
}
```

### aws_secretsmanager_secret_version

```hcl
# Fetch secret from Secrets Manager
data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = "prod/db/credentials"
}

# Parse JSON secret
locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)
}

resource "aws_db_instance" "main" {
  username = local.db_creds.username
  password = local.db_creds.password
  # ...
}
```

---

## Common Azure Data Sources

### azurerm_resource_group

```hcl
data "azurerm_resource_group" "existing" {
  name = "my-resource-group"
}

resource "azurerm_virtual_network" "main" {
  name                = "main-vnet"
  resource_group_name = data.azurerm_resource_group.existing.name
  location            = data.azurerm_resource_group.existing.location
  address_space       = ["10.0.0.0/16"]
}
```

### azurerm_subscription

```hcl
data "azurerm_subscription" "current" {}

output "subscription_id" {
  value = data.azurerm_subscription.current.subscription_id
}
```

### azurerm_client_config

```hcl
data "azurerm_client_config" "current" {}

output "tenant_id" {
  value = data.azurerm_client_config.current.tenant_id
}

output "client_id" {
  value = data.azurerm_client_config.current.client_id
}
```

---

## Common GCP Data Sources

### google_compute_image

```hcl
data "google_compute_image" "debian" {
  family  = "debian-11"
  project = "debian-cloud"
}

resource "google_compute_instance" "vm" {
  name         = "my-vm"
  machine_type = "e2-medium"

  boot_disk {
    initialize_params {
      image = data.google_compute_image.debian.self_link
    }
  }
}
```

### google_project

```hcl
data "google_project" "current" {}

output "project_id" {
  value = data.google_project.current.project_id
}

output "project_number" {
  value = data.google_project.current.number
}
```

---

## Filtering Data

### Using filter Blocks

```hcl
data "aws_ami" "example" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["myapp-*"]
  }

  filter {
    name   = "tag:Environment"
    values = ["production"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}
```

### Using Tags

```hcl
data "aws_vpc" "main" {
  tags = {
    Name        = "main-vpc"
    Environment = "production"
  }
}

data "aws_instances" "web_servers" {
  instance_tags = {
    Role = "web"
  }

  instance_state_names = ["running"]
}
```

---

## External Data Source

Run external scripts to fetch data.

```hcl
# Fetch data from external script
data "external" "git_info" {
  program = ["bash", "${path.module}/scripts/git-info.sh"]

  query = {
    working_dir = path.root
  }
}

# scripts/git-info.sh
#!/bin/bash
# Must output valid JSON
cat <<EOF
{
  "branch": "$(git rev-parse --abbrev-ref HEAD)",
  "commit": "$(git rev-parse --short HEAD)"
}
EOF

# Use in resources
resource "aws_instance" "web" {
  # ...
  tags = {
    GitBranch = data.external.git_info.result.branch
    GitCommit = data.external.git_info.result.commit
  }
}
```

---

## HTTP Data Source

Fetch data from HTTP endpoints.

```hcl
data "http" "my_ip" {
  url = "https://ipv4.icanhazip.com"
}

resource "aws_security_group_rule" "allow_my_ip" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["${chomp(data.http.my_ip.response_body)}/32"]
  security_group_id = aws_security_group.main.id
}
```

---

## Template File Data Source

```hcl
data "template_file" "user_data" {
  template = file("${path.module}/templates/user_data.sh")

  vars = {
    hostname = var.hostname
    env      = var.environment
  }
}

# Better: Use templatefile function
resource "aws_instance" "web" {
  user_data = templatefile("${path.module}/templates/user_data.sh", {
    hostname = var.hostname
    env      = var.environment
  })
}
```

---

## Remote State Data Source

Access outputs from other Terraform configurations.

```hcl
# Read outputs from another state
data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = "terraform-state-bucket"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use outputs
resource "aws_instance" "web" {
  subnet_id         = data.terraform_remote_state.network.outputs.public_subnet_id
  security_groups   = data.terraform_remote_state.network.outputs.security_group_ids
  availability_zone = data.terraform_remote_state.network.outputs.availability_zone
}
```

---

## Data Sources vs Resources

| Aspect | Data Source | Resource |
|--------|-------------|----------|
| **Purpose** | Query existing | Create/manage |
| **Action** | Read-only | Create, update, delete |
| **State** | Refreshed each run | Tracked in state |
| **Lifecycle** | No lifecycle | Has lifecycle |
| **Keyword** | `data` | `resource` |

### When to Use Data Sources

1. **Reference existing resources** not managed by Terraform
2. **Query dynamic values** (latest AMI, current IP)
3. **Cross-stack references** (remote state)
4. **Build policies** (IAM policy documents)
5. **Fetch secrets** from secret managers

---

## Best Practices

### 1. Use for Truly External Resources

```hcl
# Good - VPC exists outside this config
data "aws_vpc" "existing" {
  id = "vpc-12345678"
}

# Avoid - You're managing this VPC
# Use resource instead
resource "aws_vpc" "managed" {
  cidr_block = "10.0.0.0/16"
}
```

### 2. Handle Missing Data

```hcl
# Add count to make optional
data "aws_ami" "custom" {
  count = var.use_custom_ami ? 1 : 0

  filter {
    name   = "name"
    values = [var.custom_ami_name]
  }
}

# Use with conditional
resource "aws_instance" "web" {
  ami = var.use_custom_ami ? data.aws_ami.custom[0].id : data.aws_ami.ubuntu.id
}
```

### 3. Document Dependencies

```hcl
# Clearly document what this data source expects
data "aws_vpc" "main" {
  # Expects a VPC with this tag to exist
  # Created by the network team's Terraform
  tags = {
    Name = "main-vpc"
  }
}
```

---

## Summary

| Data Source Type | Use Case |
|------------------|----------|
| `aws_ami` | Find AMI images |
| `aws_vpc` | Reference existing VPC |
| `aws_subnets` | Find subnets |
| `aws_availability_zones` | Get available AZs |
| `aws_caller_identity` | Get account info |
| `aws_iam_policy_document` | Build IAM policies |
| `aws_secretsmanager_secret` | Fetch secrets |
| `terraform_remote_state` | Cross-config references |
| `external` | Run external scripts |
| `http` | Fetch HTTP data |

### Reference Syntax

```hcl
# Data source reference
data.<TYPE>.<NAME>.<ATTRIBUTE>

# Examples
data.aws_ami.ubuntu.id
data.aws_vpc.main.cidr_block
data.aws_caller_identity.current.account_id
```
