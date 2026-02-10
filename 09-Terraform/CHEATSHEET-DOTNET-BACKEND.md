# Terraform Cheat Sheet for .NET Backend Engineers

## Quick Reference - Infrastructure for .NET Apps

### Azure App Service for .NET
```hcl
# Provider configuration
provider "azurerm" {
  features {}
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "myapp-rg"
  location = "East US"
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "myapp-plan"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = "P1v3"
}

# App Service for .NET
resource "azurerm_linux_web_app" "main" {
  name                = "myapp-webapp"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      dotnet_version = "8.0"
    }

    always_on        = true
    health_check_path = "/health"
  }

  app_settings = {
    "ASPNETCORE_ENVIRONMENT"           = "Production"
    "ApplicationInsights__ConnectionString" = azurerm_application_insights.main.connection_string
  }

  connection_string {
    name  = "DefaultConnection"
    type  = "SQLAzure"
    value = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name};Database=${azurerm_mssql_database.main.name};User ID=${var.sql_admin_login};Password=${var.sql_admin_password};"
  }
}
```

---

## Complete .NET Infrastructure

### Full Stack with SQL, Redis, App Service
```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "prod"
}

variable "sql_admin_login" {
  type      = string
  sensitive = true
}

variable "sql_admin_password" {
  type      = string
  sensitive = true
}

# main.tf
locals {
  common_tags = {
    Environment = var.environment
    Application = "MyDotNetApp"
    ManagedBy   = "Terraform"
  }
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "myapp-${var.environment}-rg"
  location = "East US"
  tags     = local.common_tags
}

# SQL Server
resource "azurerm_mssql_server" "main" {
  name                         = "myapp-${var.environment}-sql"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = var.sql_admin_login
  administrator_login_password = var.sql_admin_password
  minimum_tls_version          = "1.2"

  tags = local.common_tags
}

# SQL Database
resource "azurerm_mssql_database" "main" {
  name         = "myapp-db"
  server_id    = azurerm_mssql_server.main.id
  collation    = "SQL_Latin1_General_CP1_CI_AS"
  license_type = "LicenseIncluded"
  sku_name     = "S1"

  tags = local.common_tags
}

# SQL Firewall Rule (Allow Azure Services)
resource "azurerm_mssql_firewall_rule" "azure" {
  name             = "AllowAzureServices"
  server_id        = azurerm_mssql_server.main.id
  start_ip_address = "0.0.0.0"
  end_ip_address   = "0.0.0.0"
}

# Redis Cache
resource "azurerm_redis_cache" "main" {
  name                = "myapp-${var.environment}-redis"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  capacity            = 1
  family              = "C"
  sku_name            = "Standard"
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"

  tags = local.common_tags
}

# Application Insights
resource "azurerm_application_insights" "main" {
  name                = "myapp-${var.environment}-insights"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  application_type    = "web"

  tags = local.common_tags
}

# App Service
resource "azurerm_linux_web_app" "main" {
  name                = "myapp-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      dotnet_version = "8.0"
    }
    always_on         = true
    health_check_path = "/health"
  }

  app_settings = {
    "ASPNETCORE_ENVIRONMENT"                    = "Production"
    "APPLICATIONINSIGHTS_CONNECTION_STRING"     = azurerm_application_insights.main.connection_string
    "Redis__ConnectionString"                   = azurerm_redis_cache.main.primary_connection_string
  }

  connection_string {
    name  = "DefaultConnection"
    type  = "SQLAzure"
    value = "Server=tcp:${azurerm_mssql_server.main.fully_qualified_domain_name},1433;Database=${azurerm_mssql_database.main.name};User ID=${var.sql_admin_login};Password=${var.sql_admin_password};Encrypt=true;TrustServerCertificate=false;"
  }

  tags = local.common_tags
}

# outputs.tf
output "app_url" {
  value = "https://${azurerm_linux_web_app.main.default_hostname}"
}

output "sql_server_fqdn" {
  value = azurerm_mssql_server.main.fully_qualified_domain_name
}

output "app_insights_instrumentation_key" {
  value     = azurerm_application_insights.main.instrumentation_key
  sensitive = true
}
```

---

## Azure Kubernetes Service (AKS) for .NET

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  name                = "myapp-${var.environment}-aks"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  dns_prefix          = "myapp-${var.environment}"

  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_D2_v2"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = local.common_tags
}

# Container Registry
resource "azurerm_container_registry" "main" {
  name                = "myapp${var.environment}acr"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
  admin_enabled       = true

  tags = local.common_tags
}

# Allow AKS to pull from ACR
resource "azurerm_role_assignment" "aks_acr" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.main.kube_config_raw
  sensitive = true
}
```

---

## AWS Infrastructure for .NET

### ECS Fargate for .NET
```hcl
provider "aws" {
  region = "us-east-1"
}

# VPC
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "myapp-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "myapp-cluster"
}

# ECR Repository
resource "aws_ecr_repository" "main" {
  name = "myapp"
}

# Task Definition
resource "aws_ecs_task_definition" "main" {
  family                   = "myapp"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 256
  memory                   = 512
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([
    {
      name  = "myapp"
      image = "${aws_ecr_repository.main.repository_url}:latest"
      portMappings = [
        {
          containerPort = 8080
          protocol      = "tcp"
        }
      ]
      environment = [
        {
          name  = "ASPNETCORE_ENVIRONMENT"
          value = "Production"
        },
        {
          name  = "ASPNETCORE_URLS"
          value = "http://+:8080"
        }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/myapp"
          "awslogs-region"        = "us-east-1"
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}

# ECS Service
resource "aws_ecs_service" "main" {
  name            = "myapp-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.main.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = module.vpc.private_subnets
    security_groups = [aws_security_group.ecs.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.main.arn
    container_name   = "myapp"
    container_port   = 8080
  }
}
```

---

## Interview Q&A

### Q1: How do you provision infrastructure for a .NET application?
**A:**
1. Define resource group/VPC
2. Create database (SQL Server, PostgreSQL)
3. Create app hosting (App Service, ECS, AKS)
4. Configure networking (VNet, subnets, security groups)
5. Set up monitoring (Application Insights, CloudWatch)
6. Configure secrets (Key Vault, Secrets Manager)

### Q2: How do you handle connection strings in Terraform?
**A:**
```hcl
# Store password in variable (from env or tfvars)
variable "db_password" {
  type      = string
  sensitive = true
}

# Build connection string
locals {
  connection_string = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name};Database=${azurerm_mssql_database.main.name};User ID=${var.db_user};Password=${var.db_password};"
}

# Pass to App Service
resource "azurerm_linux_web_app" "main" {
  connection_string {
    name  = "DefaultConnection"
    type  = "SQLAzure"
    value = local.connection_string
  }
}
```

### Q3: How do you manage different environments (dev/staging/prod)?
**A:**
```hcl
# Using workspaces
terraform workspace new staging
terraform workspace new production

# Or using tfvars
terraform apply -var-file="environments/production.tfvars"

# Or using directory structure
environments/
  dev/
    main.tf
  staging/
    main.tf
  prod/
    main.tf
```

### Q4: How do you handle database migrations with Terraform?
**A:**
Terraform provisions the database, but migrations are handled separately:
- CI/CD pipeline runs `dotnet ef database update`
- Use provisioners (not recommended)
- Azure SQL deployment tasks

### Q5: How do you secure secrets in Terraform for .NET apps?
**A:**
```hcl
# Use Key Vault
resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = var.db_password
  key_vault_id = azurerm_key_vault.main.id
}

# Reference in App Service
resource "azurerm_linux_web_app" "main" {
  app_settings = {
    "ConnectionStrings__Default" = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.db_password.id})"
  }

  identity {
    type = "SystemAssigned"
  }
}

# Grant access
resource "azurerm_key_vault_access_policy" "app" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = azurerm_linux_web_app.main.identity[0].principal_id

  secret_permissions = ["Get"]
}
```

---

## Modules for .NET Infrastructure

### Reusable App Service Module
```hcl
# modules/app-service/variables.tf
variable "name" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "location" {
  type = string
}

variable "dotnet_version" {
  type    = string
  default = "8.0"
}

variable "app_settings" {
  type    = map(string)
  default = {}
}

# modules/app-service/main.tf
resource "azurerm_service_plan" "main" {
  name                = "${var.name}-plan"
  resource_group_name = var.resource_group_name
  location            = var.location
  os_type             = "Linux"
  sku_name            = "P1v3"
}

resource "azurerm_linux_web_app" "main" {
  name                = var.name
  resource_group_name = var.resource_group_name
  location            = var.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      dotnet_version = var.dotnet_version
    }
  }

  app_settings = merge({
    "ASPNETCORE_ENVIRONMENT" = "Production"
  }, var.app_settings)
}

# Using the module
module "api" {
  source = "./modules/app-service"

  name                = "myapi"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  app_settings = {
    "Feature__Enabled" = "true"
  }
}
```

---

## Best Practices for .NET Infrastructure

1. **Use managed services** - App Service, Azure SQL, RDS
2. **Enable health checks** - Set `health_check_path`
3. **Configure autoscaling** - Based on CPU/memory metrics
4. **Use Key Vault** - Never hardcode secrets
5. **Enable Application Insights** - Monitoring and diagnostics
6. **Use private endpoints** - Secure database connections
7. **Implement backup** - Database and blob storage backups
8. **Use staging slots** - Zero-downtime deployments
