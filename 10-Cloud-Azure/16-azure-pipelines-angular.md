# Azure Pipelines for Angular

## Pipeline Overview

Complete CI/CD pipeline for Angular applications with build, test, and deployment to Azure.

```
┌─────────────────────────────────────────────────────────────┐
│              ANGULAR CI/CD PIPELINE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│  │  Push   │───▶│ Install │───▶│  Lint   │───▶│  Test   │ │
│  │         │    │   npm   │    │         │    │         │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘ │
│                                                     │       │
│                                              ┌──────┘       │
│                                              ▼              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     BUILD                           │   │
│  │  ng build --configuration=production                │   │
│  └─────────────────────────┬───────────────────────────┘   │
│                            │                                │
│              ┌─────────────┴─────────────┐                 │
│              ▼                           ▼                  │
│  ┌─────────────────────┐    ┌─────────────────────┐       │
│  │  Static Web Apps    │    │  Blob Storage + CDN │       │
│  │  (Auto-deploy)      │    │                     │       │
│  └─────────────────────┘    └─────────────────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Basic Angular Pipeline

### Simple Build Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - src/**
      - angular.json
      - package.json

pool:
  vmImage: 'ubuntu-latest'

variables:
  nodeVersion: '20.x'

steps:
  - task: NodeTool@0
    displayName: 'Install Node.js'
    inputs:
      versionSpec: $(nodeVersion)

  - task: Cache@2
    displayName: 'Cache npm packages'
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      restoreKeys: |
        npm | "$(Agent.OS)"
      path: $(npm_config_cache)

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npm run lint
    displayName: 'Lint'

  - script: npm run test -- --watch=false --browsers=ChromeHeadless --code-coverage
    displayName: 'Run tests'

  - task: PublishTestResults@2
    displayName: 'Publish test results'
    inputs:
      testResultsFormat: 'JUnit'
      testResultsFiles: '**/TESTS-*.xml'
    condition: succeededOrFailed()

  - task: PublishCodeCoverageResults@2
    displayName: 'Publish code coverage'
    inputs:
      summaryFileLocation: 'coverage/**/cobertura-coverage.xml'

  - script: npm run build -- --configuration=production
    displayName: 'Build production'

  - publish: dist/
    displayName: 'Publish artifact'
    artifact: angular-build
```

---

## Deploy to Static Web Apps

### Complete Pipeline for Static Web Apps

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  nodeVersion: '20.x'
  buildOutputPath: 'dist/my-angular-app/browser'

stages:
  # ========== BUILD STAGE ==========
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: Build
        displayName: 'Build Angular App'
        steps:
          - task: NodeTool@0
            displayName: 'Install Node.js'
            inputs:
              versionSpec: $(nodeVersion)

          - task: Cache@2
            displayName: 'Cache npm'
            inputs:
              key: 'npm | "$(Agent.OS)" | package-lock.json'
              path: $(npm_config_cache)

          - script: npm ci
            displayName: 'npm ci'

          - script: npm run lint
            displayName: 'Lint'

          - script: npm run test -- --watch=false --browsers=ChromeHeadless
            displayName: 'Test'

          - script: npm run build -- --configuration=production
            displayName: 'Build'

          - publish: $(buildOutputPath)
            artifact: angular-build

  # ========== DEPLOY TO STAGING (PR) ==========
  - stage: DeployPreview
    displayName: 'Deploy Preview'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.Reason'], 'PullRequest'))
    jobs:
      - job: DeployPreview
        steps:
          - download: current
            artifact: angular-build

          - task: AzureStaticWebApp@0
            displayName: 'Deploy to SWA Preview'
            inputs:
              app_location: '$(Pipeline.Workspace)/angular-build'
              skip_app_build: true
              azure_static_web_apps_api_token: $(SWA_DEPLOYMENT_TOKEN)

  # ========== DEPLOY TO PRODUCTION ==========
  - stage: DeployProduction
    displayName: 'Deploy Production'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        displayName: 'Deploy to Production'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: angular-build

                - task: AzureStaticWebApp@0
                  displayName: 'Deploy to Static Web Apps'
                  inputs:
                    app_location: '$(Pipeline.Workspace)/angular-build'
                    skip_app_build: true
                    azure_static_web_apps_api_token: $(SWA_DEPLOYMENT_TOKEN)
```

---

## Deploy to Blob Storage + CDN

### Pipeline for Blob Storage Deployment

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  nodeVersion: '20.x'
  storageAccount: 'stmyappprod'
  resourceGroup: 'rg-myapp-prod'
  cdnProfile: 'cdn-myapp-prod'
  cdnEndpoint: 'myapp-prod'
  buildOutputPath: 'dist/my-angular-app/browser'

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - script: npm ci
            displayName: 'Install dependencies'

          - script: npm run lint
            displayName: 'Lint'

          - script: npm run test -- --watch=false --browsers=ChromeHeadless
            displayName: 'Test'

          - script: npm run build -- --configuration=production
            displayName: 'Build'

          - publish: $(buildOutputPath)
            artifact: angular-build

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: DeployBlob
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: angular-build

                - task: AzureCLI@2
                  displayName: 'Upload to Blob Storage'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Delete old files
                      az storage blob delete-batch \
                        --account-name $(storageAccount) \
                        --source '$web' \
                        --auth-mode login

                      # Upload new files
                      az storage blob upload-batch \
                        --account-name $(storageAccount) \
                        --destination '$web' \
                        --source $(Pipeline.Workspace)/angular-build \
                        --auth-mode login \
                        --overwrite

                - task: AzureCLI@2
                  displayName: 'Purge CDN Cache'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az cdn endpoint purge \
                        --name $(cdnEndpoint) \
                        --profile-name $(cdnProfile) \
                        --resource-group $(resourceGroup) \
                        --content-paths "/*"
```

---

## Multi-Environment Pipeline

### Dev, Staging, Production

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop

variables:
  nodeVersion: '20.x'

stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - script: npm ci
            displayName: 'Install'

          - script: npm run test -- --watch=false --browsers=ChromeHeadless
            displayName: 'Test'

          # Build for each environment
          - script: npm run build -- --configuration=development
            displayName: 'Build Dev'

          - publish: dist/my-angular-app/browser
            artifact: build-dev

          - script: npm run build -- --configuration=staging
            displayName: 'Build Staging'

          - publish: dist/my-angular-app/browser
            artifact: build-staging

          - script: npm run build -- --configuration=production
            displayName: 'Build Prod'

          - publish: dist/my-angular-app/browser
            artifact: build-prod

  # Deploy to Dev (auto)
  - stage: DeployDev
    displayName: 'Deploy to Dev'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: Deploy
        environment: 'development'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: build-dev

                - task: AzureCLI@2
                  displayName: 'Deploy to Dev Storage'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az storage blob upload-batch \
                        --account-name stmyappdev \
                        --destination '$web' \
                        --source $(Pipeline.Workspace)/build-dev \
                        --auth-mode login \
                        --overwrite

  # Deploy to Staging (auto on main)
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        environment: 'staging'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: build-staging

                - task: AzureCLI@2
                  displayName: 'Deploy to Staging Storage'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az storage blob upload-batch \
                        --account-name stmyappstaging \
                        --destination '$web' \
                        --source $(Pipeline.Workspace)/build-staging \
                        --auth-mode login \
                        --overwrite

  # Deploy to Production (with approval)
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: 'production'  # Requires approval
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: build-prod

                - task: AzureCLI@2
                  displayName: 'Deploy to Prod Storage'
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az storage blob upload-batch \
                        --account-name stmyappprod \
                        --destination '$web' \
                        --source $(Pipeline.Workspace)/build-prod \
                        --auth-mode login \
                        --overwrite

                      # Purge CDN
                      az cdn endpoint purge \
                        --name myapp-prod \
                        --profile-name cdn-myapp-prod \
                        --resource-group rg-myapp-prod \
                        --content-paths "/*"
```

---

## Angular Environment Configuration

### environment.ts files

```typescript
// src/environments/environment.ts (development)
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5000/api'
};

// src/environments/environment.staging.ts
export const environment = {
  production: true,
  apiUrl: 'https://app-myapi-staging.azurewebsites.net/api'
};

// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://app-myapi-prod.azurewebsites.net/api'
};
```

### angular.json configurations

```json
{
  "configurations": {
    "development": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.ts"
        }
      ]
    },
    "staging": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.staging.ts"
        }
      ],
      "optimization": true,
      "outputHashing": "all"
    },
    "production": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.prod.ts"
        }
      ],
      "optimization": true,
      "outputHashing": "all",
      "sourceMap": false
    }
  }
}
```

---

## Pipeline Templates

### Angular Build Template

```yaml
# templates/angular-build.yml
parameters:
  - name: configuration
    type: string
    default: 'production'
  - name: artifactName
    type: string

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'

  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      path: $(npm_config_cache)

  - script: npm ci
    displayName: 'Install'

  - script: npm run lint
    displayName: 'Lint'

  - script: npm run test -- --watch=false --browsers=ChromeHeadless
    displayName: 'Test'

  - script: npm run build -- --configuration=${{ parameters.configuration }}
    displayName: 'Build'

  - publish: dist/my-angular-app/browser
    artifact: ${{ parameters.artifactName }}
```

### Usage

```yaml
stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - template: templates/angular-build.yml
            parameters:
              configuration: 'production'
              artifactName: 'angular-build'
```

---

## Summary

| Deployment Target | Method |
|-------------------|--------|
| **Static Web Apps** | `AzureStaticWebApp@0` task |
| **Blob Storage** | `az storage blob upload-batch` |
| **CDN Purge** | `az cdn endpoint purge` |

### Pipeline Best Practices

```yaml
# ✓ Cache npm dependencies
# ✓ Run lint and tests before build
# ✓ Publish test results and coverage
# ✓ Build once, deploy multiple times
# ✓ Use different configurations per environment
# ✓ Purge CDN after deployment
# ✓ Use deployment approvals for production
```
