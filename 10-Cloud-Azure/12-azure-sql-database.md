# Azure SQL Database

## What is Azure SQL?

Azure SQL Database is a fully managed relational database service based on Microsoft SQL Server.

```
┌─────────────────────────────────────────────────────────────┐
│                    AZURE SQL DATABASE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Azure SQL Family:                                          │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│  │  Azure SQL      │ │  Azure SQL      │ │  SQL Server │  │
│  │  Database       │ │  Managed        │ │  on VM      │  │
│  │  (Single DB)    │ │  Instance       │ │  (IaaS)     │  │
│  └─────────────────┘ └─────────────────┘ └─────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Azure SQL Database                                  │   │
│  │  ├── Logical Server (sql-myapp.database.windows.net)│   │
│  │  │   ├── Database 1                                 │   │
│  │  │   ├── Database 2                                 │   │
│  │  │   └── Elastic Pool (optional)                    │   │
│  │  └── Firewall Rules                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Features: Auto-scaling, Built-in HA, Backups, Geo-repl   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Service Tiers

| Tier | Use Case | DTU/vCore |
|------|----------|-----------|
| **Basic** | Dev/test, light workloads | 5 DTU |
| **Standard** | General production | S0-S12 (10-3000 DTU) |
| **Premium** | High IOPS, low latency | P1-P15 (125-4000 DTU) |
| **General Purpose** | Business workloads | 2-128 vCores |
| **Business Critical** | OLTP, low latency | 2-128 vCores |
| **Hyperscale** | Large databases | 2-100 vCores |

---

## Create Azure SQL

### Create SQL Server and Database

```bash
# Variables
RESOURCE_GROUP="rg-myapp-prod"
LOCATION="eastus"
SQL_SERVER="sql-myapp-prod"
SQL_ADMIN="sqladmin"
SQL_PASSWORD="MySecureP@ssw0rd123!"
SQL_DB="myappdb"

# Create SQL Server
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user $SQL_ADMIN \
  --admin-password $SQL_PASSWORD

# Create database
az sql db create \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --service-objective S1 \
  --backup-storage-redundancy Local

# Show connection string
az sql db show-connection-string \
  --server $SQL_SERVER \
  --name $SQL_DB \
  --client ado.net
```

### Bicep

```bicep
param sqlServerName string
param sqlDbName string
param location string = resourceGroup().location
param adminLogin string

@secure()
param adminPassword string

resource sqlServer 'Microsoft.Sql/servers@2022-05-01-preview' = {
  name: sqlServerName
  location: location
  properties: {
    administratorLogin: adminLogin
    administratorLoginPassword: adminPassword
    version: '12.0'
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Enabled'
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2022-05-01-preview' = {
  parent: sqlServer
  name: sqlDbName
  location: location
  sku: {
    name: 'S1'
    tier: 'Standard'
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: 268435456000  // 250 GB
    catalogCollation: 'SQL_Latin1_General_CP1_CI_AS'
  }
}

resource firewallRule 'Microsoft.Sql/servers/firewallRules@2022-05-01-preview' = {
  parent: sqlServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
output connectionString string = 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Database=${sqlDbName};'
```

---

## Firewall Configuration

### Configure Firewall Rules

```bash
# Allow Azure services
az sql server firewall-rule create \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow specific IP (your development machine)
MY_IP=$(curl -s ifconfig.me)
az sql server firewall-rule create \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --name AllowMyIP \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP

# Allow IP range
az sql server firewall-rule create \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --name AllowOffice \
  --start-ip-address 203.0.113.0 \
  --end-ip-address 203.0.113.255

# List firewall rules
az sql server firewall-rule list \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  -o table
```

---

## Authentication Options

### SQL Authentication

```csharp
// Connection string with SQL auth
"Server=tcp:sql-myapp-prod.database.windows.net,1433;Database=myappdb;User ID=sqladmin;Password=MyPassword;Encrypt=True;TrustServerCertificate=False;"
```

### Azure AD Authentication (Recommended)

```bash
# Set Azure AD admin
az sql server ad-admin create \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --display-name "DBA Team" \
  --object-id <azure-ad-group-object-id>

# Enable Azure AD only authentication
az sql server ad-only-auth enable \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP
```

### Managed Identity Authentication

```csharp
// Program.cs - Use managed identity
using Azure.Identity;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
{
    var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

    // Use managed identity for authentication
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(3);
    });
});
```

```json
// appsettings.Production.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=tcp:sql-myapp-prod.database.windows.net,1433;Database=myappdb;Authentication=Active Directory Managed Identity;Encrypt=True;"
  }
}
```

### Grant Database Access to Managed Identity

```sql
-- Connect to the database and run:
CREATE USER [app-myapi-prod] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [app-myapi-prod];
ALTER ROLE db_datawriter ADD MEMBER [app-myapi-prod];
ALTER ROLE db_ddladmin ADD MEMBER [app-myapi-prod]; -- For EF migrations
```

---

## Entity Framework Core Integration

### Setup EF Core

```bash
# Add NuGet packages
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Azure.Identity
```

### DbContext

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Name)
            .IsUnique();
    }
}

public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public decimal Price { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

### Migrations

```bash
# Create migration
dotnet ef migrations add InitialCreate

# Apply migration locally
dotnet ef database update

# Generate SQL script for production
dotnet ef migrations script -o migration.sql
```

### Apply Migrations on Startup

```csharp
// Program.cs
var app = builder.Build();

// Apply pending migrations
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

    if (app.Environment.IsProduction())
    {
        // Apply migrations on startup in production
        await db.Database.MigrateAsync();
    }
}
```

---

## Connection Resiliency

### Configure Retry Logic

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);

            sqlOptions.CommandTimeout(30);
        });
});
```

---

## Scaling

### Scale Up/Down

```bash
# Scale to higher tier
az sql db update \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --service-objective S3

# Scale to vCore model
az sql db update \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2
```

### Auto-scaling (Serverless)

```bash
# Create serverless database
az sql db create \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --edition GeneralPurpose \
  --compute-model Serverless \
  --auto-pause-delay 60 \
  --min-capacity 0.5 \
  --max-capacity 4
```

---

## Backup and Recovery

### Built-in Backups

```bash
# Azure SQL automatically creates:
# - Full backups (weekly)
# - Differential backups (every 12 hours)
# - Transaction log backups (every 5-10 minutes)

# Configure retention
az sql db update \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --backup-storage-redundancy Local

# Point-in-time restore
az sql db restore \
  --dest-name myappdb-restored \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --time "2024-01-15T10:00:00Z"
```

### Long-term Retention

```bash
# Configure LTR policy (keep weekly backups for 4 weeks)
az sql db ltr-policy set \
  --server $SQL_SERVER \
  --database $SQL_DB \
  --resource-group $RESOURCE_GROUP \
  --weekly-retention "P4W" \
  --monthly-retention "P12M" \
  --yearly-retention "P5Y" \
  --week-of-year 1
```

---

## Private Endpoint

### Secure Database with Private Link

```bash
# Create private endpoint
az network private-endpoint create \
  --name pe-sql-myapp \
  --resource-group $RESOURCE_GROUP \
  --vnet-name vnet-myapp-prod \
  --subnet snet-private-endpoints \
  --private-connection-resource-id $(az sql server show --name $SQL_SERVER --resource-group $RESOURCE_GROUP --query id -o tsv) \
  --group-id sqlServer \
  --connection-name sql-connection

# Disable public access
az sql server update \
  --name $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --public-network-access Disabled

# Create private DNS zone
az network private-dns zone create \
  --name privatelink.database.windows.net \
  --resource-group $RESOURCE_GROUP

# Link to VNet
az network private-dns link vnet create \
  --name sql-dns-link \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.database.windows.net \
  --virtual-network vnet-myapp-prod \
  --registration-enabled false

# Create DNS record
az network private-endpoint dns-zone-group create \
  --name sql-dns-group \
  --resource-group $RESOURCE_GROUP \
  --endpoint-name pe-sql-myapp \
  --private-dns-zone privatelink.database.windows.net \
  --zone-name default
```

---

## Monitoring

### Enable Diagnostics

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --name log-myapp-prod \
  --resource-group $RESOURCE_GROUP

# Enable diagnostics
az monitor diagnostic-settings create \
  --name sql-diagnostics \
  --resource $(az sql db show --name $SQL_DB --server $SQL_SERVER --resource-group $RESOURCE_GROUP --query id -o tsv) \
  --workspace log-myapp-prod \
  --logs '[{"category": "SQLInsights", "enabled": true}, {"category": "QueryStoreRuntimeStatistics", "enabled": true}]' \
  --metrics '[{"category": "Basic", "enabled": true}]'
```

### Query Performance Insight

```sql
-- Find top queries by CPU
SELECT TOP 10
    qs.total_worker_time/qs.execution_count AS avg_cpu_time,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
ORDER BY avg_cpu_time DESC;
```

---

## Summary

| Feature | How To |
|---------|--------|
| Create server | `az sql server create` |
| Create database | `az sql db create` |
| Firewall rules | `az sql server firewall-rule create` |
| Azure AD auth | `az sql server ad-admin create` |
| Managed Identity | Grant access via T-SQL |
| Scale | `az sql db update --service-objective` |
| Private endpoint | `az network private-endpoint create` |

### Quick Commands

```bash
# Create server and database
az sql server create -n sql-name -g rg -l eastus -u admin -p Password123!
az sql db create -n dbname -s sql-name -g rg --service-objective S1

# Firewall
az sql server firewall-rule create -s sql-name -g rg -n AllowAzure --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0

# Connection string
az sql db show-connection-string -s sql-name -n dbname -c ado.net
```
