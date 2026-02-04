# Introduction to Azure Cloud

## What is Cloud Computing?

Cloud computing delivers computing services over the internet, including servers, storage, databases, networking, software, and analytics.

```
┌─────────────────────────────────────────────────────────────┐
│                    CLOUD COMPUTING                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Traditional IT              Cloud Computing                │
│  ┌─────────────┐            ┌─────────────┐                │
│  │ Buy Servers │            │ Rent as     │                │
│  │ Own Hardware│            │ Needed      │                │
│  │ Maintain    │            │ Pay-per-Use │                │
│  │ Scale Slowly│            │ Scale Fast  │                │
│  └─────────────┘            └─────────────┘                │
│                                                             │
│  Benefits:                                                  │
│  • No upfront costs    • Global reach                      │
│  • Pay for what you use • High availability                │
│  • Scale on demand     • Security built-in                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Cloud Service Models

### IaaS, PaaS, SaaS

```
┌─────────────────────────────────────────────────────────────┐
│              CLOUD SERVICE MODELS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  On-Premises    IaaS         PaaS         SaaS             │
│  ┌─────────┐   ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ App     │   │ App     │  │ App     │  │░░░░░░░░░│      │
│  │ Data    │   │ Data    │  │ Data    │  │░░░░░░░░░│      │
│  │ Runtime │   │ Runtime │  │░░░░░░░░░│  │░░░░░░░░░│      │
│  │ OS      │   │ OS      │  │░░░░░░░░░│  │░░░░░░░░░│      │
│  │ Virtual │   │░░░░░░░░░│  │░░░░░░░░░│  │░░░░░░░░░│      │
│  │ Servers │   │░░░░░░░░░│  │░░░░░░░░░│  │░░░░░░░░░│      │
│  │ Storage │   │░░░░░░░░░│  │░░░░░░░░░│  │░░░░░░░░░│      │
│  │ Network │   │░░░░░░░░░│  │░░░░░░░░░│  │░░░░░░░░░│      │
│  └─────────┘   └─────────┘  └─────────┘  └─────────┘      │
│                                                             │
│  █ You Manage    ░ Cloud Provider Manages                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Model | You Manage | Provider Manages | Azure Examples |
|-------|------------|------------------|----------------|
| **IaaS** | Apps, Data, Runtime, OS | Hardware, Network | Virtual Machines, VNet |
| **PaaS** | Apps, Data | Everything else | App Service, Azure SQL |
| **SaaS** | Nothing (just use it) | Everything | Microsoft 365, Dynamics |

---

## What is Microsoft Azure?

Microsoft Azure is a cloud computing platform offering 200+ services for building, deploying, and managing applications.

### Azure for DevOps

```
┌─────────────────────────────────────────────────────────────┐
│                   AZURE FOR DEVOPS                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│  │  Code   │───▶│  Build  │───▶│  Test   │───▶│ Deploy  │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘ │
│       │              │              │              │        │
│       ▼              ▼              ▼              ▼        │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│  │ Azure   │    │ Azure   │    │ Azure   │    │ App     │ │
│  │ Repos   │    │Pipelines│    │ Test    │    │ Service │ │
│  │ GitHub  │    │ Actions │    │ Plans   │    │ AKS     │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘ │
│                                                             │
│  Monitor & Operate:                                         │
│  • Application Insights  • Log Analytics  • Azure Monitor  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Azure Services for DevOps

| Category | Services | Use Case |
|----------|----------|----------|
| **Compute** | App Service, AKS, VMs | Host .NET/Angular apps |
| **Containers** | ACR, ACI, AKS | Docker images & orchestration |
| **Databases** | Azure SQL, Cosmos DB | Application data |
| **DevOps** | Azure DevOps, GitHub | CI/CD pipelines |
| **Storage** | Blob, Files | Static assets, backups |
| **Security** | Key Vault, Defender | Secrets, security |
| **Monitoring** | App Insights, Monitor | Observability |

---

## Azure Global Infrastructure

### Regions and Availability Zones

```
┌─────────────────────────────────────────────────────────────┐
│              AZURE GLOBAL INFRASTRUCTURE                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Geography (e.g., United States)                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Region: East US          Region: West US           │   │
│  │  ┌─────────────────┐     ┌─────────────────┐       │   │
│  │  │ Zone 1  Zone 2  │     │ Zone 1  Zone 2  │       │   │
│  │  │ ┌────┐  ┌────┐  │     │ ┌────┐  ┌────┐  │       │   │
│  │  │ │ DC │  │ DC │  │     │ │ DC │  │ DC │  │       │   │
│  │  │ └────┘  └────┘  │     │ └────┘  └────┘  │       │   │
│  │  │      Zone 3     │     │      Zone 3     │       │   │
│  │  │      ┌────┐     │     │      ┌────┐     │       │   │
│  │  │      │ DC │     │     │      │ DC │     │       │   │
│  │  │      └────┘     │     │      └────┘     │       │   │
│  │  └─────────────────┘     └─────────────────┘       │   │
│  │          ▲                       ▲                  │   │
│  │          └───── Region Pair ─────┘                  │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Concept | Description |
|---------|-------------|
| **Geography** | Country/region with data residency boundaries |
| **Region** | Set of datacenters (60+ regions worldwide) |
| **Availability Zone** | Physically separate datacenter in a region |
| **Region Pair** | Two regions paired for disaster recovery |

### Choosing a Region

```bash
# Factors to consider:
# 1. Latency - closest to users
# 2. Compliance - data residency requirements
# 3. Service availability - not all services in all regions
# 4. Cost - prices vary by region
# 5. Disaster recovery - use region pairs
```

---

## Azure vs AWS vs GCP

### Service Comparison

| Category | Azure | AWS | GCP |
|----------|-------|-----|-----|
| **Compute** | Virtual Machines | EC2 | Compute Engine |
| **Containers** | AKS | EKS | GKE |
| **Serverless** | Functions | Lambda | Cloud Functions |
| **PaaS** | App Service | Elastic Beanstalk | App Engine |
| **SQL Database** | Azure SQL | RDS | Cloud SQL |
| **NoSQL** | Cosmos DB | DynamoDB | Firestore |
| **Object Storage** | Blob Storage | S3 | Cloud Storage |
| **CDN** | Azure CDN | CloudFront | Cloud CDN |
| **Container Registry** | ACR | ECR | GCR/Artifact Registry |
| **DevOps** | Azure DevOps | CodePipeline | Cloud Build |
| **Monitoring** | Azure Monitor | CloudWatch | Cloud Monitoring |
| **Secrets** | Key Vault | Secrets Manager | Secret Manager |

### Why Azure for .NET?

```
┌─────────────────────────────────────────────────────────────┐
│              AZURE + .NET INTEGRATION                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  • First-class .NET support from Microsoft                  │
│  • Visual Studio & VS Code integration                      │
│  • Azure SDK for .NET                                       │
│  • App Service optimized for .NET 10                        │
│  • Application Insights for .NET                            │
│  • Entity Framework + Azure SQL                             │
│  • GitHub Copilot + Azure integration                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Azure Account Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│              AZURE ACCOUNT STRUCTURE                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Azure AD Tenant (Organization)                             │
│  └── Management Groups (Optional)                           │
│      └── Subscriptions (Billing boundary)                   │
│          └── Resource Groups (Logical containers)           │
│              └── Resources (VMs, DBs, Apps, etc.)          │
│                                                             │
│  Example:                                                   │
│  ┌─────────────────────────────────────────┐               │
│  │ Tenant: contoso.onmicrosoft.com         │               │
│  │ ├── Management Group: Production        │               │
│  │ │   ├── Subscription: Prod-East         │               │
│  │ │   │   ├── RG: rg-webapp-prod          │               │
│  │ │   │   │   ├── App Service             │               │
│  │ │   │   │   ├── Azure SQL               │               │
│  │ │   │   │   └── Key Vault               │               │
│  │ │   │   └── RG: rg-infra-prod           │               │
│  │ │   └── Subscription: Prod-West         │               │
│  │ └── Management Group: Development       │               │
│  │     └── Subscription: Dev               │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## DevOps Workflow with Azure

### Full Stack .NET 10 + Angular Deployment

```
┌─────────────────────────────────────────────────────────────┐
│           FULL STACK DEPLOYMENT ARCHITECTURE                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Developer                                                  │
│      │                                                      │
│      ▼                                                      │
│  ┌─────────┐     ┌──────────────┐     ┌───────────────┐   │
│  │ GitHub  │────▶│Azure Pipelines│────▶│   Deploy      │   │
│  │ Repo    │     │GitHub Actions │     │               │   │
│  └─────────┘     └──────────────┘     └───────────────┘   │
│                         │                     │             │
│              ┌──────────┴──────────┐          │             │
│              ▼                     ▼          ▼             │
│        ┌──────────┐         ┌──────────┐ ┌─────────┐       │
│        │ .NET 10  │         │ Angular  │ │ Azure   │       │
│        │ Build    │         │ Build    │ │ SQL     │       │
│        └──────────┘         └──────────┘ └─────────┘       │
│              │                     │                        │
│              ▼                     ▼                        │
│        ┌──────────┐         ┌──────────┐                   │
│        │   App    │         │  Static  │                   │
│        │ Service  │◀────────│ Web Apps │                   │
│        │ .NET API │  API    │ Angular  │                   │
│        └──────────┘  Calls  └──────────┘                   │
│              │                                              │
│              ▼                                              │
│        ┌──────────┐                                        │
│        │Key Vault │ ◀── Managed Identity                   │
│        │ Secrets  │                                        │
│        └──────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Cloud Computing** | On-demand IT resources over internet |
| **IaaS/PaaS/SaaS** | Different levels of cloud management |
| **Azure** | Microsoft's cloud platform |
| **Region** | Geographic location with datacenters |
| **Availability Zone** | Independent datacenter within region |
| **Subscription** | Billing and access boundary |
| **Resource Group** | Logical container for resources |

### Next Steps

1. Set up Azure account and subscription
2. Install Azure CLI
3. Learn Azure Resource Manager
4. Deploy your first .NET 10 application
