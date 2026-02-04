# Azure DevOps

## What is Azure DevOps?

Azure DevOps is Microsoft's cloud-based DevOps platform providing development collaboration tools.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure DevOps Services                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│   │ Azure Boards │  │ Azure Repos  │  │ Azure        │         │
│   │              │  │              │  │ Pipelines    │         │
│   │ Work Items   │  │ Git Repos    │  │              │         │
│   │ Sprints      │  │ Pull Requests│  │ CI/CD        │         │
│   │ Backlogs     │  │ Branches     │  │ Releases     │         │
│   └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
│   ┌──────────────┐  ┌──────────────┐                           │
│   │ Azure Test   │  │ Azure        │                           │
│   │ Plans        │  │ Artifacts    │                           │
│   │              │  │              │                           │
│   │ Test Cases   │  │ Package Mgmt │                           │
│   │ Test Runs    │  │ npm, NuGet   │                           │
│   └──────────────┘  └──────────────┘                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Azure Pipelines Overview

### YAML Pipeline Structure

```yaml
# azure-pipelines.yml
trigger:
  - main
  - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'build'
              projects: '**/*.csproj'

  - stage: Test
    dependsOn: Build
    jobs:
      - job: TestJob
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'test'

  - stage: Deploy
    dependsOn: Test
    jobs:
      - job: DeployJob
        steps:
          - script: echo "Deploying..."
```

## Triggers

### Branch Triggers

```yaml
trigger:
  branches:
    include:
      - main
      - release/*
    exclude:
      - feature/experimental

  paths:
    include:
      - src/*
    exclude:
      - docs/*

  tags:
    include:
      - v*
```

### Pull Request Triggers

```yaml
pr:
  branches:
    include:
      - main
      - develop
  paths:
    exclude:
      - README.md
  drafts: false
```

### Scheduled Triggers

```yaml
schedules:
  - cron: "0 2 * * *"
    displayName: "Nightly build"
    branches:
      include:
        - main
    always: true
```

## Pools and Agents

### Microsoft-Hosted Agents

```yaml
pool:
  vmImage: 'ubuntu-latest'
  # Options: ubuntu-latest, windows-latest, macos-latest
  # ubuntu-22.04, windows-2022, macos-13
```

### Self-Hosted Agents

```yaml
pool:
  name: 'MyAgentPool'
  demands:
    - docker
    - Agent.OS -equals Linux
```

### Container Jobs

```yaml
pool:
  vmImage: 'ubuntu-latest'

container: node:20

steps:
  - script: npm install
  - script: npm test
```

## Variables

### Pipeline Variables

```yaml
variables:
  # Simple variable
  buildConfiguration: 'Release'

  # Variable group reference
  - group: 'my-variable-group'

  # Template variable
  - template: variables/common.yml

  # Runtime expression
  - name: version
    value: $[counter(variables['Build.SourceBranchName'], 1)]
```

### Variable Groups

```yaml
# Reference variable group
variables:
  - group: 'production-secrets'

steps:
  - script: echo $(DatabasePassword)
```

### Runtime Expressions

```yaml
variables:
  environment: ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
    'production'
  ${{ else }}:
    'development'

steps:
  - script: echo "Deploying to $(environment)"
```

## Stages and Jobs

### Multi-Stage Pipeline

```yaml
stages:
  - stage: Build
    displayName: 'Build Stage'
    jobs:
      - job: BuildJob
        displayName: 'Build'
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'build'

  - stage: Test
    displayName: 'Test Stage'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - job: UnitTests
        steps:
          - script: npm test

      - job: IntegrationTests
        dependsOn: UnitTests
        steps:
          - script: npm run test:integration

  - stage: DeployStaging
    dependsOn: Test
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToStaging
        environment: 'staging'
```

### Parallel Jobs

```yaml
jobs:
  - job: TestLinux
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - script: npm test

  - job: TestWindows
    pool:
      vmImage: 'windows-latest'
    steps:
      - script: npm test

  - job: TestMac
    pool:
      vmImage: 'macos-latest'
    steps:
      - script: npm test
```

### Job Dependencies

```yaml
jobs:
  - job: A
    steps:
      - script: echo "Job A"

  - job: B
    dependsOn: A
    condition: succeeded()
    steps:
      - script: echo "Job B"

  - job: C
    dependsOn:
      - A
      - B
    condition: |
      and(
        succeeded('A'),
        succeeded('B')
      )
    steps:
      - script: echo "Job C"
```

## Deployment Jobs

### Environment Deployment

```yaml
stages:
  - stage: Deploy
    jobs:
      - deployment: DeployWeb
        displayName: 'Deploy to Production'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying..."
                - task: AzureWebApp@1
                  inputs:
                    appName: 'mywebapp'
```

### Rolling Deployment

```yaml
jobs:
  - deployment: Deploy
    environment:
      name: 'production'
      resourceType: VirtualMachine
    strategy:
      rolling:
        maxParallel: 2
        preDeploy:
          steps:
            - script: echo "Pre-deploy"
        deploy:
          steps:
            - script: echo "Deploying"
        routeTraffic:
          steps:
            - script: echo "Routing traffic"
        postDeploy:
          steps:
            - script: echo "Post-deploy"
```

### Canary Deployment

```yaml
strategy:
  canary:
    increments: [10, 20]
    preDeploy:
      steps:
        - script: echo "Pre-deploy"
    deploy:
      steps:
        - script: echo "Deploying $(Strategy.CycleNumber)%"
    routeTraffic:
      steps:
        - script: echo "Routing $(Strategy.CycleNumber)% traffic"
    postRouteTraffic:
      steps:
        - script: echo "Testing deployment"
    on:
      failure:
        steps:
          - script: echo "Rollback"
      success:
        steps:
          - script: echo "Success"
```

## Templates

### Step Template

```yaml
# templates/build-steps.yml
parameters:
  - name: configuration
    default: 'Release'

steps:
  - task: DotNetCoreCLI@2
    displayName: 'Restore'
    inputs:
      command: 'restore'

  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      arguments: '--configuration ${{ parameters.configuration }}'
```

```yaml
# azure-pipelines.yml
steps:
  - template: templates/build-steps.yml
    parameters:
      configuration: 'Debug'
```

### Job Template

```yaml
# templates/test-job.yml
parameters:
  - name: vmImage
    default: 'ubuntu-latest'
  - name: testCommand
    default: 'npm test'

jobs:
  - job: Test
    pool:
      vmImage: ${{ parameters.vmImage }}
    steps:
      - checkout: self
      - script: ${{ parameters.testCommand }}
```

```yaml
# azure-pipelines.yml
jobs:
  - template: templates/test-job.yml
    parameters:
      vmImage: 'windows-latest'
      testCommand: 'npm run test:windows'
```

### Stage Template

```yaml
# templates/deploy-stage.yml
parameters:
  - name: environment
  - name: dependsOn
    default: []

stages:
  - stage: Deploy_${{ parameters.environment }}
    dependsOn: ${{ parameters.dependsOn }}
    jobs:
      - deployment: Deploy
        environment: ${{ parameters.environment }}
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to ${{ parameters.environment }}"
```

### Extends Templates

```yaml
# templates/pipeline-template.yml
parameters:
  - name: buildConfiguration
    default: 'Release'

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - script: echo "Building with ${{ parameters.buildConfiguration }}"

  - stage: Test
    dependsOn: Build
    jobs:
      - job: Test
        steps:
          - script: echo "Testing"
```

```yaml
# azure-pipelines.yml
extends:
  template: templates/pipeline-template.yml
  parameters:
    buildConfiguration: 'Debug'
```

## Tasks

### Common Tasks

```yaml
steps:
  # Checkout
  - checkout: self
    clean: true
    fetchDepth: 1

  # Script
  - script: echo "Hello World"
    displayName: 'Run script'

  # PowerShell
  - powershell: Write-Host "Hello"
    displayName: 'Run PowerShell'

  # Bash
  - bash: echo "Hello"
    displayName: 'Run Bash'

  # .NET
  - task: DotNetCoreCLI@2
    inputs:
      command: 'build'
      projects: '**/*.csproj'

  # Node.js
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'

  # Docker
  - task: Docker@2
    inputs:
      command: 'buildAndPush'
      repository: 'myrepo/myimage'
      dockerfile: '**/Dockerfile'
      containerRegistry: 'dockerhubConnection'

  # Azure Web App
  - task: AzureWebApp@1
    inputs:
      azureSubscription: 'my-subscription'
      appName: 'my-web-app'
      package: '$(System.DefaultWorkingDirectory)/**/*.zip'
```

### Publish Artifacts

```yaml
steps:
  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: '$(Build.ArtifactStagingDirectory)'
      artifactName: 'drop'

  # Or use Pipeline Artifacts (newer)
  - publish: $(Build.ArtifactStagingDirectory)
    artifact: 'drop'
```

### Download Artifacts

```yaml
steps:
  - download: current
    artifact: 'drop'

  # From another pipeline
  - download: PipelineName
    artifact: 'drop'
    patterns: '**/*.zip'
```

## Service Connections

### Creating Service Connection

```
Project Settings → Service connections → New service connection

Types:
- Azure Resource Manager
- Docker Registry
- Kubernetes
- GitHub
- SSH
- Generic
```

### Using Service Connection

```yaml
steps:
  - task: AzureWebApp@1
    inputs:
      azureSubscription: 'my-azure-connection'  # Service connection name
      appName: 'myapp'

  - task: Docker@2
    inputs:
      containerRegistry: 'my-docker-registry'  # Service connection name
      command: 'push'
```

## Conditions

### Job Conditions

```yaml
jobs:
  - job: Build
    condition: succeeded()

  - job: Deploy
    condition: |
      and(
        succeeded(),
        eq(variables['Build.SourceBranch'], 'refs/heads/main')
      )

  - job: Notify
    condition: always()

  - job: Rollback
    condition: failed()
```

### Step Conditions

```yaml
steps:
  - script: echo "Only on main"
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')

  - script: echo "Only for PR"
    condition: eq(variables['Build.Reason'], 'PullRequest')

  - script: echo "Run if previous succeeded"
    condition: succeeded()

  - script: echo "Always runs"
    condition: always()
```

## Complete Example

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pr:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: 'common-variables'
  - name: imageRepository
    value: 'myapp'
  - name: dockerRegistryServiceConnection
    value: 'docker-hub'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: Build
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'

          - script: npm ci
            displayName: 'Install dependencies'

          - script: npm run lint
            displayName: 'Lint'

          - script: npm test
            displayName: 'Test'

          - script: npm run build
            displayName: 'Build'

          - task: Docker@2
            displayName: 'Build Docker image'
            inputs:
              command: 'build'
              repository: $(imageRepository)
              dockerfile: 'Dockerfile'
              tags: '$(Build.BuildId)'

          - task: Docker@2
            displayName: 'Push Docker image'
            condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
            inputs:
              command: 'push'
              containerRegistry: $(dockerRegistryServiceConnection)
              repository: $(imageRepository)
              tags: '$(Build.BuildId)'

  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to staging"

  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to production"
```

## Quick Reference

### Pipeline Syntax

| Element | Description |
|---------|-------------|
| `trigger` | Branch triggers |
| `pr` | Pull request triggers |
| `schedules` | Cron triggers |
| `pool` | Agent pool |
| `stages` | Pipeline stages |
| `jobs` | Stage jobs |
| `steps` | Job steps |

### Condition Functions

| Function | Description |
|----------|-------------|
| `succeeded()` | Previous succeeded |
| `failed()` | Previous failed |
| `always()` | Always run |
| `eq(a, b)` | Equals |
| `ne(a, b)` | Not equals |
| `and()` | Logical AND |
| `or()` | Logical OR |

### Variables

| Variable | Description |
|----------|-------------|
| `$(Build.BuildId)` | Build ID |
| `$(Build.SourceBranch)` | Source branch |
| `$(Build.SourceVersion)` | Commit SHA |
| `$(System.DefaultWorkingDirectory)` | Working directory |
| `$(Pipeline.Workspace)` | Pipeline workspace |

---

**Previous:** [13-github-actions-advanced.md](13-github-actions-advanced.md) | **Next:** [15-build-automation.md](15-build-automation.md)
