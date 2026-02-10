# Azure Cloud Cheat Sheet for .NET Backend Engineers

## Quick Reference - Azure Services for .NET

### Azure App Service for .NET
```bash
# Create App Service for .NET 8
az webapp create -g myRG -p myPlan -n myDotNetApp --runtime "DOTNET|8.0"

# Deploy from local
dotnet publish -c Release -o ./publish
cd publish && zip -r ../app.zip .
az webapp deploy -g myRG -n myDotNetApp --src-path app.zip

# Configure settings
az webapp config appsettings set -g myRG -n myDotNetApp \
  --settings ASPNETCORE_ENVIRONMENT=Production

# Connection string
az webapp config connection-string set -g myRG -n myDotNetApp \
  --connection-string-type SQLAzure \
  --settings DefaultConnection="Server=..."

# View logs
az webapp log tail -g myRG -n myDotNetApp
```

### Azure SQL with .NET
```bash
# Create SQL Server and Database
az sql server create -g myRG -n mysqlserver -u sqladmin -p <password>
az sql db create -g myRG -s mysqlserver -n mydb --service-objective S0

# Allow Azure services
az sql server firewall-rule create -g myRG -s mysqlserver \
  -n AllowAzure --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

**Connection String in .NET:**
```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=tcp:mysqlserver.database.windows.net,1433;Database=mydb;User ID=sqladmin;Password={password};Encrypt=true;TrustServerCertificate=false;"
  }
}

// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

---

## Azure SDK for .NET

### Common NuGet Packages
```bash
dotnet add package Azure.Identity
dotnet add package Azure.Storage.Blobs
dotnet add package Azure.Security.KeyVault.Secrets
dotnet add package Azure.Messaging.ServiceBus
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

### Azure Identity (Authentication)
```csharp
using Azure.Identity;

// DefaultAzureCredential works with:
// - Environment variables
// - Managed Identity
// - Visual Studio
// - Azure CLI
// - Azure PowerShell

var credential = new DefaultAzureCredential();

// For specific credential
var credential = new ManagedIdentityCredential();
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
```

### Blob Storage
```csharp
using Azure.Storage.Blobs;

// Connection string approach
var blobClient = new BlobServiceClient(connectionString);

// Managed identity approach
var blobClient = new BlobServiceClient(
    new Uri("https://mystorageacct.blob.core.windows.net"),
    new DefaultAzureCredential());

// Upload
var containerClient = blobClient.GetBlobContainerClient("mycontainer");
var blob = containerClient.GetBlobClient("myfile.txt");
await blob.UploadAsync(stream, overwrite: true);

// Download
var response = await blob.DownloadAsync();
using var reader = new StreamReader(response.Value.Content);
var content = await reader.ReadToEndAsync();

// List blobs
await foreach (var blobItem in containerClient.GetBlobsAsync())
{
    Console.WriteLine(blobItem.Name);
}
```

### Key Vault
```csharp
using Azure.Security.KeyVault.Secrets;

var client = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net/"),
    new DefaultAzureCredential());

// Get secret
KeyVaultSecret secret = await client.GetSecretAsync("MySecret");
string secretValue = secret.Value;

// Set secret
await client.SetSecretAsync("MySecret", "secret-value");

// In Program.cs - Add Key Vault to configuration
builder.Configuration.AddAzureKeyVault(
    new Uri("https://mykeyvault.vault.azure.net/"),
    new DefaultAzureCredential());

// Access in code
var mySecret = builder.Configuration["MySecret"];
```

### Service Bus
```csharp
using Azure.Messaging.ServiceBus;

// Send message
var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("myqueue");
await sender.SendMessageAsync(new ServiceBusMessage("Hello"));

// Receive messages
var processor = client.CreateProcessor("myqueue");
processor.ProcessMessageAsync += async args =>
{
    string body = args.Message.Body.ToString();
    await args.CompleteMessageAsync(args.Message);
};
processor.ProcessErrorAsync += args =>
{
    Console.WriteLine(args.Exception);
    return Task.CompletedTask;
};
await processor.StartProcessingAsync();
```

---

## Application Insights

### Setup in .NET
```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry();

// appsettings.json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=xxx;IngestionEndpoint=..."
  }
}

// Or via environment variable
// APPLICATIONINSIGHTS_CONNECTION_STRING
```

### Custom Telemetry
```csharp
public class MyService
{
    private readonly TelemetryClient _telemetry;

    public MyService(TelemetryClient telemetry)
    {
        _telemetry = telemetry;
    }

    public void DoWork()
    {
        // Track custom event
        _telemetry.TrackEvent("OrderProcessed", new Dictionary<string, string>
        {
            { "OrderId", "12345" }
        });

        // Track metric
        _telemetry.TrackMetric("OrderAmount", 99.99);

        // Track dependency
        using var operation = _telemetry.StartOperation<DependencyTelemetry>("ExternalAPI");
        // ... call external API
        operation.Telemetry.Success = true;

        // Track exception
        try { /* ... */ }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex);
            throw;
        }
    }
}
```

---

## Managed Identity

### Enable in App Service
```bash
az webapp identity assign -g myRG -n myDotNetApp
```

### Access Azure Resources
```csharp
// No connection strings needed!
// Managed identity authenticates automatically

// Key Vault
var secretClient = new SecretClient(
    new Uri("https://mykeyvault.vault.azure.net/"),
    new DefaultAzureCredential());

// Blob Storage
var blobClient = new BlobServiceClient(
    new Uri("https://mystorageacct.blob.core.windows.net"),
    new DefaultAzureCredential());

// SQL Database (with Azure AD authentication)
// Connection string: Server=myserver.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity;

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### Grant Permissions
```bash
# Key Vault access
az keyvault set-policy --name mykeyvault \
  --object-id <managed-identity-object-id> \
  --secret-permissions get list

# Storage access
az role assignment create \
  --assignee <managed-identity-object-id> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>
```

---

## Interview Q&A

### Q1: How do you deploy a .NET application to Azure?
**A:** Multiple options:
1. **App Service**: `az webapp deploy` or CI/CD
2. **AKS**: Docker container to Kubernetes
3. **Container Instances**: Quick container deployment
4. **Azure Functions**: Serverless functions

### Q2: How do you handle configuration in Azure for .NET?
**A:**
- **App Settings**: Non-sensitive configuration
- **Connection Strings**: Database connections
- **Key Vault**: Secrets (use managed identity)
- **Azure App Configuration**: Centralized config with feature flags

```csharp
// Load from Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
    options.Connect(connectionString)
           .Select("MyApp:*")
           .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential())));
```

### Q3: What is managed identity and why use it?
**A:** Azure-managed identity that eliminates credential management:
- No secrets in code or config
- Automatic credential rotation
- Works with Azure services (Key Vault, SQL, Storage)
- System-assigned or user-assigned

### Q4: How do you monitor a .NET application in Azure?
**A:**
- **Application Insights**: APM, traces, exceptions
- **Azure Monitor**: Metrics, alerts
- **Log Analytics**: Query logs with KQL
- **Health checks**: Endpoint monitoring

### Q5: How do you handle database connections in Azure?
**A:**
```csharp
// Connection string in Key Vault or App Settings
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(3);  // Transient fault handling
        sqlOptions.CommandTimeout(30);
    });
});

// With managed identity
// Server=myserver.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity;
```

### Q6: How do you implement caching in Azure?
**A:**
```csharp
// Azure Cache for Redis
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "myredis.redis.cache.windows.net:6380,password=...,ssl=True";
});

// Usage
public class MyService
{
    private readonly IDistributedCache _cache;

    public async Task<string> GetDataAsync(string key)
    {
        var cached = await _cache.GetStringAsync(key);
        if (cached != null) return cached;

        var data = await FetchDataAsync();
        await _cache.SetStringAsync(key, data, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        });
        return data;
    }
}
```

---

## Azure Functions with .NET

### HTTP Trigger
```csharp
public class HttpTriggerFunction
{
    [Function("HttpTrigger")]
    public HttpResponseData Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequestData req)
    {
        var response = req.CreateResponse(HttpStatusCode.OK);
        response.WriteString("Hello from Azure Functions!");
        return response;
    }
}
```

### Timer Trigger
```csharp
public class TimerTriggerFunction
{
    [Function("TimerTrigger")]
    public void Run([TimerTrigger("0 */5 * * * *")] TimerInfo timer)
    {
        // Runs every 5 minutes
    }
}
```

### Service Bus Trigger
```csharp
public class ServiceBusTriggerFunction
{
    [Function("ProcessOrder")]
    public void Run(
        [ServiceBusTrigger("orders", Connection = "ServiceBusConnection")] string message)
    {
        var order = JsonSerializer.Deserialize<Order>(message);
        // Process order
    }
}
```

---

## Best Practices

1. **Use managed identity** - No credentials in code
2. **Store secrets in Key Vault** - Never in appsettings
3. **Enable Application Insights** - Full observability
4. **Use connection resiliency** - `EnableRetryOnFailure`
5. **Implement health checks** - For load balancers
6. **Use staging slots** - Zero-downtime deployments
7. **Enable diagnostic logging** - Debug production issues
8. **Use Azure App Configuration** - Centralized config management
