# Azure Networking

## Virtual Network (VNet) Overview

A Virtual Network (VNet) is the fundamental building block for private networking in Azure.

```
┌─────────────────────────────────────────────────────────────┐
│                    AZURE VNET                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  VNet: vnet-myapp-prod (10.0.0.0/16)                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Subnet: snet-web (10.0.1.0/24)                    │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  VM 1   │  │  VM 2   │  │  VM 3   │            │   │
│  │  │10.0.1.4 │  │10.0.1.5 │  │10.0.1.6 │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  │                                                     │   │
│  │  Subnet: snet-api (10.0.2.0/24)                    │   │
│  │  ┌─────────┐  ┌─────────┐                         │   │
│  │  │ App Svc │  │ App Svc │ (VNet Integration)      │   │
│  │  └─────────┘  └─────────┘                         │   │
│  │                                                     │   │
│  │  Subnet: snet-db (10.0.3.0/24)                     │   │
│  │  ┌─────────┐                                       │   │
│  │  │Azure SQL│ (Private Endpoint)                   │   │
│  │  └─────────┘                                       │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Creating a Virtual Network

### Azure CLI

```bash
# Create VNet
az network vnet create \
  --name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --location eastus \
  --address-prefix 10.0.0.0/16

# Create subnets
az network vnet subnet create \
  --name snet-web \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --address-prefix 10.0.1.0/24

az network vnet subnet create \
  --name snet-api \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --address-prefix 10.0.2.0/24

az network vnet subnet create \
  --name snet-db \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --address-prefix 10.0.3.0/24

# List VNets
az network vnet list -o table

# Show VNet details
az network vnet show \
  --name vnet-myapp-prod \
  --resource-group rg-myapp-prod
```

### Bicep

```bicep
param vnetName string = 'vnet-myapp-prod'
param location string = resourceGroup().location

resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: 'snet-web'
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
      {
        name: 'snet-api'
        properties: {
          addressPrefix: '10.0.2.0/24'
          delegations: [
            {
              name: 'delegation'
              properties: {
                serviceName: 'Microsoft.Web/serverFarms'
              }
            }
          ]
        }
      }
      {
        name: 'snet-db'
        properties: {
          addressPrefix: '10.0.3.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
    ]
  }
}

output vnetId string = vnet.id
output webSubnetId string = vnet.properties.subnets[0].id
```

---

## Network Security Groups (NSG)

### What are NSGs?

NSGs filter network traffic to and from Azure resources using security rules.

```
┌─────────────────────────────────────────────────────────────┐
│              NETWORK SECURITY GROUP                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  NSG: nsg-web                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Inbound Rules:                                     │   │
│  │  ┌─────┬──────────┬──────┬────────┬───────────┐   │   │
│  │  │Prior│   Name   │ Port │ Source │  Action   │   │   │
│  │  ├─────┼──────────┼──────┼────────┼───────────┤   │   │
│  │  │ 100 │ AllowHTTP│  80  │  Any   │  Allow    │   │   │
│  │  │ 110 │AllowHTTPS│ 443  │  Any   │  Allow    │   │   │
│  │  │ 120 │ AllowSSH │  22  │10.0.0.0│  Allow    │   │   │
│  │  │65000│ DenyAll  │  *   │  Any   │  Deny     │   │   │
│  │  └─────┴──────────┴──────┴────────┴───────────┘   │   │
│  │                                                     │   │
│  │  Outbound Rules:                                    │   │
│  │  ┌─────┬──────────┬──────┬────────┬───────────┐   │   │
│  │  │65000│ AllowAll │  *   │  Any   │  Allow    │   │   │
│  │  └─────┴──────────┴──────┴────────┴───────────┘   │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create NSG

```bash
# Create NSG
az network nsg create \
  --name nsg-web \
  --resource-group rg-myapp-prod \
  --location eastus

# Add inbound rule - Allow HTTP
az network nsg rule create \
  --nsg-name nsg-web \
  --resource-group rg-myapp-prod \
  --name AllowHTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80

# Add inbound rule - Allow HTTPS
az network nsg rule create \
  --nsg-name nsg-web \
  --resource-group rg-myapp-prod \
  --name AllowHTTPS \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443

# Add inbound rule - Allow SSH from specific IP
az network nsg rule create \
  --nsg-name nsg-web \
  --resource-group rg-myapp-prod \
  --name AllowSSH \
  --priority 120 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.0.0.0/8 \
  --destination-port-ranges 22

# Associate NSG with subnet
az network vnet subnet update \
  --name snet-web \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --network-security-group nsg-web

# List NSG rules
az network nsg rule list \
  --nsg-name nsg-web \
  --resource-group rg-myapp-prod \
  -o table
```

### Bicep NSG

```bicep
resource nsgWeb 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-web'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTP'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '80'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
      {
        name: 'AllowHTTPS'
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}
```

---

## VNet Peering

Connect two VNets to allow resources to communicate.

```
┌─────────────────────────────────────────────────────────────┐
│                    VNET PEERING                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────┐     ┌─────────────────────┐       │
│  │ VNet: vnet-hub      │     │ VNet: vnet-spoke    │       │
│  │ (10.0.0.0/16)       │◄───▶│ (10.1.0.0/16)       │       │
│  │                     │     │                     │       │
│  │ ┌─────────────────┐ │     │ ┌─────────────────┐ │       │
│  │ │  Shared Svcs    │ │     │ │  App Workload   │ │       │
│  │ │  - DNS          │ │     │ │  - Web VMs      │ │       │
│  │ │  - Firewall     │ │     │ │  - API          │ │       │
│  │ └─────────────────┘ │     │ └─────────────────┘ │       │
│  └─────────────────────┘     └─────────────────────┘       │
│                                                             │
│  • Non-transitive (A↔B, B↔C doesn't mean A↔C)              │
│  • Can peer across regions (Global VNet Peering)           │
│  • Can peer across subscriptions                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create VNet Peering

```bash
# Create peering from hub to spoke
az network vnet peering create \
  --name hub-to-spoke \
  --resource-group rg-hub \
  --vnet-name vnet-hub \
  --remote-vnet /subscriptions/<sub-id>/resourceGroups/rg-spoke/providers/Microsoft.Network/virtualNetworks/vnet-spoke \
  --allow-vnet-access \
  --allow-forwarded-traffic

# Create peering from spoke to hub
az network vnet peering create \
  --name spoke-to-hub \
  --resource-group rg-spoke \
  --vnet-name vnet-spoke \
  --remote-vnet /subscriptions/<sub-id>/resourceGroups/rg-hub/providers/Microsoft.Network/virtualNetworks/vnet-hub \
  --allow-vnet-access \
  --allow-forwarded-traffic

# Check peering status
az network vnet peering list \
  --resource-group rg-hub \
  --vnet-name vnet-hub \
  -o table
```

---

## Service Endpoints

Allow Azure services to be accessed directly from VNet subnets.

```
┌─────────────────────────────────────────────────────────────┐
│                 SERVICE ENDPOINTS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Before: Traffic goes over internet                         │
│  ┌────────┐     Internet      ┌────────────┐               │
│  │  VNet  │ ─────────────────▶│ Azure SQL  │               │
│  └────────┘                   └────────────┘               │
│                                                             │
│  After: Traffic stays on Azure backbone                     │
│  ┌────────┐   Azure Backbone   ┌────────────┐              │
│  │  VNet  │ ─────────────────▶│ Azure SQL  │              │
│  │(Service│                   │(Firewall:  │              │
│  │Endpoint│                   │ VNet only) │              │
│  │enabled)│                   └────────────┘              │
│  └────────┘                                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Enable Service Endpoints

```bash
# Enable service endpoint for SQL on subnet
az network vnet subnet update \
  --name snet-api \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --service-endpoints Microsoft.Sql Microsoft.Storage Microsoft.KeyVault

# Add VNet rule to SQL server
az sql server vnet-rule create \
  --name allow-api-subnet \
  --server sql-myapp-prod \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-myapp-prod \
  --subnet snet-api
```

---

## Private Endpoints

Bring Azure services into your VNet with a private IP.

```
┌─────────────────────────────────────────────────────────────┐
│                  PRIVATE ENDPOINT                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  VNet: vnet-myapp-prod (10.0.0.0/16)                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Subnet: snet-db (10.0.3.0/24)                     │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │                                             │   │   │
│  │  │  Private Endpoint: pe-sql-myapp             │   │   │
│  │  │  IP: 10.0.3.5                               │   │   │
│  │  │         │                                   │   │   │
│  │  │         ▼                                   │   │   │
│  │  │  ┌─────────────────┐                       │   │   │
│  │  │  │   Azure SQL     │                       │   │   │
│  │  │  │sql-myapp.database                       │   │   │
│  │  │  │.windows.net     │                       │   │   │
│  │  │  │ (10.0.3.5)      │                       │   │   │
│  │  │  └─────────────────┘                       │   │   │
│  │  │                                             │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  DNS: sql-myapp.database.windows.net → 10.0.3.5            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create Private Endpoint

```bash
# Get SQL server resource ID
SQL_ID=$(az sql server show \
  --name sql-myapp-prod \
  --resource-group rg-myapp-prod \
  --query "id" -o tsv)

# Create private endpoint
az network private-endpoint create \
  --name pe-sql-myapp \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-myapp-prod \
  --subnet snet-db \
  --private-connection-resource-id $SQL_ID \
  --group-id sqlServer \
  --connection-name sql-connection

# Create private DNS zone
az network private-dns zone create \
  --name privatelink.database.windows.net \
  --resource-group rg-myapp-prod

# Link DNS zone to VNet
az network private-dns link vnet create \
  --name sql-dns-link \
  --resource-group rg-myapp-prod \
  --zone-name privatelink.database.windows.net \
  --virtual-network vnet-myapp-prod \
  --registration-enabled false

# Create DNS record
az network private-endpoint dns-zone-group create \
  --name sql-dns-group \
  --resource-group rg-myapp-prod \
  --endpoint-name pe-sql-myapp \
  --private-dns-zone privatelink.database.windows.net \
  --zone-name privatelink.database.windows.net
```

---

## App Service VNet Integration

Connect App Service to a VNet for outbound traffic.

```
┌─────────────────────────────────────────────────────────────┐
│            APP SERVICE VNET INTEGRATION                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  App Service: app-myapi-prod                        │   │
│  │  (Public inbound)                                   │   │
│  │         │                                           │   │
│  │         │ VNet Integration (Outbound)               │   │
│  │         ▼                                           │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  VNet: vnet-myapp-prod                      │   │   │
│  │  │  Subnet: snet-api-integration               │   │   │
│  │  │         │                                   │   │   │
│  │  │         ├──▶ Azure SQL (Private Endpoint)   │   │   │
│  │  │         ├──▶ Key Vault (Private Endpoint)   │   │   │
│  │  │         └──▶ Storage (Service Endpoint)     │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Configure VNet Integration

```bash
# Create dedicated subnet for App Service integration
az network vnet subnet create \
  --name snet-api-integration \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --address-prefix 10.0.4.0/24 \
  --delegations Microsoft.Web/serverFarms

# Enable VNet integration for App Service
az webapp vnet-integration add \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --vnet vnet-myapp-prod \
  --subnet snet-api-integration

# Verify integration
az webapp vnet-integration list \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod
```

---

## Load Balancer vs Application Gateway

```
┌─────────────────────────────────────────────────────────────┐
│         LOAD BALANCER vs APPLICATION GATEWAY                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Azure Load Balancer (Layer 4)                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • TCP/UDP traffic                                  │   │
│  │  • High performance, low latency                    │   │
│  │  • Zone redundant                                   │   │
│  │  • Internal or Public                               │   │
│  │  Use for: Non-HTTP, internal LB, high throughput   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Application Gateway (Layer 7)                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • HTTP/HTTPS traffic                               │   │
│  │  • URL-based routing                                │   │
│  │  • SSL termination                                  │   │
│  │  • WAF (Web Application Firewall)                   │   │
│  │  • Cookie-based session affinity                    │   │
│  │  Use for: Web apps, APIs, microservices            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create Application Gateway

```bash
# Create public IP for App Gateway
az network public-ip create \
  --name pip-appgw \
  --resource-group rg-myapp-prod \
  --sku Standard \
  --allocation-method Static

# Create App Gateway subnet
az network vnet subnet create \
  --name snet-appgw \
  --vnet-name vnet-myapp-prod \
  --resource-group rg-myapp-prod \
  --address-prefix 10.0.5.0/24

# Create Application Gateway
az network application-gateway create \
  --name appgw-myapp \
  --resource-group rg-myapp-prod \
  --location eastus \
  --vnet-name vnet-myapp-prod \
  --subnet snet-appgw \
  --public-ip-address pip-appgw \
  --sku Standard_v2 \
  --capacity 2 \
  --http-settings-cookie-based-affinity Disabled \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --servers app-myapi-prod.azurewebsites.net
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **VNet** | Private network in Azure |
| **Subnet** | IP address range within VNet |
| **NSG** | Firewall rules for subnets/NICs |
| **VNet Peering** | Connect VNets together |
| **Service Endpoint** | Direct access to Azure services |
| **Private Endpoint** | Private IP for Azure services |
| **VNet Integration** | App Service outbound to VNet |
| **Application Gateway** | Layer 7 load balancer + WAF |

### Quick Commands

```bash
# VNet
az network vnet create -n vnet -g rg --address-prefix 10.0.0.0/16
az network vnet subnet create -n subnet --vnet-name vnet -g rg --address-prefix 10.0.1.0/24

# NSG
az network nsg create -n nsg -g rg
az network nsg rule create --nsg-name nsg -g rg -n AllowHTTP --priority 100 --destination-port-ranges 80

# Private Endpoint
az network private-endpoint create -n pe -g rg --vnet-name vnet --subnet subnet --connection-name conn --private-connection-resource-id <resource-id> --group-id <group>
```
