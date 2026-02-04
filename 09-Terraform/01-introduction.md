# Introduction to Terraform

## What is Terraform?

Terraform is an open-source Infrastructure as Code (IaC) tool created by HashiCorp that allows you to define, provision, and manage infrastructure using a declarative configuration language.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM WORKFLOW                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌────────┐ │
│   │  Write  │───►│  Plan   │───►│  Apply  │───►│Infrastructure│
│   │  (.tf)  │    │         │    │         │    │              │
│   └─────────┘    └─────────┘    └─────────┘    └────────┘ │
│                                                             │
│   Define         Preview        Execute       Resources     │
│   resources      changes        changes       created       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Declarative** | Define what you want, not how to create it |
| **Cloud Agnostic** | Works with AWS, Azure, GCP, and 3000+ providers |
| **State Management** | Tracks infrastructure state for updates |
| **Idempotent** | Running multiple times produces same result |
| **Open Source** | Free to use, large community |

---

## Why Terraform?

### 1. Multi-Cloud Support

```hcl
# AWS
provider "aws" {
  region = "us-east-1"
}

# Azure
provider "azurerm" {
  features {}
}

# Google Cloud
provider "google" {
  project = "my-project"
  region  = "us-central1"
}
```

### 2. Declarative Syntax

```hcl
# You define WHAT you want
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "WebServer"
  }
}
# Terraform figures out HOW to create it
```

### 3. State Management

```
┌─────────────────────────────────────────┐
│            terraform.tfstate            │
├─────────────────────────────────────────┤
│                                         │
│  Tracks:                                │
│  • Resource IDs                         │
│  • Current configuration                │
│  • Dependencies                         │
│  • Metadata                             │
│                                         │
│  Enables:                               │
│  • Change detection                     │
│  • Resource updates                     │
│  • Dependency ordering                  │
│                                         │
└─────────────────────────────────────────┘
```

### 4. Plan Before Apply

```bash
$ terraform plan

Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t2.micro"
      + tags          = {
          + "Name" = "WebServer"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### 5. Infrastructure as Code Benefits

| Benefit | Description |
|---------|-------------|
| **Version Control** | Track changes in Git |
| **Code Review** | Review infrastructure changes like code |
| **Reproducibility** | Same config = same infrastructure |
| **Documentation** | Code serves as documentation |
| **Automation** | CI/CD pipeline integration |

---

## Key Concepts

### Resources

The most important element - represents infrastructure objects.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "main-vpc"
  }
}
```

### Providers

Plugins that interact with cloud platforms and services.

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

### State

Terraform's record of managed infrastructure.

```
Configuration (.tf) ◄──► State File ◄──► Real Infrastructure
```

### Modules

Reusable, shareable packages of Terraform configurations.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

### Variables

Input parameters for configurations.

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

### Outputs

Export values from your configuration.

```hcl
output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.web.public_ip
}
```

---

## Terraform vs Other Tools

### Terraform vs CloudFormation

| Feature | Terraform | CloudFormation |
|---------|-----------|----------------|
| **Cloud Support** | Multi-cloud | AWS only |
| **Language** | HCL | YAML/JSON |
| **State** | Self-managed | AWS-managed |
| **Modularity** | Modules | Nested stacks |
| **Learning Curve** | Moderate | Moderate |
| **Drift Detection** | Manual refresh | Built-in |

### Terraform vs Ansible

| Feature | Terraform | Ansible |
|---------|-----------|---------|
| **Primary Use** | Infrastructure provisioning | Configuration management |
| **Approach** | Declarative | Procedural (mostly) |
| **State** | Yes | No |
| **Idempotency** | Built-in | Task-dependent |
| **Agent** | Agentless | Agentless |

### Terraform vs Pulumi

| Feature | Terraform | Pulumi |
|---------|-----------|--------|
| **Language** | HCL | Python, TypeScript, Go, etc. |
| **State** | File-based | Pulumi Cloud or self-managed |
| **Learning Curve** | New language (HCL) | Use existing language |
| **Testing** | Limited | Full programming capabilities |
| **Community** | Larger | Growing |

### When to Choose Terraform

- Multi-cloud or hybrid cloud environments
- Team familiar with declarative approaches
- Need for extensive provider ecosystem
- Want infrastructure versioned in Git
- Prefer separation of infrastructure and application code

---

## Use Cases

### 1. Cloud Infrastructure Provisioning

```hcl
# Complete AWS infrastructure
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "public" { ... }
resource "aws_instance" "web" { ... }
resource "aws_lb" "app" { ... }
resource "aws_rds_cluster" "db" { ... }
```

### 2. Multi-Cloud Deployments

```hcl
# Deploy to multiple clouds
provider "aws" { ... }
provider "azurerm" { ... }

resource "aws_instance" "web_aws" { ... }
resource "azurerm_virtual_machine" "web_azure" { ... }
```

### 3. Kubernetes Infrastructure

```hcl
# Provision Kubernetes cluster
resource "aws_eks_cluster" "main" { ... }

# Deploy to Kubernetes
provider "kubernetes" {
  host = aws_eks_cluster.main.endpoint
}

resource "kubernetes_deployment" "app" { ... }
```

### 4. Network Infrastructure

```hcl
# DNS, CDN, Load Balancers
resource "aws_route53_zone" "main" { ... }
resource "aws_cloudfront_distribution" "cdn" { ... }
resource "aws_lb" "app" { ... }
```

### 5. GitOps Workflows

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   Git    │───►│  CI/CD   │───►│ Terraform│───►│  Cloud   │
│  Commit  │    │ Pipeline │    │  Apply   │    │ Resources│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

## Terraform Editions

| Edition | Description | Use Case |
|---------|-------------|----------|
| **Terraform CLI** | Open-source CLI tool | Individual use, small teams |
| **Terraform Cloud** | SaaS platform | Teams, collaboration |
| **Terraform Enterprise** | Self-hosted platform | Large enterprises |

### Terraform Cloud Features

- Remote state storage
- Team collaboration
- Private module registry
- Policy as code (Sentinel)
- Cost estimation
- VCS integration

---

## Getting Started Checklist

- [ ] Install Terraform CLI
- [ ] Choose a cloud provider
- [ ] Set up provider credentials
- [ ] Write your first configuration
- [ ] Run `terraform init`
- [ ] Run `terraform plan`
- [ ] Run `terraform apply`
- [ ] Explore the state file
- [ ] Run `terraform destroy`

---

## Summary

| Concept | Description |
|---------|-------------|
| **Terraform** | Infrastructure as Code tool by HashiCorp |
| **HCL** | HashiCorp Configuration Language |
| **Provider** | Plugin for cloud/service interaction |
| **Resource** | Infrastructure object to manage |
| **State** | Record of managed infrastructure |
| **Module** | Reusable configuration package |
| **Plan** | Preview of changes |
| **Apply** | Execute changes |

Terraform enables teams to define infrastructure in code, version it, review it, and automate its deployment across any cloud or service provider.
