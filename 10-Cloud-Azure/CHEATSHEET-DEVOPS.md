# Azure Cloud Cheat Sheet for DevOps Engineers

## Quick Reference - Azure CLI

### Authentication & Account
```bash
az login                            # Interactive login
az login --service-principal -u <app-id> -p <password> --tenant <tenant>
az account list                     # List subscriptions
az account set --subscription <id>  # Set subscription
az account show                     # Current subscription
```

### Resource Groups
```bash
az group list                       # List resource groups
az group create -n myRG -l eastus   # Create resource group
az group delete -n myRG --yes       # Delete resource group
az group show -n myRG               # Show details
```

### Virtual Machines
```bash
# Create VM
az vm create \
  -g myRG \
  -n myVM \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys

# Manage VMs
az vm list -g myRG                  # List VMs
az vm show -g myRG -n myVM          # VM details
az vm start -g myRG -n myVM         # Start VM
az vm stop -g myRG -n myVM          # Stop VM
az vm deallocate -g myRG -n myVM    # Deallocate (stop billing)
az vm delete -g myRG -n myVM --yes  # Delete VM
az vm list-ip-addresses -g myRG -n myVM  # Get IP

# SSH into VM
az vm run-command invoke -g myRG -n myVM \
  --command-id RunShellScript \
  --scripts "apt-get update"
```

### Azure Kubernetes Service (AKS)
```bash
# Create AKS cluster
az aks create \
  -g myRG \
  -n myAKS \
  --node-count 3 \
  --node-vm-size Standard_D2_v2 \
  --enable-managed-identity \
  --generate-ssh-keys

# Manage AKS
az aks get-credentials -g myRG -n myAKS  # Get kubeconfig
az aks show -g myRG -n myAKS             # Cluster details
az aks scale -g myRG -n myAKS --node-count 5
az aks upgrade -g myRG -n myAKS --kubernetes-version 1.28.0
az aks stop -g myRG -n myAKS             # Stop cluster
az aks start -g myRG -n myAKS            # Start cluster
```

### Azure Container Registry (ACR)
```bash
# Create ACR
az acr create -g myRG -n myacr --sku Basic

# Manage ACR
az acr login -n myacr                    # Login to registry
az acr build -r myacr -t myimage:v1 .    # Build and push
az acr repository list -n myacr          # List repos
az acr repository show-tags -n myacr --repository myimage

# Attach ACR to AKS
az aks update -g myRG -n myAKS --attach-acr myacr
```

### App Service
```bash
# Create App Service
az appservice plan create -g myRG -n myPlan --sku P1V2 --is-linux
az webapp create -g myRG -p myPlan -n myApp --runtime "DOTNET|8.0"

# Deploy
az webapp deploy -g myRG -n myApp --src-path app.zip
az webapp deployment source config-zip -g myRG -n myApp --src app.zip

# Configuration
az webapp config appsettings set -g myRG -n myApp --settings KEY=value
az webapp config connection-string set -g myRG -n myApp \
  --connection-string-type SQLAzure \
  --settings DefaultConnection="Server=..."

# Logs
az webapp log tail -g myRG -n myApp
az webapp log download -g myRG -n myApp
```

### Azure SQL
```bash
# Create SQL Server
az sql server create -g myRG -n mysqlserver \
  -u sqladmin -p <password> -l eastus

# Create Database
az sql db create -g myRG -s mysqlserver -n mydb --service-objective S0

# Firewall rules
az sql server firewall-rule create -g myRG -s mysqlserver \
  -n AllowAzure --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

### Storage
```bash
# Create storage account
az storage account create -g myRG -n mystorageacct --sku Standard_LRS

# Blob operations
az storage container create -n mycontainer --account-name mystorageacct
az storage blob upload -c mycontainer --account-name mystorageacct \
  -f file.txt -n file.txt
az storage blob list -c mycontainer --account-name mystorageacct
```

### Key Vault
```bash
# Create Key Vault
az keyvault create -g myRG -n mykeyvault -l eastus

# Manage secrets
az keyvault secret set --vault-name mykeyvault -n mysecret --value "secret123"
az keyvault secret show --vault-name mykeyvault -n mysecret
az keyvault secret list --vault-name mykeyvault
```

---

## Azure Services Overview

| Service | Purpose | Use Case |
|---------|---------|----------|
| **Compute** |||
| Virtual Machines | IaaS VMs | Custom workloads |
| App Service | PaaS web hosting | Web apps, APIs |
| Azure Functions | Serverless compute | Event-driven |
| AKS | Managed Kubernetes | Container orchestration |
| Container Instances | Simple containers | Quick deployments |
| **Storage** |||
| Blob Storage | Object storage | Files, backups |
| Azure Files | File shares | Shared storage |
| Disk Storage | VM disks | Persistent storage |
| **Database** |||
| Azure SQL | Managed SQL Server | Relational data |
| Cosmos DB | NoSQL database | Global distribution |
| Azure Database for PostgreSQL | Managed PostgreSQL | Open source SQL |
| **Networking** |||
| Virtual Network | Private network | Isolation |
| Load Balancer | L4 load balancing | Traffic distribution |
| Application Gateway | L7 load balancing | WAF, SSL |
| Azure DNS | DNS hosting | Domain management |
| **Security** |||
| Key Vault | Secret management | Keys, secrets, certs |
| Azure AD | Identity management | Authentication |
| **Monitoring** |||
| Azure Monitor | Metrics and logs | Observability |
| Application Insights | APM | App performance |
| Log Analytics | Log aggregation | Log queries |

---

## Interview Q&A

### Q1: What is the difference between Azure regions and availability zones?
**A:**
- **Region**: Geographic area containing data centers (e.g., East US)
- **Availability Zone**: Physically separate data centers within a region (3 zones per region)
Use zones for high availability within a region.

### Q2: Explain Azure Resource Manager (ARM)
**A:** ARM is Azure's deployment and management service. All operations go through ARM API. Key concepts:
- Resource groups for organization
- Templates for IaC (ARM templates, Bicep)
- RBAC for access control
- Tags for organization and billing

### Q3: What is the difference between App Service and Azure Functions?
**A:**
- **App Service**: Always-on web hosting, predictable workloads, more control
- **Azure Functions**: Serverless, event-driven, pay-per-execution, auto-scale to zero

### Q4: How do you secure an Azure App Service?
**A:**
- Enable HTTPS only
- Use managed identity for Azure resources
- Store secrets in Key Vault
- Enable Azure AD authentication
- Configure IP restrictions
- Use private endpoints for databases

### Q5: What is Azure Virtual Network (VNet)?
**A:** Private network in Azure for:
- Isolating resources
- Defining subnets
- Network security groups (NSGs)
- VPN/ExpressRoute connectivity
- Private endpoints

### Q6: How does AKS networking work?
**A:**
- **Kubenet**: Basic networking, NAT for pods
- **Azure CNI**: Pods get VNet IPs, better integration
- **Azure CNI Overlay**: Efficient IP usage with overlay

### Q7: What is Azure Policy?
**A:** Governance service to enforce rules:
- Require tags on resources
- Restrict resource types
- Enforce naming conventions
- Audit compliance

### Q8: How do you implement disaster recovery in Azure?
**A:**
- Geo-redundant storage (GRS)
- Azure Site Recovery
- Multi-region deployment
- Traffic Manager for failover
- Database geo-replication

### Q9: What is Azure DevOps vs GitHub Actions?
**A:**
- **Azure DevOps**: Full ALM suite (repos, boards, pipelines, artifacts, test plans)
- **GitHub Actions**: CI/CD integrated with GitHub
Both work with Azure deployments.

### Q10: How do you monitor Azure resources?
**A:**
- **Azure Monitor**: Metrics and alerts
- **Application Insights**: Application performance
- **Log Analytics**: Query logs with KQL
- **Azure Alerts**: Proactive notifications

---

## Azure Networking

### Network Security Group (NSG) Rules
```bash
# Create NSG
az network nsg create -g myRG -n myNSG

# Add rule
az network nsg rule create -g myRG --nsg-name myNSG \
  -n AllowHTTP --priority 100 \
  --destination-port-ranges 80 443 \
  --access Allow --protocol Tcp --direction Inbound
```

### VNet and Subnets
```bash
# Create VNet
az network vnet create -g myRG -n myVNet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name default --subnet-prefix 10.0.1.0/24

# Add subnet
az network vnet subnet create -g myRG --vnet-name myVNet \
  -n appSubnet --address-prefix 10.0.2.0/24
```

---

## Cost Management

### Best Practices
1. **Use reserved instances** - 1-3 year commitment for discounts
2. **Right-size resources** - Monitor and adjust VM sizes
3. **Auto-shutdown dev VMs** - Schedule non-production resources
4. **Use spot instances** - For interruptible workloads
5. **Set budgets and alerts** - Monitor spending
6. **Use Azure Advisor** - Cost optimization recommendations

### Cost Analysis
```bash
# View costs
az consumption usage list --start-date 2024-01-01 --end-date 2024-01-31

# Set budget
az consumption budget create -b myBudget --amount 1000 \
  --time-grain Monthly --start-date 2024-01-01 --end-date 2024-12-31
```

---

## Best Practices

1. **Use managed identities** - No credentials in code
2. **Store secrets in Key Vault** - Never in app settings
3. **Enable monitoring** - Application Insights + Azure Monitor
4. **Use IaC** - Terraform, Bicep, or ARM templates
5. **Implement tagging** - Organization and cost tracking
6. **Enable backup** - VMs, databases, and storage
7. **Use availability zones** - High availability
8. **Follow naming conventions** - Consistent resource naming
