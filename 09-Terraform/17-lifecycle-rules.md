# Terraform Lifecycle Rules

## What are Lifecycle Rules?

Lifecycle rules are meta-arguments that customize how Terraform creates, updates, and destroys resources. They provide fine-grained control over resource behavior.

```
┌─────────────────────────────────────────────────────────────┐
│                    LIFECYCLE RULES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  resource "aws_instance" "web" {                           │
│    ami           = "ami-12345678"                          │
│    instance_type = "t2.micro"                              │
│                                                             │
│    lifecycle {                                              │
│      create_before_destroy = true                          │
│      prevent_destroy       = false                         │
│      ignore_changes        = [tags]                        │
│      replace_triggered_by  = [...]                         │
│                                                             │
│      precondition { ... }                                  │
│      postcondition { ... }                                 │
│    }                                                        │
│  }                                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## create_before_destroy

Create the replacement resource before destroying the original.

### Default Behavior (Without)

```
┌─────────────────────────────────────────────────────────────┐
│            DEFAULT: DESTROY THEN CREATE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Destroy old instance                                    │
│     ┌─────────┐                                            │
│     │ web-old │ ──► DESTROYED                              │
│     └─────────┘                                            │
│                    ⚠️ Downtime here!                        │
│  2. Create new instance                                     │
│                         ┌─────────┐                        │
│              CREATED ◄── │ web-new │                        │
│                         └─────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### With create_before_destroy

```
┌─────────────────────────────────────────────────────────────┐
│           CREATE BEFORE DESTROY                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Create new instance                                     │
│     ┌─────────┐        ┌─────────┐                        │
│     │ web-old │        │ web-new │ ◄── CREATED             │
│     └─────────┘        └─────────┘                        │
│                                                             │
│  2. Update references (if any)                              │
│                                                             │
│  3. Destroy old instance                                    │
│     ┌─────────┐        ┌─────────┐                        │
│     │ web-old │ ──►    │ web-new │                         │
│     └─────────┘        └─────────┘                        │
│      DESTROYED           ✓ Running                         │
│                                                             │
│  Result: Zero downtime!                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Example

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true
  }
}
```

### Use Cases

```hcl
# Auto Scaling Launch Configuration
resource "aws_launch_configuration" "web" {
  name_prefix   = "web-"
  image_id      = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
  }
}

# Security Group (often has dependent resources)
resource "aws_security_group" "web" {
  name_prefix = "web-sg-"
  vpc_id      = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}

# IAM Role
resource "aws_iam_role" "app" {
  name_prefix = "app-role-"

  assume_role_policy = jsonencode({...})

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## prevent_destroy

Prevent Terraform from destroying the resource.

### Basic Usage

```hcl
resource "aws_db_instance" "production" {
  identifier     = "production-database"
  engine         = "postgres"
  instance_class = "db.r5.large"

  lifecycle {
    prevent_destroy = true
  }
}
```

### Behavior

```bash
# Attempting to destroy
$ terraform destroy

Error: Instance cannot be destroyed

  on main.tf line 1:
   1: resource "aws_db_instance" "production" {

Resource aws_db_instance.production has lifecycle.prevent_destroy set,
but the plan calls for this resource to be destroyed.
```

### Removing Protection

```hcl
# To destroy, first remove or set to false
resource "aws_db_instance" "production" {
  # ...

  lifecycle {
    prevent_destroy = false  # Now can be destroyed
  }
}
```

### Use Cases

```hcl
# Production databases
resource "aws_db_instance" "prod_db" {
  identifier = "production-db"
  lifecycle {
    prevent_destroy = true
  }
}

# S3 buckets with important data
resource "aws_s3_bucket" "backups" {
  bucket = "company-critical-backups"
  lifecycle {
    prevent_destroy = true
  }
}

# KMS keys
resource "aws_kms_key" "encryption" {
  description = "Production encryption key"
  lifecycle {
    prevent_destroy = true
  }
}

# Route53 hosted zones
resource "aws_route53_zone" "primary" {
  name = "example.com"
  lifecycle {
    prevent_destroy = true
  }
}
```

---

## ignore_changes

Ignore changes to specified attributes.

### Basic Usage

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    # Ignore changes to tags
    ignore_changes = [tags]
  }
}
```

### Multiple Attributes

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    ignore_changes = [
      tags,
      user_data,
      ami,
    ]
  }
}
```

### Ignore All

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    # Ignore ALL attribute changes
    ignore_changes = all
  }
}
```

### Nested Attributes

```hcl
resource "aws_autoscaling_group" "web" {
  name             = "web-asg"
  min_size         = 1
  max_size         = 10
  desired_capacity = 2

  lifecycle {
    # Ignore changes to desired_capacity (managed by auto-scaling)
    ignore_changes = [desired_capacity]
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  root_block_device {
    volume_size = 20
  }

  lifecycle {
    # Ignore nested attribute changes
    ignore_changes = [
      root_block_device[0].volume_size,
    ]
  }
}
```

### Common Use Cases

```hcl
# ASG desired capacity (managed by auto-scaling policies)
resource "aws_autoscaling_group" "web" {
  desired_capacity = 2
  lifecycle {
    ignore_changes = [desired_capacity]
  }
}

# ECS task definition (managed by CI/CD)
resource "aws_ecs_service" "app" {
  task_definition = aws_ecs_task_definition.app.arn
  lifecycle {
    ignore_changes = [task_definition]
  }
}

# Lambda function code (deployed separately)
resource "aws_lambda_function" "api" {
  filename = "dummy.zip"
  lifecycle {
    ignore_changes = [filename, source_code_hash]
  }
}

# Tags managed by external system
resource "aws_instance" "web" {
  tags = {}
  lifecycle {
    ignore_changes = [tags]
  }
}
```

---

## replace_triggered_by

Force resource replacement when referenced resources change.

### Basic Usage

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    # Replace when AMI resource changes
    replace_triggered_by = [
      aws_ami_from_instance.golden.id
    ]
  }
}
```

### Multiple Triggers

```hcl
resource "null_resource" "config" {
  triggers = {
    config_hash = filemd5("config/settings.json")
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    replace_triggered_by = [
      # Replace when config changes
      null_resource.config,
      # Replace when security group changes
      aws_security_group.web.id,
    ]
  }
}
```

### Use Cases

```hcl
# Replace when configuration changes
resource "null_resource" "app_config" {
  triggers = {
    config = filemd5("app/config.yaml")
  }
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    replace_triggered_by = [null_resource.app_config]
  }
}

# Replace when certificate renews
resource "aws_acm_certificate" "main" {
  domain_name = "example.com"
  # ...
}

resource "aws_lb_listener" "https" {
  # ...

  lifecycle {
    replace_triggered_by = [aws_acm_certificate.main.id]
  }
}
```

---

## Preconditions

Validate conditions before creating or updating resources.

### Basic Usage

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = var.instance_type != "t2.nano"
      error_message = "Instance type t2.nano is not allowed for web servers."
    }
  }
}
```

### Multiple Preconditions

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = data.aws_ami.ubuntu.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }

    precondition {
      condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
      error_message = "Instance type must be t3.micro, t3.small, or t3.medium."
    }

    precondition {
      condition     = var.environment != "production" || var.instance_type != "t3.micro"
      error_message = "Production instances cannot use t3.micro."
    }
  }
}
```

### Validating Data Sources

```hcl
data "aws_ami" "app" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["myapp-*"]
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = "t3.micro"

  lifecycle {
    precondition {
      condition     = data.aws_ami.app.id != null
      error_message = "No AMI found matching the filter criteria."
    }
  }
}
```

---

## Postconditions

Validate conditions after resource creation.

### Basic Usage

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP address."
    }
  }
}
```

### Multiple Postconditions

```hcl
resource "aws_db_instance" "main" {
  identifier     = "production-db"
  engine         = "postgres"
  engine_version = "14.7"
  instance_class = "db.r5.large"

  lifecycle {
    postcondition {
      condition     = self.status == "available"
      error_message = "Database instance is not available."
    }

    postcondition {
      condition     = self.multi_az == true
      error_message = "Database must be Multi-AZ for production."
    }
  }
}
```

### Using self Reference

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Environment = var.environment
  }

  lifecycle {
    postcondition {
      condition     = contains(keys(self.tags), "Environment")
      error_message = "Instance must have an Environment tag."
    }

    postcondition {
      condition     = self.instance_state == "running"
      error_message = "Instance must be in running state."
    }
  }
}
```

---

## Combining Lifecycle Rules

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name

  tags = {
    Name        = "web-server"
    Environment = var.environment
  }

  lifecycle {
    # Zero-downtime replacement
    create_before_destroy = true

    # Protect from accidental deletion
    prevent_destroy = true

    # Ignore external tag changes
    ignore_changes = [
      tags["LastModified"],
      tags["ModifiedBy"],
    ]

    # Validation
    precondition {
      condition     = var.environment != "production" || var.instance_type != "t3.micro"
      error_message = "Production requires larger instance types."
    }

    postcondition {
      condition     = self.public_ip != null
      error_message = "Instance must have a public IP."
    }
  }
}
```

---

## Best Practices

### 1. Use create_before_destroy for Dependent Resources

```hcl
resource "aws_security_group" "web" {
  name_prefix = "web-"
  # Resources depend on this SG

  lifecycle {
    create_before_destroy = true
  }
}
```

### 2. Protect Critical Resources

```hcl
resource "aws_db_instance" "production" {
  # Production database
  lifecycle {
    prevent_destroy = true
  }
}
```

### 3. Ignore Externally Managed Attributes

```hcl
resource "aws_autoscaling_group" "web" {
  # Desired capacity managed by policies
  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

### 4. Validate Assumptions

```hcl
resource "aws_instance" "web" {
  lifecycle {
    precondition {
      condition     = var.subnet_id != ""
      error_message = "Subnet ID must be provided."
    }
  }
}
```

---

## Summary

| Rule | Purpose |
|------|---------|
| `create_before_destroy` | Zero-downtime replacements |
| `prevent_destroy` | Protect critical resources |
| `ignore_changes` | Ignore external modifications |
| `replace_triggered_by` | Force replacement on changes |
| `precondition` | Validate before apply |
| `postcondition` | Validate after apply |

### Quick Reference

```hcl
lifecycle {
  create_before_destroy = true
  prevent_destroy       = true
  ignore_changes        = [tags, user_data]
  replace_triggered_by  = [null_resource.trigger]

  precondition {
    condition     = var.env != "prod" || var.size != "small"
    error_message = "Production requires larger size."
  }

  postcondition {
    condition     = self.status == "active"
    error_message = "Resource must be active."
  }
}
```
