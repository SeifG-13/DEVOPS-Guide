# Git Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET Projects in Git

### Essential .gitignore for .NET
```gitignore
# Build results
bin/
obj/
[Dd]ebug/
[Rr]elease/

# Visual Studio
.vs/
*.user
*.suo
*.userosscache
*.sln.docstates

# Rider
.idea/

# User-specific files
*.rsuser
*.suo
*.user
*.userosscache
*.sln.docstates

# NuGet
*.nupkg
*.snupkg
packages/
!packages/build/

# Test results
TestResults/
[Tt]est[Rr]esult*/

# Publish output
publish/

# ASP.NET
project.lock.json
**/Properties/launchSettings.json

# Secrets
appsettings.*.json
!appsettings.json
!appsettings.Development.json.template
*.pfx
*.key
secrets.json

# OS files
.DS_Store
Thumbs.db
```

### Creating .gitignore
```bash
# Using dotnet CLI
dotnet new gitignore

# Using Visual Studio template
# Automatically created with new project
```

---

## Common Git Workflows for .NET

### Starting New Project
```bash
# Create project
dotnet new webapi -n MyApi
cd MyApi

# Initialize Git
git init
git add .
git commit -m "Initial commit: Create Web API project"

# Push to remote
git remote add origin https://github.com/user/MyApi.git
git push -u origin main
```

### Feature Branch Workflow
```bash
# Start feature
git checkout -b feature/add-user-authentication

# Work on feature...
dotnet build
dotnet test

# Commit changes
git add src/
git commit -m "Add JWT authentication middleware"

# Push and create PR
git push -u origin feature/add-user-authentication
# Create Pull Request via GitHub/Azure DevOps
```

### Handling Solution Files
```bash
# Solution file changes
git add *.sln
git commit -m "Add MyProject to solution"

# Note: .sln files can cause conflicts
# Resolve by keeping both project references
```

---

## Version Control for NuGet

### packages.lock.json
```bash
# Enable in .csproj
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>

# Commit lock file
git add packages.lock.json
git commit -m "Add package lock file for reproducible builds"
```

### Central Package Management
```xml
<!-- Directory.Packages.props (commit this!) -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>
</Project>
```

---

## Branching Strategies for .NET

### GitFlow for Release Management
```
main
  └── develop
        ├── feature/add-api-endpoint
        ├── feature/user-service
        └── release/1.0.0
              └── hotfix/fix-auth-bug
```

### Semantic Versioning Tags
```bash
# Tag releases
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# In .csproj (auto-version from git tag)
<PropertyGroup>
  <Version>1.0.0</Version>
</PropertyGroup>
```

### GitVersion Integration
```yaml
# GitVersion.yml
mode: ContinuousDeployment
branches:
  main:
    tag: ''
  develop:
    tag: alpha
  feature:
    tag: feature-{BranchName}
```

---

## Interview Q&A

### Q1: How do you handle database migrations in Git?
**A:**
```bash
# Generate migration
dotnet ef migrations add AddUserTable

# Commit migration files
git add Migrations/
git commit -m "Add migration: AddUserTable"
```
Best practices:
- Always commit migrations with related code changes
- Never modify existing migrations in shared branches
- Use migration bundles for production

### Q2: How do you handle secrets in .NET Git repos?
**A:**
- **Never commit secrets** - Use .gitignore for appsettings.*.json
- **Use User Secrets** - `dotnet user-secrets set "Key" "Value"`
- **Use environment variables** - For production
- **Use Azure Key Vault / AWS Secrets Manager** - For cloud deployments
- **Use .env files with .gitignore** - For local development

### Q3: How do you version a .NET library/NuGet package?
**A:**
```xml
<!-- In .csproj -->
<PropertyGroup>
  <Version>1.2.3</Version>
  <PackageVersion>1.2.3</PackageVersion>
</PropertyGroup>
```
Or use GitVersion/Nerdbank.GitVersioning for automatic versioning based on Git tags.

### Q4: What files should be committed vs ignored in .NET?
**Commit:**
- Source code (.cs, .fs, .vb)
- Project files (.csproj, .sln)
- appsettings.json (without secrets)
- packages.lock.json
- Migrations/

**Ignore:**
- bin/, obj/
- .vs/, .idea/
- *.user
- appsettings.*.json (with secrets)
- TestResults/

### Q5: How do you handle merge conflicts in .csproj files?
**A:**
```xml
<!-- Conflict usually looks like: -->
<ItemGroup>
<<<<<<< HEAD
  <PackageReference Include="PackageA" Version="1.0" />
=======
  <PackageReference Include="PackageB" Version="2.0" />
>>>>>>> feature-branch
</ItemGroup>

<!-- Resolve by keeping both: -->
<ItemGroup>
  <PackageReference Include="PackageA" Version="1.0" />
  <PackageReference Include="PackageB" Version="2.0" />
</ItemGroup>
```

### Q6: How do you set up Git hooks for .NET?
**A:**
```bash
# Using Husky.Net
dotnet new tool-manifest
dotnet tool install Husky

# Initialize
dotnet husky install

# Add pre-commit hook
dotnet husky add pre-commit -c "dotnet format --verify-no-changes"
dotnet husky add pre-commit -c "dotnet build --no-restore"
```

### Q7: How do you handle configuration for different environments?
**A:**
```bash
# File structure
appsettings.json              # Base config (committed)
appsettings.Development.json  # Dev overrides (committed, no secrets)
appsettings.Production.json   # Prod (NOT committed or use env vars)

# .gitignore
appsettings.*.json
!appsettings.Development.json
```

---

## CI/CD Pipeline Integration

### GitHub Actions for .NET
```yaml
name: .NET Build and Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore

      - name: Test
        run: dotnet test --no-build --verbosity normal
```

### Azure DevOps Pipeline
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: UseDotNet@2
    inputs:
      version: '8.0.x'

  - script: dotnet restore
    displayName: 'Restore'

  - script: dotnet build --no-restore
    displayName: 'Build'

  - script: dotnet test --no-build
    displayName: 'Test'
```

---

## Git Commands for .NET Development

### Finding Changes
```bash
# Find who changed a method
git log -p -S "MethodName" -- "*.cs"

# Find changes to specific file
git log --follow -p -- src/Services/UserService.cs

# Find when a line was added
git blame src/Controllers/UserController.cs
```

### Undoing Changes
```bash
# Discard local changes to .cs files
git checkout -- "*.cs"

# Unstage all .cs files
git reset HEAD -- "*.cs"

# Revert migration commit
git revert <migration-commit-hash>
dotnet ef migrations remove  # If needed
```

### Working with Branches
```bash
# List branches with last commit date
git branch -a --sort=-committerdate

# Delete merged feature branches
git branch --merged main | grep feature | xargs git branch -d

# Update feature branch with main
git checkout feature/my-feature
git rebase main
```

---

## Best Practices

1. **Use .gitignore from day one** - Template for .NET projects
2. **Commit migrations with features** - Keep code and DB changes together
3. **Never commit secrets** - Use User Secrets for development
4. **Lock package versions** - Use packages.lock.json
5. **Small, focused commits** - One logical change per commit
6. **Meaningful commit messages** - Reference issue/ticket numbers
7. **Use Pull Requests** - Code review before merge
8. **Tag releases** - Semantic versioning for packages
