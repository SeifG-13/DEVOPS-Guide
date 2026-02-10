# CI/CD Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET CI/CD Pipeline

### Typical .NET Pipeline Stages
```
Restore → Build → Test → Analyze → Publish → Docker → Deploy
```

---

## GitHub Actions for .NET

### Complete .NET CI/CD Workflow
```yaml
name: .NET CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release --collect:"XPlat Code Coverage" --results-directory ./coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          directory: ./coverage

  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-test

    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  publish:
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Publish
        run: dotnet publish -c Release -o ./publish

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  deploy:
    runs-on: ubuntu-latest
    needs: publish
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'my-dotnet-app'
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          images: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
```

### .NET with SQL Server Migrations
```yaml
  migrate:
    runs-on: ubuntu-latest
    needs: build-and-test
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Install EF Core tools
        run: dotnet tool install --global dotnet-ef

      - name: Generate migration script
        run: dotnet ef migrations script --idempotent --output migrations.sql --project src/MyApp.Data

      - name: Run migrations
        run: sqlcmd -S ${{ secrets.DB_SERVER }} -d ${{ secrets.DB_NAME }} -U ${{ secrets.DB_USER }} -P ${{ secrets.DB_PASSWORD }} -i migrations.sql
```

---

## Azure DevOps for .NET

### Complete Azure Pipeline
```yaml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    exclude:
      - '**/*.md'

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: Build
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

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Run tests'
            inputs:
              command: 'test'
              projects: '**/*Tests.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'

          - task: PublishCodeCoverageResults@1
            displayName: 'Publish code coverage'
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'

          - task: DotNetCoreCLI@2
            displayName: 'Publish'
            inputs:
              command: 'publish'
              projects: 'src/MyApi/MyApi.csproj'
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'

          - task: PublishBuildArtifacts@1
            displayName: 'Publish artifacts'
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)'
              artifactName: 'drop'

  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployStaging
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'MyAzureSubscription'
                    appType: 'webAppLinux'
                    appName: 'myapp-staging'
                    package: '$(Pipeline.Workspace)/drop/*.zip'

  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployProduction
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'MyAzureSubscription'
                    appType: 'webAppLinux'
                    appName: 'myapp-production'
                    package: '$(Pipeline.Workspace)/drop/*.zip'
```

---

## GitLab CI for .NET

### .gitlab-ci.yml
```yaml
image: mcr.microsoft.com/dotnet/sdk:8.0

stages:
  - build
  - test
  - publish
  - deploy

variables:
  CONFIGURATION: Release
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .nuget/

before_script:
  - export NUGET_PACKAGES=.nuget

build:
  stage: build
  script:
    - dotnet restore
    - dotnet build --configuration $CONFIGURATION --no-restore
  artifacts:
    paths:
      - src/*/bin/
      - src/*/obj/
    expire_in: 1 hour

test:
  stage: test
  script:
    - dotnet test --configuration $CONFIGURATION --no-build --collect:"XPlat Code Coverage"
  coverage: '/Total[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: '**/coverage.cobertura.xml'

publish:
  stage: publish
  script:
    - dotnet publish src/MyApi/MyApi.csproj --configuration $CONFIGURATION --output ./publish
    - docker build -t $DOCKER_IMAGE .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $DOCKER_IMAGE
  only:
    - main

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapi myapi=$DOCKER_IMAGE
  environment:
    name: production
    url: https://api.example.com
  only:
    - main
  when: manual
```

---

## Interview Q&A

### Q1: How do you set up CI/CD for a .NET application?
**A:**
1. **Build stage**: `dotnet restore`, `dotnet build`
2. **Test stage**: `dotnet test` with code coverage
3. **Publish stage**: `dotnet publish` or Docker build
4. **Deploy stage**: Azure Web App, Kubernetes, or VM deployment
5. Add caching for NuGet packages
6. Configure environment-specific settings

### Q2: How do you handle database migrations in .NET CI/CD?
**A:**
```yaml
# Option 1: EF Core migrations script
- run: dotnet ef migrations script --idempotent -o migration.sql
- run: sqlcmd -S server -d db -i migration.sql

# Option 2: EF Bundle
- run: dotnet ef migrations bundle
- run: ./efbundle --connection "$CONNECTION_STRING"

# Option 3: Apply on startup (dev/staging only)
public static void Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();
    using var scope = host.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
    host.Run();
}
```

### Q3: How do you manage secrets in .NET CI/CD?
**A:**
- Use CI/CD platform secrets (GitHub Secrets, Azure DevOps Variables)
- Azure Key Vault integration
- Never store in appsettings.json
```yaml
# GitHub Actions
env:
  ConnectionStrings__Default: ${{ secrets.DB_CONNECTION }}

# Azure DevOps
variables:
  - group: 'production-secrets'
```

### Q4: How do you handle multiple environments in .NET CI/CD?
**A:**
```yaml
# Environment-specific deployments
deploy-staging:
  environment: staging
  variables:
    ASPNETCORE_ENVIRONMENT: Staging

deploy-production:
  environment: production
  variables:
    ASPNETCORE_ENVIRONMENT: Production
```
Use `appsettings.{Environment}.json` and environment variables.

### Q5: How do you optimize .NET build times in CI/CD?
**A:**
1. **Cache NuGet packages**
2. **Use incremental builds** (`--no-restore`)
3. **Parallel test execution**
4. **Multi-stage Docker builds**
5. **Separate solution for tests if large**
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: nuget-${{ hashFiles('**/*.csproj') }}
```

### Q6: How do you run integration tests in CI/CD?
**A:**
```yaml
services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    env:
      SA_PASSWORD: YourPassword123
      ACCEPT_EULA: Y
    ports:
      - 1433:1433

steps:
  - run: dotnet test --filter "Category=Integration"
    env:
      ConnectionStrings__Test: "Server=localhost;..."
```

### Q7: How do you version .NET applications in CI/CD?
**A:**
```yaml
# Using GitVersion
- uses: gittools/actions/gitversion/setup@v0
- uses: gittools/actions/gitversion/execute@v0

# Or manual versioning
- run: dotnet build /p:Version=${{ github.run_number }}

# In .csproj
<PropertyGroup>
  <Version>1.0.$(Build.BuildNumber)</Version>
</PropertyGroup>
```

---

## NuGet Package Publishing

```yaml
  publish-nuget:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Pack
        run: dotnet pack --configuration Release --output ./nupkg

      - name: Push to NuGet
        run: dotnet nuget push ./nupkg/*.nupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json
```

---

## Code Quality Integration

### SonarQube/SonarCloud
```yaml
- name: SonarCloud Scan
  uses: SonarSource/sonarcloud-github-action@master
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

# Or with dotnet-sonarscanner
- run: |
    dotnet tool install --global dotnet-sonarscanner
    dotnet sonarscanner begin /k:"project-key" /o:"org" /d:sonar.token="${{ secrets.SONAR_TOKEN }}"
    dotnet build
    dotnet sonarscanner end /d:sonar.token="${{ secrets.SONAR_TOKEN }}"
```

### Code Coverage with Coverlet
```yaml
- name: Test with coverage
  run: dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage

- name: Upload to Codecov
  uses: codecov/codecov-action@v3
  with:
    directory: ./coverage
    fail_ci_if_error: true
```

---

## Best Practices

1. **Cache NuGet packages** - Significant build time reduction
2. **Use specific .NET version** - Avoid `latest` for reproducibility
3. **Run tests in parallel** - Faster feedback
4. **Separate unit and integration tests** - Different stages
5. **Use artifacts** - Pass build output between stages
6. **Environment-based secrets** - Different values per environment
7. **Automate everything** - Migrations, deployments, rollbacks
8. **Add health checks** - Validate deployments
