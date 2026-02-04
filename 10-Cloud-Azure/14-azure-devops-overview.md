# Azure DevOps Overview

## What is Azure DevOps?

Azure DevOps is a set of development tools and services for planning, developing, testing, and deploying applications.

```
┌─────────────────────────────────────────────────────────────┐
│                    AZURE DEVOPS SERVICES                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │ Boards  │  │  Repos  │  │Pipelines│            │   │
│  │  │         │  │         │  │         │            │   │
│  │  │ • Work  │  │ • Git   │  │ • Build │            │   │
│  │  │   Items │  │ • TFVC  │  │ • Deploy│            │   │
│  │  │ • Sprints│ │ • PRs   │  │ • YAML  │            │   │
│  │  │ • Kanban│  │ • Branch│  │ • Release│           │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  │                                                     │   │
│  │  ┌─────────┐  ┌─────────┐                         │   │
│  │  │Test Plans│ │Artifacts│                         │   │
│  │  │         │  │         │                         │   │
│  │  │ • Manual│  │ • NuGet │                         │   │
│  │  │ • Auto  │  │ • npm   │                         │   │
│  │  │ • Load  │  │ • Maven │                         │   │
│  │  └─────────┘  └─────────┘                         │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  URL: https://dev.azure.com/{organization}                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Getting Started

### Create Organization and Project

```bash
# Azure DevOps is primarily managed via web UI
# URL: https://dev.azure.com

# 1. Sign in with Microsoft account
# 2. Create new organization
# 3. Create new project

# Or use Azure CLI with DevOps extension
az extension add --name azure-devops

# Login
az devops login

# Configure defaults
az devops configure --defaults organization=https://dev.azure.com/myorg project=MyProject

# Create project
az devops project create \
  --name "MyApp" \
  --description "My application project" \
  --visibility private
```

---

## Azure Repos

### Create and Clone Repository

```bash
# List repos
az repos list -o table

# Create repo
az repos create --name "my-app" --project "MyProject"

# Clone repo
git clone https://dev.azure.com/myorg/MyProject/_git/my-app

# Configure Git credentials
git config --global credential.helper manager
```

### Branch Policies

```bash
# Create branch policy (require PR)
az repos policy approver-count create \
  --branch main \
  --repository-id <repo-id> \
  --minimum-approver-count 1 \
  --creator-vote-counts false \
  --enabled true

# Require build validation
az repos policy build create \
  --branch main \
  --repository-id <repo-id> \
  --build-definition-id <pipeline-id> \
  --display-name "Build Validation" \
  --enabled true \
  --blocking true
```

---

## Azure Boards

### Work Item Types

```
┌─────────────────────────────────────────────────────────────┐
│                    WORK ITEM HIERARCHY                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Agile Process:                                             │
│  ┌─────────────┐                                           │
│  │    Epic     │  Large feature/initiative                 │
│  └──────┬──────┘                                           │
│         │                                                   │
│  ┌──────▼──────┐                                           │
│  │   Feature   │  Customer-facing functionality            │
│  └──────┬──────┘                                           │
│         │                                                   │
│  ┌──────▼──────┐                                           │
│  │ User Story  │  Valuable functionality from user POV     │
│  └──────┬──────┘                                           │
│         │                                                   │
│  ┌──────▼──────┐                                           │
│  │    Task     │  Development work                         │
│  └─────────────┘                                           │
│                                                             │
│  Other types: Bug, Issue, Test Case                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Create Work Items

```bash
# Create work item
az boards work-item create \
  --title "Implement user authentication" \
  --type "User Story" \
  --assigned-to "user@company.com" \
  --description "As a user, I want to log in securely"

# Create task under user story
az boards work-item create \
  --title "Create login API endpoint" \
  --type "Task" \
  --parent <user-story-id>

# Update work item
az boards work-item update \
  --id 123 \
  --state "Active"

# List work items
az boards query --wiql "SELECT [ID], [Title], [State] FROM WorkItems WHERE [State] = 'Active'"
```

---

## Azure Pipelines Overview

### Pipeline Types

| Type | Use Case | Configuration |
|------|----------|---------------|
| **YAML Pipelines** | CI/CD as code | `azure-pipelines.yml` |
| **Classic Build** | GUI-based builds | Web UI |
| **Classic Release** | GUI-based deployments | Web UI |

### Basic YAML Pipeline Structure

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - src/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - task: UseDotNet@2
            inputs:
              version: '10.x'

          - script: dotnet build --configuration $(buildConfiguration)
            displayName: 'Build'

  - stage: Deploy
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    appName: 'app-myapi-prod'
```

---

## Service Connections

### Create Azure Service Connection

```bash
# Create service connection for Azure Resource Manager
# (Usually done via UI: Project Settings > Service Connections)

# Using CLI (requires PAT token)
az devops service-endpoint azurerm create \
  --name "Azure-Production" \
  --azure-rm-service-principal-id <sp-app-id> \
  --azure-rm-subscription-id <subscription-id> \
  --azure-rm-subscription-name "Production" \
  --azure-rm-tenant-id <tenant-id>

# List service connections
az devops service-endpoint list -o table
```

### Create Service Principal for DevOps

```bash
# Create service principal
az ad sp create-for-rbac \
  --name "sp-azure-devops" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>

# Output values needed for service connection:
# - Application (client) ID
# - Client secret
# - Directory (tenant) ID
```

---

## Azure Artifacts

### Create Feed

```bash
# Create feed
az artifacts feed create \
  --name "my-packages" \
  --description "Internal packages"

# List feeds
az artifacts feed list -o table
```

### Publish NuGet Package

```yaml
# In pipeline
- task: NuGetCommand@2
  inputs:
    command: 'push'
    packagesToPush: '$(Build.ArtifactStagingDirectory)/**/*.nupkg'
    nuGetFeedType: 'internal'
    publishVstsFeed: 'my-packages'
```

### Configure nuget.config

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="MyFeed" value="https://pkgs.dev.azure.com/myorg/_packaging/my-packages/nuget/v3/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <MyFeed>
      <add key="Username" value="any" />
      <add key="ClearTextPassword" value="$(SYSTEM_ACCESSTOKEN)" />
    </MyFeed>
  </packageSourceCredentials>
</configuration>
```

---

## Environments

### Create Deployment Environments

```yaml
# Define environments with approvals and checks
# Configure in Azure DevOps UI: Pipelines > Environments

# Use in pipeline
stages:
  - stage: DeployDev
    jobs:
      - deployment: Deploy
        environment: 'development'  # No approval

  - stage: DeployProd
    jobs:
      - deployment: Deploy
        environment: 'production'  # Requires approval
```

### Environment Features

```
┌─────────────────────────────────────────────────────────────┐
│                  ENVIRONMENTS                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  development                    production                  │
│  ┌─────────────────────┐       ┌─────────────────────┐     │
│  │ Approvals: None     │       │ Approvals: Required │     │
│  │ Checks: None        │       │ Checks:             │     │
│  │                     │       │  - Business hours   │     │
│  │ Resources:          │       │  - Branch policy    │     │
│  │  - Kubernetes       │       │                     │     │
│  │  - VM               │       │ Resources:          │     │
│  │                     │       │  - Kubernetes       │     │
│  └─────────────────────┘       │  - VM               │     │
│                                └─────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Variable Groups and Secrets

### Create Variable Group

```bash
# Create variable group
az pipelines variable-group create \
  --name "Production-Variables" \
  --variables \
    ENVIRONMENT=production \
    API_URL=https://api.myapp.com

# Add secret variable
az pipelines variable-group variable create \
  --group-id <group-id> \
  --name "DATABASE_PASSWORD" \
  --value "secret" \
  --secret true

# Link to Key Vault
# Done via UI: Pipelines > Library > Variable Groups > Link secrets from Azure Key Vault
```

### Use in Pipeline

```yaml
variables:
  - group: Production-Variables  # Variable group
  - name: localVar
    value: 'local-value'

steps:
  - script: |
      echo "Environment: $(ENVIRONMENT)"
      echo "API URL: $(API_URL)"
```

---

## Integration with GitHub

### GitHub + Azure Pipelines

```yaml
# azure-pipelines.yml in GitHub repo
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - checkout: self

  - script: echo "Building from GitHub repo"
```

### Setup Steps

1. Go to Azure DevOps > Pipelines > Create Pipeline
2. Select "GitHub" as source
3. Authorize Azure Pipelines app in GitHub
4. Select repository
5. Configure pipeline

---

## Azure DevOps CLI Reference

```bash
# Configure
az devops configure --defaults organization=https://dev.azure.com/myorg project=MyProject

# Projects
az devops project list -o table
az devops project create --name "NewProject"

# Repos
az repos list -o table
az repos create --name "new-repo"

# Pipelines
az pipelines list -o table
az pipelines run --name "My-Pipeline"
az pipelines build list -o table

# Work items
az boards work-item create --title "Task" --type "Task"
az boards work-item show --id 123

# Artifacts
az artifacts feed list -o table
```

---

## Summary

| Service | Purpose |
|---------|---------|
| **Boards** | Work tracking, sprints, Kanban |
| **Repos** | Git repositories, PRs, branch policies |
| **Pipelines** | CI/CD, build and deploy |
| **Test Plans** | Manual and automated testing |
| **Artifacts** | Package management (NuGet, npm) |

### Getting Started Checklist

1. Create organization at dev.azure.com
2. Create project
3. Set up repository (or connect to GitHub)
4. Create service connection to Azure
5. Create first pipeline
6. Configure environments with approvals
7. Set up variable groups for secrets
