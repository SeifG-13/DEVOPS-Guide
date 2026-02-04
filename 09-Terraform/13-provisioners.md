# Terraform Provisioners

## What are Provisioners?

Provisioners are used to execute scripts or commands on resources after they are created or destroyed. They are a "last resort" for tasks that can't be handled by Terraform's declarative model.

```
┌─────────────────────────────────────────────────────────────┐
│                    PROVISIONERS                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Resource Creation                                          │
│  ┌─────────────────┐                                       │
│  │ terraform apply │                                       │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ Create Resource │                                       │
│  │ (e.g., EC2)     │                                       │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ Run Provisioner │  ◄── Scripts, commands               │
│  │ (if defined)    │      run AFTER creation              │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ Resource Ready  │                                       │
│  └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Provisioner Types

### local-exec

Executes commands on the machine running Terraform.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  # Run on local machine after instance is created
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ip_addresses.txt"
  }
}
```

#### local-exec Options

```hcl
provisioner "local-exec" {
  # Command to execute
  command = "ansible-playbook -i '${self.public_ip},' playbook.yml"

  # Working directory
  working_dir = "/path/to/ansible"

  # Interpreter (default: /bin/sh on Linux, cmd.exe on Windows)
  interpreter = ["bash", "-c"]

  # Environment variables
  environment = {
    HOST     = self.public_ip
    SSH_KEY  = var.ssh_private_key_path
  }

  # When to run (default: create)
  when = create  # or destroy
}
```

#### Practical local-exec Examples

```hcl
# Update Ansible inventory
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command = <<-EOT
      echo "[webservers]" > inventory.ini
      echo "${self.public_ip} ansible_user=ubuntu" >> inventory.ini
    EOT
  }
}

# Run Ansible playbook
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command = "sleep 60 && ansible-playbook -i '${self.public_ip},' configure.yml"
  }
}

# Notify external system
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command = "curl -X POST -d '{\"ip\":\"${self.public_ip}\"}' https://api.example.com/servers"
  }
}
```

---

### remote-exec

Executes commands on the remote resource via SSH or WinRM.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name

  # Connection details for remote-exec
  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  # Run on remote machine
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx"
    ]
  }
}
```

#### remote-exec Options

```hcl
provisioner "remote-exec" {
  # Inline commands (list of strings)
  inline = [
    "echo 'Hello World'",
    "sudo apt-get update"
  ]

  # OR script file (local file, copied and executed)
  script = "scripts/setup.sh"

  # OR multiple scripts
  scripts = [
    "scripts/setup.sh",
    "scripts/configure.sh"
  ]
}
```

#### Connection Block

```hcl
# SSH Connection (Linux)
connection {
  type        = "ssh"
  user        = "ubuntu"
  private_key = file("~/.ssh/id_rsa")
  host        = self.public_ip

  # Optional settings
  port        = 22
  timeout     = "5m"
  agent       = false  # Use SSH agent

  # Bastion/Jump host
  bastion_host        = "bastion.example.com"
  bastion_user        = "ubuntu"
  bastion_private_key = file("~/.ssh/bastion_key")
}

# WinRM Connection (Windows)
connection {
  type     = "winrm"
  user     = "Administrator"
  password = var.admin_password
  host     = self.public_ip

  # WinRM settings
  port     = 5986
  https    = true
  insecure = true  # Skip certificate verification
  timeout  = "10m"
}
```

---

### file

Copies files or directories to the remote resource.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  # Copy single file
  provisioner "file" {
    source      = "configs/app.conf"
    destination = "/tmp/app.conf"
  }

  # Copy directory
  provisioner "file" {
    source      = "scripts/"  # Trailing slash = contents only
    destination = "/opt/scripts"
  }

  # Copy from content
  provisioner "file" {
    content     = "Hello, ${var.name}!"
    destination = "/tmp/greeting.txt"
  }
}
```

---

## Provisioner Behavior

### When to Run

```hcl
# Run on create (default)
provisioner "local-exec" {
  command = "echo 'Resource created!'"
  when    = create
}

# Run on destroy
provisioner "local-exec" {
  command = "echo 'Resource being destroyed!'"
  when    = destroy
}
```

### Failure Handling

```hcl
# Default: Fail resource creation if provisioner fails
provisioner "remote-exec" {
  inline = ["exit 1"]  # This will fail the resource
}

# Continue on failure
provisioner "remote-exec" {
  inline = ["exit 1"]
  on_failure = continue  # Resource still created
}

# Fail (default behavior)
provisioner "remote-exec" {
  inline = ["exit 1"]
  on_failure = fail
}
```

### Multiple Provisioners

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  # Provisioners run in order
  provisioner "file" {
    source      = "scripts/setup.sh"
    destination = "/tmp/setup.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/setup.sh",
      "/tmp/setup.sh"
    ]
  }

  provisioner "local-exec" {
    command = "echo 'Setup complete for ${self.public_ip}'"
  }
}
```

---

## Why Avoid Provisioners

### Problems with Provisioners

| Issue | Description |
|-------|-------------|
| **Not Declarative** | Imperative scripts break IaC model |
| **No State Tracking** | Changes not tracked in state |
| **Timing Issues** | May run before resource is ready |
| **Idempotency** | Scripts may not be idempotent |
| **Error Handling** | Failure handling is limited |
| **Drift** | Can't detect/fix configuration drift |

### Better Alternatives

```hcl
# AVOID: Provisioner for software installation
provisioner "remote-exec" {
  inline = ["sudo apt-get install -y nginx"]
}

# BETTER: Use user_data (cloud-init)
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
  EOF
}

# BEST: Use pre-built AMI with Packer
resource "aws_instance" "web" {
  ami           = data.aws_ami.custom_nginx.id
  instance_type = "t2.micro"
}
```

### Alternatives to Provisioners

| Task | Instead of Provisioner | Use |
|------|----------------------|-----|
| Install software | remote-exec | user_data, cloud-init |
| Configure server | remote-exec | Ansible, Chef, Puppet |
| Bootstrap instance | remote-exec | Pre-built AMI (Packer) |
| Notify services | local-exec | AWS SNS, webhooks |
| Run one-time setup | remote-exec | User data, init systems |

---

## When to Use Provisioners

### Acceptable Use Cases

```hcl
# 1. Bootstrap configuration management tools
resource "aws_instance" "web" {
  # ...

  provisioner "remote-exec" {
    inline = [
      "curl -L https://chef.io/chef/install.sh | sudo bash",
      "sudo chef-client -j /etc/chef/first-boot.json"
    ]
  }
}

# 2. Notify external systems
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command = "curl -X POST https://api.example.com/register -d '{\"ip\":\"${self.public_ip}\"}'"
  }
}

# 3. Cleanup on destroy
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    when    = destroy
    command = "curl -X DELETE https://api.example.com/servers/${self.id}"
  }
}
```

---

## null_resource with Provisioners

Use null_resource for provisioners not tied to a specific resource.

```hcl
resource "null_resource" "configure_cluster" {
  # Trigger when instances change
  triggers = {
    instance_ids = join(",", aws_instance.web[*].id)
  }

  provisioner "local-exec" {
    command = "ansible-playbook -i inventory.ini cluster.yml"
  }

  depends_on = [aws_instance.web]
}
```

### Re-running Provisioners

```hcl
resource "null_resource" "always_run" {
  # Always run by using timestamp
  triggers = {
    always_run = timestamp()
  }

  provisioner "local-exec" {
    command = "echo 'This runs every apply'"
  }
}

resource "null_resource" "on_file_change" {
  # Run when file changes
  triggers = {
    config_hash = filemd5("config/settings.json")
  }

  provisioner "local-exec" {
    command = "deploy.sh"
  }
}
```

---

## Complete Example

```hcl
# Key pair for SSH access
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_rsa.pub")
}

# Security group allowing SSH
resource "aws_security_group" "allow_ssh" {
  name = "allow-ssh"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 instance with provisioners
resource "aws_instance" "web" {
  ami                    = "ami-12345678"
  instance_type          = "t2.micro"
  key_name               = aws_key_pair.deployer.key_name
  vpc_security_group_ids = [aws_security_group.allow_ssh.id]

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
    timeout     = "5m"
  }

  # Copy application files
  provisioner "file" {
    source      = "app/"
    destination = "/opt/app"
  }

  # Install and configure
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y python3-pip",
      "cd /opt/app && pip3 install -r requirements.txt",
      "sudo systemctl restart myapp"
    ]
  }

  # Local notification
  provisioner "local-exec" {
    command = "echo 'Deployed to ${self.public_ip}' | mail -s 'Deployment' admin@example.com"
  }

  # Cleanup on destroy
  provisioner "local-exec" {
    when    = destroy
    command = "ssh ubuntu@${self.public_ip} 'sudo systemctl stop myapp'"
    on_failure = continue
  }

  tags = {
    Name = "web-server"
  }
}
```

---

## Summary

| Provisioner | Runs On | Use Case |
|-------------|---------|----------|
| `local-exec` | Terraform machine | Scripts, notifications, Ansible |
| `remote-exec` | Target resource | Remote commands (last resort) |
| `file` | Target resource | Copy files to remote |

### Key Options

| Option | Description |
|--------|-------------|
| `when` | `create` or `destroy` |
| `on_failure` | `fail` or `continue` |
| `connection` | SSH/WinRM settings |

### Best Practices

1. **Avoid provisioners** when possible
2. **Use user_data** for initial bootstrapping
3. **Use configuration management** tools (Ansible, Chef)
4. **Build immutable images** with Packer
5. **If using provisioners**, make scripts idempotent
6. **Use null_resource** for provisioners not tied to resources
