# Terraform Resources

## What are Resources?

Resources are the most important element in Terraform. Each resource block describes one or more infrastructure objects, such as virtual machines, networks, DNS records, or storage buckets.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM RESOURCES                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  resource "aws_instance" "web" {                           │
│      │          │        │                                  │
│      │          │        └── Resource Name (local)          │
│      │          └── Resource Type                           │
│      └── Block Type                                         │
│                                                             │
│    ami           = "ami-12345678"  ◄── Arguments           │
│    instance_type = "t2.micro"                              │
│  }                                                          │
│                                                             │
│            ▼ terraform apply ▼                              │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              AWS EC2 Instance                        │   │
│  │              (Real Infrastructure)                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Resource Syntax

### Basic Structure

```hcl
resource "PROVIDER_TYPE" "NAME" {
  # Required arguments
  required_argument = "value"

  # Optional arguments
  optional_argument = "value"

  # Nested blocks
  nested_block {
    setting = "value"
  }
}
```

### Example Resources

```hcl
# AWS EC2 Instance
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }
}

# AWS S3 Bucket
resource "aws_s3_bucket" "data" {
  bucket = "my-unique-bucket-name"

  tags = {
    Environment = "production"
  }
}

# Azure Resource Group
resource "azurerm_resource_group" "main" {
  name     = "my-resource-group"
  location = "East US"
}

# GCP Compute Instance
resource "google_compute_instance" "vm" {
  name         = "my-instance"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }
}
```

---

## Resource Arguments

### Required vs Optional

```hcl
resource "aws_instance" "example" {
  # REQUIRED - must be specified
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # OPTIONAL - have defaults or can be omitted
  associate_public_ip_address = true
  monitoring                  = false

  # Check documentation for required/optional status
  # terraform providers schema -json
}
```

### Argument Types

```hcl
resource "aws_instance" "example" {
  # String
  instance_type = "t2.micro"

  # Number
  cpu_core_count = 2

  # Boolean
  monitoring = true

  # List
  security_groups = ["sg-12345678", "sg-87654321"]

  # Map
  tags = {
    Name        = "web-server"
    Environment = "production"
  }

  # Nested Block
  ebs_block_device {
    device_name = "/dev/sda1"
    volume_size = 100
    volume_type = "gp3"
  }
}
```

---

## Resource Attributes

After creation, resources expose attributes that can be referenced.

### Referencing Attributes

```hcl
# Create a VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Reference VPC ID in subnet
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # Reference attribute
  cidr_block = "10.0.1.0/24"
}

# Reference subnet ID in instance
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id  # Reference attribute
}
```

### Common Attribute Patterns

```hcl
# Resource ID
aws_instance.web.id

# ARN (AWS)
aws_instance.web.arn

# Name
aws_s3_bucket.data.bucket

# IP addresses
aws_instance.web.public_ip
aws_instance.web.private_ip

# DNS names
aws_lb.main.dns_name
aws_rds_instance.db.endpoint

# List of IDs
aws_subnet.public[*].id
```

---

## Resource Dependencies

### Implicit Dependencies

Terraform automatically determines dependencies from references.

```hcl
# VPC must be created first (no explicit depends_on needed)
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Terraform knows to create VPC before subnet
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # Implicit dependency
  cidr_block = "10.0.1.0/24"
}

# Terraform knows to create subnet before instance
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id  # Implicit dependency
}
```

### Explicit Dependencies

Use `depends_on` when there's no direct reference but order matters.

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_s3_bucket_policy" "data_policy" {
  bucket = aws_s3_bucket.data.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [...]
  })
}

# IAM role might need bucket policy to exist first
# even though there's no direct reference
resource "aws_iam_role" "app" {
  name = "app-role"

  assume_role_policy = jsonencode({...})

  # Explicit dependency
  depends_on = [aws_s3_bucket_policy.data_policy]
}
```

### Dependency Graph

```bash
# View dependency graph
terraform graph | dot -Tpng > graph.png

# View graph in text
terraform graph
```

---

## Resource Behavior

### Create, Update, Destroy

```
┌─────────────────────────────────────────────────────────────┐
│                 RESOURCE LIFECYCLE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  New Resource           Existing Resource                   │
│  ┌─────────┐           ┌─────────────────┐                 │
│  │ CREATE  │           │ Config changed? │                 │
│  └────┬────┘           └────────┬────────┘                 │
│       │                         │                           │
│       ▼                    Yes  │  No                       │
│  ┌─────────┐           ┌───────┴───────┐                   │
│  │ Resource│           │               │                   │
│  │ Created │           ▼               ▼                   │
│  └─────────┘     ┌──────────┐   ┌──────────┐              │
│                  │  UPDATE  │   │ No Change│              │
│                  │ in-place │   └──────────┘              │
│                  └────┬─────┘                              │
│                       │                                     │
│            Can update │ Must replace                        │
│            in-place?  │ (forces new)                       │
│                  ┌────┴─────┐                              │
│                  │          │                               │
│                  ▼          ▼                               │
│            ┌─────────┐ ┌─────────┐                         │
│            │ Updated │ │ Destroy │                         │
│            └─────────┘ │ + Create│                         │
│                        └─────────┘                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Understanding Plan Output

```bash
$ terraform plan

# Create new resource
  + resource "aws_instance" "new" {
      + ami           = "ami-12345678"
      + instance_type = "t2.micro"
    }

# Update in-place
  ~ resource "aws_instance" "existing" {
      ~ tags = {
          ~ "Name" = "old-name" -> "new-name"
        }
    }

# Destroy and recreate (forces replacement)
-/+ resource "aws_instance" "replaced" {
      ~ ami           = "ami-old" -> "ami-new" # forces replacement
      ~ id            = "i-12345" -> (known after apply)
    }

# Destroy
  - resource "aws_instance" "removed" {
      - ami           = "ami-12345678"
      - instance_type = "t2.micro"
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

---

## Resource Addressing

### Full Address Format

```hcl
# Format: [module.]resource_type.resource_name[index]

# Simple resource
aws_instance.web

# Resource with count
aws_instance.web[0]
aws_instance.web[2]

# Resource with for_each
aws_instance.web["production"]
aws_instance.web["staging"]

# Resource in module
module.vpc.aws_subnet.public

# Nested module
module.app.module.database.aws_db_instance.main
```

### Using Addresses

```bash
# Target specific resource
terraform plan -target=aws_instance.web
terraform apply -target=aws_instance.web[0]

# Refresh specific resource
terraform refresh -target=aws_instance.web

# Show resource state
terraform state show aws_instance.web
```

---

## Resource Block Examples

### Complete AWS Instance

```hcl
resource "aws_instance" "web" {
  # Required
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  # Network
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  # Storage
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    delete_on_termination = true
    encrypted             = true
  }

  ebs_block_device {
    device_name = "/dev/sdf"
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }

  # User data (bootstrap script)
  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
  EOF

  # Key pair for SSH
  key_name = aws_key_pair.deployer.key_name

  # Tags
  tags = {
    Name        = "web-server"
    Environment = var.environment
  }

  # Lifecycle
  lifecycle {
    create_before_destroy = true
    ignore_changes        = [user_data]
  }
}
```

### Complete Azure VM

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"
  admin_username      = "adminuser"

  network_interface_ids = [
    azurerm_network_interface.web.id,
  ]

  admin_ssh_key {
    username   = "adminuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts"
    version   = "latest"
  }

  tags = {
    Environment = "production"
  }
}
```

---

## Special Resources

### Null Resource

For running provisioners without creating infrastructure.

```hcl
resource "null_resource" "example" {
  # Triggers re-run when value changes
  triggers = {
    always_run = timestamp()
    # or based on another resource
    instance_id = aws_instance.web.id
  }

  provisioner "local-exec" {
    command = "echo 'Instance ${aws_instance.web.id} created'"
  }
}
```

### Random Resources

For generating random values.

```hcl
resource "random_id" "bucket_suffix" {
  byte_length = 8
}

resource "random_password" "db_password" {
  length  = 16
  special = true
}

resource "random_pet" "server_name" {
  length = 2
}

# Use random values
resource "aws_s3_bucket" "data" {
  bucket = "my-bucket-${random_id.bucket_suffix.hex}"
}
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Resource** | Infrastructure object managed by Terraform |
| **Type** | Provider-specific resource type (e.g., aws_instance) |
| **Name** | Local identifier for the resource |
| **Arguments** | Configuration parameters |
| **Attributes** | Values exposed after creation |
| **Dependencies** | Order of resource creation |

### Resource Reference Syntax

```hcl
# Reference format
<RESOURCE_TYPE>.<NAME>.<ATTRIBUTE>

# Examples
aws_instance.web.id
aws_vpc.main.cidr_block
aws_subnet.public[0].id
aws_instance.servers["web"].public_ip
```

### Key Commands

```bash
terraform plan              # Preview changes
terraform apply             # Apply changes
terraform destroy           # Destroy resources
terraform state show <addr> # Show resource state
terraform graph             # View dependencies
```
