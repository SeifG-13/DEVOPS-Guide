# Complete .NET Backend Engineer Cheat Sheet

> A comprehensive guide for .NET backend interview preparation and daily reference

---

## Table of Contents
1. [.NET on Linux](#1-net-on-linux)
2. [Networking in .NET](#2-networking-in-net)
3. [Git for .NET Projects](#3-git-for-net-projects)
4. [Automation Scripts](#4-automation-scripts)
5. [Docker for .NET](#5-docker-for-net)
6. [Kubernetes for .NET](#6-kubernetes-for-net)
7. [CI/CD for .NET](#7-cicd-for-net)
8. [Infrastructure Automation](#8-infrastructure-automation)
9. [Cloud (.NET on Azure)](#9-cloud-net-on-azure)
10. [Monitoring .NET Apps](#10-monitoring-net-apps)
11. [Logging in .NET](#11-logging-in-net)
12. [GitOps for .NET](#12-gitops-for-net)
13. [Security in .NET](#13-security-in-net)
14. [Helm for .NET](#14-helm-for-net)
15. [Interview Quick Reference](#15-interview-quick-reference)

---

## 1. .NET on Linux

### Essential Commands
```bash
# .NET CLI
dotnet --version
dotnet new webapi -n MyApi
dotnet build | run | publish | test
dotnet watch run                    # Hot reload

# Run in production
ASPNETCORE_ENVIRONMENT=Production dotnet MyApi.dll
```

### Systemd Service
```ini
[Unit]
Description=My .NET API
After=network.target

[Service]
WorkingDirectory=/var/www/myapi
ExecStart=/usr/bin/dotnet /var/www/myapi/MyApi.dll
Restart=always
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production

[Install]
WantedBy=multi-user.target
```

### Key Paths
| Path | Purpose |
|------|---------|
| `/var/www/myapi/` | Application files |
| `/etc/myapi/` | Configuration |
| `/var/log/myapi/` | Logs |
| `~/.dotnet/` | User tools |

---

## 2. Networking in .NET

### HttpClient Best Practice
```csharp
// WRONG - Socket exhaustion
using var client = new HttpClient();

// RIGHT - Use IHttpClientFactory
builder.Services.AddHttpClient("MyApi", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
});

public class MyService
{
    private readonly HttpClient _client;
    public MyService(IHttpClientFactory factory)
    {
        _client = factory.CreateClient("MyApi");
    }
}
```

### Resilience with Polly
```csharp
builder.Services.AddHttpClient("MyApi")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, _ => TimeSpan.FromMilliseconds(300)))
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

### Kestrel Configuration
```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(8080);
    options.ListenAnyIP(8443, listenOptions =>
        listenOptions.UseHttps("cert.pfx", "password"));
});
```

---

## 3. Git for .NET Projects

### Essential .gitignore
```gitignore
bin/
obj/
.vs/
*.user
packages/
TestResults/
publish/
appsettings.*.json
!appsettings.Development.json
```

### Package Lock
```xml
<!-- Enable in .csproj -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

### Common Commands
```bash
dotnet new gitignore
git add *.cs *.csproj
git commit -m "Add feature"
```

---

## 4. Automation Scripts

### Build & Deploy Script
```bash
#!/bin/bash
set -euo pipefail

# Build
dotnet restore
dotnet build --configuration Release --no-restore
dotnet test --no-build --configuration Release

# Publish
dotnet publish -c Release -o ./publish

# Deploy
sudo systemctl stop myapi
cp -r ./publish/* /var/www/myapi/
sudo systemctl start myapi

# Health check
sleep 5
curl -f http://localhost:5000/health || exit 1
echo "Deployment successful!"
```

### Database Migration Script
```bash
#!/bin/bash
dotnet ef database update --project src/MyApp.Data
```

---

## 5. Docker for .NET

### Optimized Dockerfile
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["*.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
HEALTHCHECK CMD wget -q --spider http://localhost:8080/health || exit 1
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### Docker Compose
```yaml
services:
  api:
    build: .
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__Default=Server=db;Database=mydb;User=sa;Password=secret
    depends_on:
      - db

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Password
```

### Key Images
| Image | Purpose | Size |
|-------|---------|------|
| `sdk:8.0` | Build | ~700MB |
| `aspnet:8.0` | Runtime | ~210MB |
| `aspnet:8.0-alpine` | Minimal | ~100MB |

---

## 6. Kubernetes for .NET

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapi
          image: myregistry/myapi:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
          envFrom:
            - configMapRef:
                name: myapi-config
            - secretRef:
                name: myapi-secrets
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

### Health Checks in .NET
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(redisConnection);

app.MapHealthChecks("/health/ready");
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false
});
```

---

## 7. CI/CD for .NET

### GitHub Actions
```yaml
name: .NET CI/CD

on:
  push:
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
        run: dotnet build --no-restore -c Release

      - name: Test
        run: dotnet test --no-build -c Release

      - name: Publish
        run: dotnet publish -c Release -o ./publish

      - name: Docker Build & Push
        run: |
          docker build -t myregistry/myapi:${{ github.sha }} .
          docker push myregistry/myapi:${{ github.sha }}
```

### Key Pipeline Stages
```
Restore → Build → Test → Publish → Docker → Deploy
```

---

## 8. Infrastructure Automation

### Terraform for .NET (Azure)
```hcl
resource "azurerm_linux_web_app" "api" {
  name                = "myapi-app"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      dotnet_version = "8.0"
    }
    health_check_path = "/health"
  }

  app_settings = {
    "ASPNETCORE_ENVIRONMENT" = "Production"
  }

  connection_string {
    name  = "DefaultConnection"
    type  = "SQLAzure"
    value = "Server=${azurerm_mssql_server.main.fqdn};..."
  }
}
```

### Ansible Deployment
```yaml
- name: Deploy .NET App
  hosts: webservers
  tasks:
    - name: Stop service
      systemd:
        name: myapi
        state: stopped

    - name: Copy files
      copy:
        src: publish/
        dest: /var/www/myapi/

    - name: Start service
      systemd:
        name: myapi
        state: started
```

---

## 9. Cloud (.NET on Azure)

### Azure SDK
```csharp
// Managed Identity (no credentials!)
var credential = new DefaultAzureCredential();

// Key Vault
var secretClient = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net/"),
    credential);
var secret = await secretClient.GetSecretAsync("MySecret");

// Blob Storage
var blobClient = new BlobServiceClient(
    new Uri("https://mystorageacct.blob.core.windows.net"),
    credential);
```

### App Settings
```csharp
// Load from Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
    options.Connect(connectionString)
           .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential())));
```

### Key Services for .NET
| Service | Use Case |
|---------|----------|
| App Service | Web API hosting |
| Azure SQL | Database |
| Key Vault | Secrets |
| Application Insights | Monitoring |
| Azure Cache for Redis | Caching |

---

## 10. Monitoring .NET Apps

### Prometheus Metrics
```csharp
// Install: prometheus-net.AspNetCore
app.UseHttpMetrics();
app.MapMetrics();

// Custom metrics
private static readonly Counter OrdersProcessed = Metrics
    .CreateCounter("orders_processed_total", "Total orders",
        new CounterConfiguration { LabelNames = new[] { "status" } });

private static readonly Histogram RequestDuration = Metrics
    .CreateHistogram("request_duration_seconds", "Request duration");

// Usage
OrdersProcessed.WithLabels("success").Inc();
using (RequestDuration.NewTimer())
{
    await ProcessAsync();
}
```

### Key Metrics
```promql
# Request rate
rate(http_requests_received_total[5m])

# Error rate
sum(rate(http_requests_received_total{code=~"5.."}[5m])) / sum(rate(http_requests_received_total[5m]))

# Latency p95
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

---

## 11. Logging in .NET

### Serilog to Elasticsearch
```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(
        new Uri("http://elasticsearch:9200"))
    {
        AutoRegisterTemplate = true,
        IndexFormat = "myapi-logs-{0:yyyy.MM}"
    })
    .CreateLogger();

builder.Host.UseSerilog();
app.UseSerilogRequestLogging();
```

### Structured Logging
```csharp
// GOOD - Structured
_logger.LogInformation("Order {OrderId} processed for {CustomerId}", orderId, customerId);

// BAD - String interpolation
_logger.LogInformation($"Order {orderId} processed");
```

### Correlation IDs
```csharp
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
        ?? Guid.NewGuid().ToString();

    using (LogContext.PushProperty("CorrelationId", correlationId))
    {
        await next();
    }
});
```

---

## 12. GitOps for .NET

### Repository Structure
```
app-repo/                    # Source code
├── src/MyApi/
├── Dockerfile
└── .github/workflows/ci.yml

gitops-repo/                 # K8s manifests
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    └── production/
```

### CI Updates GitOps Repo
```yaml
- name: Update image tag
  run: |
    cd gitops-repo/overlays/production
    kustomize edit set image myapi=${{ env.IMAGE }}:${{ github.sha }}
    git commit -am "Update myapi to ${{ github.sha }}"
    git push
```

---

## 13. Security in .NET

### Input Validation
```csharp
public class CreateUserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [MinLength(8)]
    public string Password { get; set; }
}
```

### SQL Injection Prevention
```csharp
// SAFE - EF Core (parameterized)
var user = await _context.Users
    .Where(u => u.Email == email)
    .FirstOrDefaultAsync();

// SAFE - Interpolated
var users = await _context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}")
    .ToListAsync();

// DANGEROUS - Raw with concatenation
var query = $"SELECT * FROM Users WHERE Email = '{email}'";  // NO!
```

### JWT Authentication
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = config["Jwt:Issuer"],
            ValidAudience = config["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(config["Jwt:Key"]))
        };
    });
```

### Security Headers
```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    await next();
});
```

---

## 14. Helm for .NET

### values.yaml
```yaml
image:
  repository: myregistry/myapi
  tag: "1.0.0"

dotnet:
  environment: Production
  urls: "http://+:8080"

appSettings:
  Logging__LogLevel__Default: "Information"

secrets:
  existingSecret: myapi-secrets

resources:
  limits:
    cpu: 500m
    memory: 256Mi

healthCheck:
  livenessPath: /health/live
  readinessPath: /health/ready
```

### Install
```bash
helm install myapi ./charts/myapi \
  --namespace production \
  --set image.tag=v1.2.3 \
  -f values-production.yaml
```

---

## 15. Interview Quick Reference

### Common Questions & Answers

| Question | Key Points |
|----------|------------|
| How do you handle configuration? | appsettings.json, environment variables, Azure App Configuration, Key Vault |
| How do you prevent SQL injection? | EF Core (parameterized), never concatenate user input |
| How do you secure an API? | JWT, HTTPS, input validation, rate limiting, CORS |
| How do you handle logging? | Serilog, structured logging, correlation IDs, ELK |
| How do you deploy to Kubernetes? | Docker image, Deployment, ConfigMap, Secrets, Ingress |
| How do you manage secrets? | Key Vault, User Secrets (dev), environment variables, never in code |

### .NET 8 Defaults
- Default port: **8080** (changed from 80)
- Minimal APIs preferred for simple endpoints
- Native AOT support
- Better performance

### Architecture Patterns
```csharp
// Clean Architecture Layers
Presentation → Application → Domain ← Infrastructure
(Controllers)  (Services)   (Entities) (Repositories)
```

### Performance Tips
```csharp
// Use async/await
public async Task<User> GetUserAsync(int id)

// Use IAsyncEnumerable for large datasets
public async IAsyncEnumerable<Order> GetOrdersAsync()

// Use caching
builder.Services.AddStackExchangeRedisCache(...)
```

### Quick Commands
```bash
# Development
dotnet watch run
dotnet test --filter "Category=Unit"

# Production
dotnet publish -c Release -o ./publish
dotnet MyApi.dll --urls "http://+:8080"

# Docker
docker build -t myapi .
docker run -p 5000:8080 myapi

# Kubernetes
kubectl apply -f k8s/
kubectl logs -f deployment/myapi
```

### Health Check Template
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(redisConnection)
    .AddUrlGroup(new Uri("https://external-api/health"), "external-api");

app.MapHealthChecks("/health/ready");
app.MapHealthChecks("/health/live", new() { Predicate = _ => false });
```

---

## Quick Reference Card

### Project Structure
```
src/
├── MyApi/                 # Web API
├── MyApi.Core/            # Domain logic
├── MyApi.Infrastructure/  # Data access
└── MyApi.Tests/           # Tests
```

### NuGet Essentials
```bash
dotnet add package Serilog.AspNetCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Azure.Identity
dotnet add package prometheus-net.AspNetCore
dotnet add package Polly
```

### Configuration Priority (highest to lowest)
1. Command line arguments
2. Environment variables
3. User secrets (development)
4. appsettings.{Environment}.json
5. appsettings.json

---

> **Pro Tip**: In interviews, focus on explaining your reasoning and trade-offs. Show that you understand not just *how* but *why*.
