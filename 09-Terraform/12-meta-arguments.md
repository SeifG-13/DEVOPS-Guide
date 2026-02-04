# Terraform Meta-Arguments

## What are Meta-Arguments?

Meta-arguments are special arguments that can be used with any resource type to change its behavior. They are defined by Terraform itself, not by providers.

```
┌─────────────────────────────────────────────────────────────┐
│                    META-ARGUMENTS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  resource "aws_instance" "web" {                           │
│    # Regular arguments (provider-specific)                  │
│    ami           = "ami-12345678"                          │
│    instance_type = "t2.micro"                              │
│                                                             │
│    # Meta-arguments (Terraform-defined)                     │
│    count      = 3           # Create multiple              │
│    for_each   = var.map     # Iterate over collection      │
│    depends_on = [...]       # Explicit dependencies        │
│    provider   = aws.west    # Select provider              │
│    lifecycle  { ... }       # Customize lifecycle          │
│  }                                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## count

Create multiple instances of a resource using a number.

### Basic Usage

```hcl
# Create 3 instances
resource "aws_instance" "web" {
  count = 3

  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server-${count.index}"
  }
}

# Access by index
output "instance_ids" {
  value = aws_instance.web[*].id  # All IDs
}

output "first_instance" {
  value = aws_instance.web[0].id  # First instance
}
```

### count.index

```hcl
resource "aws_instance" "web" {
  count = 3

  ami           = "ami-12345678"
  instance_type = "t2.micro"

  # count.index starts at 0
  tags = {
    Name  = "web-${count.index + 1}"  # web-1, web-2, web-3
    Index = count.index                # 0, 1, 2
  }
}
```

### Conditional Creation

```hcl
variable "create_bastion" {
  type    = bool
  default = true
}

resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0

  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = "bastion"
  }
}

# Reference with conditional
output "bastion_ip" {
  value = var.create_bastion ? aws_instance.bastion[0].public_ip : null
}
```

### Using with Lists

```hcl
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "public-${var.availability_zones[count.index]}"
  }
}
```

---

## for_each

Create multiple instances using a map or set.

### Basic Usage with Set

```hcl
variable "instance_names" {
  type    = set(string)
  default = ["web", "api", "db"]
}

resource "aws_instance" "servers" {
  for_each = var.instance_names

  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = each.value  # "web", "api", "db"
  }
}

# Access by key
output "web_instance_id" {
  value = aws_instance.servers["web"].id
}
```

### Using with Map

```hcl
variable "instances" {
  type = map(object({
    instance_type = string
    ami           = string
  }))
  default = {
    web = {
      instance_type = "t2.micro"
      ami           = "ami-12345678"
    }
    api = {
      instance_type = "t2.small"
      ami           = "ami-12345678"
    }
    db = {
      instance_type = "t2.medium"
      ami           = "ami-87654321"
    }
  }
}

resource "aws_instance" "servers" {
  for_each = var.instances

  ami           = each.value.ami
  instance_type = each.value.instance_type

  tags = {
    Name = each.key  # "web", "api", "db"
    Type = each.value.instance_type
  }
}

# Output all IDs as map
output "instance_ids" {
  value = { for k, v in aws_instance.servers : k => v.id }
}
```

### each.key and each.value

```hcl
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "charlie"])

  name = each.key    # Same as each.value for sets
}

resource "aws_s3_bucket" "buckets" {
  for_each = {
    logs    = "us-east-1"
    backups = "us-west-2"
    data    = "eu-west-1"
  }

  bucket = "company-${each.key}"
  # each.key = "logs", "backups", "data"
  # each.value = "us-east-1", "us-west-2", "eu-west-1"

  tags = {
    Name   = each.key
    Region = each.value
  }
}
```

### Converting List to Set

```hcl
variable "users" {
  type    = list(string)
  default = ["alice", "bob", "charlie"]
}

# for_each requires set or map
resource "aws_iam_user" "users" {
  for_each = toset(var.users)

  name = each.value
}
```

---

## count vs for_each

### Comparison

| Aspect | count | for_each |
|--------|-------|----------|
| **Input** | Number | Set or Map |
| **Index** | Numeric (0, 1, 2) | String keys |
| **Reference** | `resource[0]` | `resource["key"]` |
| **Reordering** | Can cause issues | Stable |
| **Removal** | Affects all after | Only affects specific |

### When to Use count

```hcl
# Simple numeric scaling
resource "aws_instance" "web" {
  count = var.instance_count
  # ...
}

# Conditional creation
resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0
  # ...
}
```

### When to Use for_each

```hcl
# Named resources
resource "aws_iam_user" "users" {
  for_each = toset(var.user_names)
  # ...
}

# Complex configurations per instance
resource "aws_instance" "servers" {
  for_each = var.server_configs
  # ...
}
```

### The Index Problem with count

```hcl
# Initial state
variable "names" {
  default = ["a", "b", "c"]  # Creates [0]=a, [1]=b, [2]=c
}

# Remove "b" from list
variable "names" {
  default = ["a", "c"]  # Now [0]=a, [1]=c
}
# Terraform will destroy "b" AND recreate "c" (index shifted!)

# Solution: Use for_each
resource "aws_instance" "servers" {
  for_each = toset(var.names)
  # Removing "b" only destroys "b", "c" stays
}
```

---

## depends_on

Explicitly declare dependencies when Terraform can't detect them.

### When to Use

```hcl
# Terraform detects this dependency automatically
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id  # Implicit dependency
}

# Use depends_on when there's no reference
resource "aws_instance" "app" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  # No direct reference, but app needs role to exist first
  depends_on = [aws_iam_role.app_role]
}
```

### Module Dependencies

```hcl
module "vpc" {
  source = "./modules/vpc"
}

module "database" {
  source = "./modules/database"

  # Explicit dependency on VPC module
  depends_on = [module.vpc]
}
```

### Multiple Dependencies

```hcl
resource "aws_instance" "app" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  depends_on = [
    aws_iam_role.app_role,
    aws_s3_bucket.config,
    aws_security_group.app
  ]
}
```

### Warning: Overuse

```hcl
# Avoid unnecessary depends_on
# This is redundant - Terraform detects from subnet_id reference
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id

  # Unnecessary!
  depends_on = [aws_subnet.main]
}
```

---

## provider

Select which provider configuration to use.

### Multiple Provider Configurations

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "europe"
  region = "eu-west-1"
}

# Use default provider
resource "aws_instance" "east" {
  ami           = "ami-east"
  instance_type = "t2.micro"
}

# Use aliased provider
resource "aws_instance" "west" {
  provider      = aws.west
  ami           = "ami-west"
  instance_type = "t2.micro"
}

resource "aws_instance" "europe" {
  provider      = aws.europe
  ami           = "ami-europe"
  instance_type = "t2.micro"
}
```

### In Modules

```hcl
# Module that accepts provider configuration
module "vpc_west" {
  source = "./modules/vpc"

  providers = {
    aws = aws.west
  }

  cidr_block = "10.0.0.0/16"
}
```

---

## lifecycle

Customize how Terraform manages resource lifecycle.

### create_before_destroy

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
  }
}

# Order: Create new → Update references → Destroy old
# Useful for zero-downtime updates
```

### prevent_destroy

```hcl
resource "aws_db_instance" "production" {
  identifier = "production-db"
  # ...

  lifecycle {
    prevent_destroy = true
  }
}

# terraform destroy will fail for this resource
# Must remove lifecycle block first
```

### ignore_changes

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    # Ignore changes to these attributes
    ignore_changes = [
      tags,           # Ignore all tag changes
      user_data,      # Ignore user_data changes
    ]
  }
}

# Ignore all attributes (rarely needed)
lifecycle {
  ignore_changes = all
}
```

### replace_triggered_by

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    # Replace when these resources change
    replace_triggered_by = [
      aws_ami.custom.id,
      null_resource.config_change
    ]
  }
}

# Force replacement with null_resource
resource "null_resource" "config_change" {
  triggers = {
    config_hash = filemd5("config/app.json")
  }
}
```

See `17-lifecycle-rules.md` for more details on lifecycle management.

---

## Practical Examples

### Multi-Region Deployment

```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}

variable "regions" {
  default = {
    us_east = { provider = "us_east", ami = "ami-111" }
    us_west = { provider = "us_west", ami = "ami-222" }
  }
}

# Create instance in each region
resource "aws_instance" "web_us_east" {
  provider      = aws.us_east
  ami           = var.regions.us_east.ami
  instance_type = "t2.micro"
}

resource "aws_instance" "web_us_west" {
  provider      = aws.us_west
  ami           = var.regions.us_west.ami
  instance_type = "t2.micro"
}
```

### Dynamic Security Group Rules

```hcl
variable "ingress_rules" {
  default = {
    http = {
      port        = 80
      cidr_blocks = ["0.0.0.0/0"]
    }
    https = {
      port        = 443
      cidr_blocks = ["0.0.0.0/0"]
    }
    ssh = {
      port        = 22
      cidr_blocks = ["10.0.0.0/8"]
    }
  }
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

---

## Summary

| Meta-Argument | Purpose |
|---------------|---------|
| `count` | Create multiple instances (numeric) |
| `for_each` | Create multiple instances (map/set) |
| `depends_on` | Explicit dependencies |
| `provider` | Select provider configuration |
| `lifecycle` | Customize resource lifecycle |

### Quick Reference

```hcl
resource "aws_instance" "example" {
  # count - numeric instances
  count = var.instance_count

  # for_each - named instances
  for_each = var.instances

  # depends_on - explicit dependencies
  depends_on = [aws_iam_role.app]

  # provider - select provider
  provider = aws.west

  # lifecycle - customize behavior
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags]
    replace_triggered_by  = [null_resource.trigger]
  }
}
```
