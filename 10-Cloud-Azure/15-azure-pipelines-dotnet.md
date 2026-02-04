# Azure Pipelines for .NET 10

## Pipeline Overview

Complete CI/CD pipeline for .NET 10 applications with build, test, and deployment to Azure App Service.

```
┌─────────────────────────────────────────────────────────────┐
│              .NET 10 CI/CD PIPELINE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│  │  Code   │───▶│  Build  │───▶│  Test   │───▶│ Publish │ │
│  │  Push   │    │         │    │         │    │         │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘ │
│                                                     │       │
│                              ┌──────────────────────┘       │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    DEPLOY                           │   │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐         │   │
│  │  │   Dev   │───▶│ Staging │───▶│  Prod   │         │   │
│  │  │(Auto)   │    │(Auto)   │    │(Approval)│        │   │
│  │  └─────────┘    └─────────┘    └─────────┘         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Basic .NET Pipeline

### Simple Build and Deploy

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
  dotnetVersion: '10.x'

steps:
  - task: UseDotNet@2
    displayName: 'Install .NET 10'
    inputs:
      version: $(dotnetVersion)

  - task: DotNetCoreCLI@2
    displayName: 'Restore'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'

  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      projects: '**/*.csproj'
      arguments: '--configuration $(buildConfiguration) --no-restore'

  - task: DotNetCoreCLI@2
    displayName: 'Test'
    inputs:
      command: 'test'
      projects: '**/*Tests.csproj'
      arguments: '--configuration $(buildConfiguration) --no-build'

  - task: DotNetCoreCLI@2
    displayName: 'Publish'
    inputs:
      command: 'publish'
      projects: 'src/MyApp.Api/MyApp.Api.csproj'
      arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
      publishWebProjects: false

  - task: AzureWebApp@1
    displayName: 'Deploy to Azure'
    inputs:
      azureSubscription: 'Azure-ServiceConnection'
      appType: 'webAppLinux'
      appName: 'app-myapi-dev'
      package: '$(Build.ArtifactStagingDirectory)/**/*.zip'
```

---

## Multi-Stage Pipeline

### Complete Pipeline with Environments

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - src/**
    exclude:
      - '**/*.md'

pr:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '10.x'
  projectPath: 'src/MyApp.Api/MyApp.Api.csproj'
  testProjectPath: 'tests/MyApp.Api.Tests/MyApp.Api.Tests.csproj'

stages:
  # ========== BUILD STAGE ==========
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: Build
        displayName: 'Build Job'
        steps:
          - task: UseDotNet@2
            displayName: 'Install .NET SDK'
            inputs:
              version: $(dotnetVersion)

          - task: DotNetCoreCLI@2
            displayName: 'Restore packages'
            inputs:
              command: 'restore'
              projects: '**/*.csproj'
              feedsToUse: 'select'

          - task: DotNetCoreCLI@2
            displayName: 'Build solution'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Run tests'
            inputs:
              command: 'test'
              projects: '$(testProjectPath)'
              arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage" --results-directory $(Agent.TempDirectory)'
              publishTestResults: true

          - task: PublishCodeCoverageResults@2
            displayName: 'Publish code coverage'
            inputs:
              summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'

          - task: DotNetCoreCLI@2
            displayName: 'Publish application'
            inputs:
              command: 'publish'
              projects: '$(projectPath)'
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/app'
              publishWebProjects: false
              zipAfterPublish: true

          - publish: $(Build.ArtifactStagingDirectory)/app
            displayName: 'Upload artifact'
            artifact: drop

  # ========== DEPLOY TO DEV ==========
  - stage: DeployDev
    displayName: 'Deploy to Dev'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployDev
        displayName: 'Deploy to Dev Environment'
        environment: 'development'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy to Dev App Service'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    appType: 'webAppLinux'
                    appName: 'app-myapi-dev'
                    package: '$(Pipeline.Workspace)/drop/*.zip'
                    runtimeStack: 'DOTNETCORE|8.0'

  # ========== DEPLOY TO STAGING ==========
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployStaging
        displayName: 'Deploy to Staging'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy to Staging Slot'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    appType: 'webAppLinux'
                    appName: 'app-myapi-prod'
                    deployToSlotOrASE: true
                    resourceGroupName: 'rg-myapp-prod'
                    slotName: 'staging'
                    package: '$(Pipeline.Workspace)/drop/*.zip'

  # ========== DEPLOY TO PRODUCTION ==========
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployProduction
        displayName: 'Deploy to Production'
        environment: 'production'  # Requires approval
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureAppServiceManage@0
                  displayName: 'Swap Staging to Production'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    action: 'Swap Slots'
                    webAppName: 'app-myapi-prod'
                    resourceGroupName: 'rg-myapp-prod'
                    sourceSlot: 'staging'
                    targetSlot: 'production'
```

---

## Pipeline Templates

### Build Template

```yaml
# templates/dotnet-build.yml
parameters:
  - name: buildConfiguration
    type: string
    default: 'Release'
  - name: projectPath
    type: string

steps:
  - task: UseDotNet@2
    displayName: 'Install .NET SDK'
    inputs:
      version: '10.x'

  - task: DotNetCoreCLI@2
    displayName: 'Restore'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'

  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      projects: '${{ parameters.projectPath }}'
      arguments: '--configuration ${{ parameters.buildConfiguration }} --no-restore'

  - task: DotNetCoreCLI@2
    displayName: 'Publish'
    inputs:
      command: 'publish'
      projects: '${{ parameters.projectPath }}'
      arguments: '--configuration ${{ parameters.buildConfiguration }} --output $(Build.ArtifactStagingDirectory)'
      publishWebProjects: false
      zipAfterPublish: true

  - publish: $(Build.ArtifactStagingDirectory)
    artifact: drop
```

### Deploy Template

```yaml
# templates/deploy-webapp.yml
parameters:
  - name: environment
    type: string
  - name: appName
    type: string
  - name: serviceConnection
    type: string
  - name: slot
    type: string
    default: ''

jobs:
  - deployment: Deploy
    displayName: 'Deploy to ${{ parameters.environment }}'
    environment: '${{ parameters.environment }}'
    strategy:
      runOnce:
        deploy:
          steps:
            - task: AzureWebApp@1
              displayName: 'Deploy to App Service'
              inputs:
                azureSubscription: '${{ parameters.serviceConnection }}'
                appType: 'webAppLinux'
                appName: '${{ parameters.appName }}'
                ${{ if ne(parameters.slot, '') }}:
                  deployToSlotOrASE: true
                  slotName: '${{ parameters.slot }}'
                package: '$(Pipeline.Workspace)/drop/*.zip'
```

### Using Templates

```yaml
# azure-pipelines.yml
trigger:
  - main

stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - template: templates/dotnet-build.yml
            parameters:
              buildConfiguration: 'Release'
              projectPath: 'src/MyApp.Api/MyApp.Api.csproj'

  - stage: DeployDev
    dependsOn: Build
    jobs:
      - template: templates/deploy-webapp.yml
        parameters:
          environment: 'development'
          appName: 'app-myapi-dev'
          serviceConnection: 'Azure-ServiceConnection'

  - stage: DeployProd
    dependsOn: DeployDev
    jobs:
      - template: templates/deploy-webapp.yml
        parameters:
          environment: 'production'
          appName: 'app-myapi-prod'
          serviceConnection: 'Azure-ServiceConnection'
          slot: 'staging'
```

---

## Database Migrations

### Run EF Core Migrations in Pipeline

```yaml
# Database migration job
- job: RunMigrations
  displayName: 'Run Database Migrations'
  pool:
    vmImage: 'ubuntu-latest'
  steps:
    - task: UseDotNet@2
      inputs:
        version: '10.x'

    - script: |
        dotnet tool install --global dotnet-ef
        dotnet ef database update --project src/MyApp.Api/MyApp.Api.csproj
      displayName: 'Run EF Migrations'
      env:
        ConnectionStrings__DefaultConnection: $(DatabaseConnectionString)
```

### Generate Migration Script

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Generate Migration Script'
  inputs:
    command: 'custom'
    custom: 'ef'
    arguments: 'migrations script --idempotent --output $(Build.ArtifactStagingDirectory)/migration.sql --project src/MyApp.Api/MyApp.Api.csproj'

- task: SqlAzureDacpacDeployment@1
  displayName: 'Run SQL Script'
  inputs:
    azureSubscription: 'Azure-ServiceConnection'
    ServerName: 'sql-myapp-prod.database.windows.net'
    DatabaseName: 'myappdb'
    SqlUsername: '$(SqlUsername)'
    SqlPassword: '$(SqlPassword)'
    deployType: 'SqlTask'
    SqlFile: '$(Build.ArtifactStagingDirectory)/migration.sql'
```

---

## Docker Build and Push

### Build and Push to ACR

```yaml
variables:
  acrName: 'acrmyapp'
  imageName: 'myapp-api'

stages:
  - stage: BuildDocker
    jobs:
      - job: BuildPush
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: Docker@2
            displayName: 'Build and Push'
            inputs:
              containerRegistry: 'acr-service-connection'
              repository: '$(imageName)'
              command: 'buildAndPush'
              Dockerfile: 'src/MyApp.Api/Dockerfile'
              buildContext: '.'
              tags: |
                $(Build.BuildId)
                latest

  - stage: Deploy
    jobs:
      - deployment: DeployContainer
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebAppContainer@1
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    appName: 'app-myapi-prod'
                    containers: '$(acrName).azurecr.io/$(imageName):$(Build.BuildId)'
```

---

## Security Scanning

### Add Security Scans

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Security Scan - dotnet list package'
  inputs:
    command: 'custom'
    custom: 'list'
    arguments: 'package --vulnerable --include-transitive'
  continueOnError: true

# Or use dedicated security tools
- task: CredScan@3
  displayName: 'Credential Scanner'
  inputs:
    toolMajorVersion: 'V2'

- task: SdtReport@2
  displayName: 'Security Report'
  inputs:
    GdnExportAllTools: true
```

---

## Variable Groups and Secrets

### Using Variable Groups

```yaml
variables:
  - group: 'Production-Secrets'  # Linked to Key Vault
  - name: buildConfiguration
    value: 'Release'

steps:
  - script: |
      echo "Using secret: $(DatabasePassword)"
    env:
      DB_PASSWORD: $(DatabasePassword)  # Mask in logs
```

### Key Vault Integration

```yaml
- task: AzureKeyVault@2
  displayName: 'Get secrets from Key Vault'
  inputs:
    azureSubscription: 'Azure-ServiceConnection'
    keyVaultName: 'kv-myapp-prod'
    secretsFilter: 'DatabasePassword,ApiKey'
    runAsPreJob: true
```

---

## Summary

| Stage | Tasks |
|-------|-------|
| **Build** | Restore, Build, Test, Publish |
| **Deploy Dev** | Auto-deploy on develop branch |
| **Deploy Staging** | Deploy to staging slot |
| **Deploy Prod** | Swap slots (with approval) |

### Pipeline Best Practices

```yaml
# ✓ Use templates for reusability
# ✓ Use variable groups for secrets
# ✓ Use environments with approvals
# ✓ Run tests before deployment
# ✓ Use deployment slots for zero-downtime
# ✓ Include security scanning
# ✓ Publish code coverage results
```
