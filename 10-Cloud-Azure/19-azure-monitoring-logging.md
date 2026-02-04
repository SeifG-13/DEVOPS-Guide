# Azure Monitoring and Logging

## Azure Monitor Overview

Azure Monitor is a comprehensive solution for collecting, analyzing, and acting on telemetry data.

```
┌─────────────────────────────────────────────────────────────┐
│                    AZURE MONITOR                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Data Sources                                               │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │   App   │ │   VM    │ │  AKS    │ │ Custom  │          │
│  │ Service │ │         │ │         │ │  Apps   │          │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
│       │           │           │           │                 │
│       └───────────┴─────┬─────┴───────────┘                 │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Azure Monitor                          │   │
│  │  ┌───────────────┐  ┌───────────────┐             │   │
│  │  │    Metrics    │  │     Logs      │             │   │
│  │  │  (Real-time)  │  │ (Log Analytics)│            │   │
│  │  └───────────────┘  └───────────────┘             │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│       ┌─────────────────┼─────────────────┐                │
│       ▼                 ▼                 ▼                │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐           │
│  │ Alerts  │      │Dashboards│     │Workbooks│           │
│  └─────────┘      └─────────┘      └─────────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Application Insights

### Enable Application Insights

```bash
# Create Application Insights resource
az monitor app-insights component create \
  --app ai-myapp-prod \
  --location eastus \
  --resource-group rg-myapp-prod \
  --application-type web

# Get connection string
az monitor app-insights component show \
  --app ai-myapp-prod \
  --resource-group rg-myapp-prod \
  --query "connectionString" -o tsv
```

### Configure in .NET 10

```bash
# Add NuGet package
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry();

var app = builder.Build();
```

```json
// appsettings.json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=xxx;IngestionEndpoint=https://eastus-0.in.applicationinsights.azure.com/"
  }
}
```

### Or via App Service Configuration

```bash
# Set via App Service settings
az webapp config appsettings set \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=xxx..."
```

### Custom Telemetry

```csharp
using Microsoft.ApplicationInsights;

public class OrderService
{
    private readonly TelemetryClient _telemetry;

    public OrderService(TelemetryClient telemetry)
    {
        _telemetry = telemetry;
    }

    public async Task ProcessOrder(Order order)
    {
        // Track custom event
        _telemetry.TrackEvent("OrderProcessed", new Dictionary<string, string>
        {
            ["OrderId"] = order.Id.ToString(),
            ["CustomerId"] = order.CustomerId.ToString()
        });

        // Track custom metric
        _telemetry.TrackMetric("OrderValue", order.TotalAmount);

        // Track dependency
        using var operation = _telemetry.StartOperation<DependencyTelemetry>("ProcessPayment");
        try
        {
            await ProcessPayment(order);
            operation.Telemetry.Success = true;
        }
        catch (Exception ex)
        {
            operation.Telemetry.Success = false;
            _telemetry.TrackException(ex);
            throw;
        }
    }
}
```

---

## Application Insights for Angular

### Install SDK

```bash
npm install @microsoft/applicationinsights-web
```

### Configure in Angular

```typescript
// src/app/services/app-insights.service.ts
import { Injectable } from '@angular/core';
import { ApplicationInsights } from '@microsoft/applicationinsights-web';
import { environment } from '../../environments/environment';

@Injectable({
  providedIn: 'root'
})
export class AppInsightsService {
  private appInsights: ApplicationInsights;

  constructor() {
    this.appInsights = new ApplicationInsights({
      config: {
        connectionString: environment.appInsightsConnectionString,
        enableAutoRouteTracking: true,
        enableCorsCorrelation: true,
        enableRequestHeaderTracking: true,
        enableResponseHeaderTracking: true
      }
    });
    this.appInsights.loadAppInsights();
  }

  trackEvent(name: string, properties?: { [key: string]: string }) {
    this.appInsights.trackEvent({ name }, properties);
  }

  trackException(error: Error) {
    this.appInsights.trackException({ exception: error });
  }

  trackPageView(name: string) {
    this.appInsights.trackPageView({ name });
  }
}
```

### Global Error Handler

```typescript
// src/app/error-handler.ts
import { ErrorHandler, Injectable } from '@angular/core';
import { AppInsightsService } from './services/app-insights.service';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private appInsights: AppInsightsService) {}

  handleError(error: Error) {
    this.appInsights.trackException(error);
    console.error(error);
  }
}

// app.module.ts
providers: [
  { provide: ErrorHandler, useClass: GlobalErrorHandler }
]
```

---

## Log Analytics Workspace

### Create Workspace

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --workspace-name log-myapp-prod \
  --resource-group rg-myapp-prod \
  --location eastus \
  --retention-time 30

# Get workspace ID
az monitor log-analytics workspace show \
  --workspace-name log-myapp-prod \
  --resource-group rg-myapp-prod \
  --query "customerId" -o tsv
```

### Enable Diagnostic Settings

```bash
# Get resource ID
APP_ID=$(az webapp show --name app-myapi-prod --resource-group rg-myapp-prod --query "id" -o tsv)
WORKSPACE_ID=$(az monitor log-analytics workspace show --workspace-name log-myapp-prod --resource-group rg-myapp-prod --query "id" -o tsv)

# Enable diagnostics
az monitor diagnostic-settings create \
  --name "app-diagnostics" \
  --resource $APP_ID \
  --workspace $WORKSPACE_ID \
  --logs '[
    {"category": "AppServiceHTTPLogs", "enabled": true},
    {"category": "AppServiceConsoleLogs", "enabled": true},
    {"category": "AppServiceAppLogs", "enabled": true}
  ]' \
  --metrics '[
    {"category": "AllMetrics", "enabled": true}
  ]'
```

### KQL Queries

```kusto
// Application errors in last 24 hours
AppExceptions
| where TimeGenerated > ago(24h)
| summarize count() by ProblemId, outerMessage
| order by count_ desc

// Request duration percentiles
AppRequests
| where TimeGenerated > ago(1h)
| summarize
    p50 = percentile(DurationMs, 50),
    p90 = percentile(DurationMs, 90),
    p99 = percentile(DurationMs, 99)
    by bin(TimeGenerated, 5m)
| render timechart

// Failed requests
AppRequests
| where TimeGenerated > ago(1h)
| where Success == false
| summarize count() by ResultCode, Name
| order by count_ desc

// Slow requests
AppRequests
| where TimeGenerated > ago(1h)
| where DurationMs > 1000
| project TimeGenerated, Name, DurationMs, ResultCode
| order by DurationMs desc

// Dependency failures
AppDependencies
| where TimeGenerated > ago(1h)
| where Success == false
| summarize count() by Type, Target, Name
| order by count_ desc

// App Service HTTP logs
AppServiceHTTPLogs
| where TimeGenerated > ago(1h)
| where ScStatus >= 500
| project TimeGenerated, CsMethod, CsUriStem, ScStatus, TimeTaken
| order by TimeGenerated desc
```

---

## Alerts

### Create Alert Rule

```bash
# Create action group for notifications
az monitor action-group create \
  --name ag-myapp-alerts \
  --resource-group rg-myapp-prod \
  --short-name myapp \
  --action email devops-team devops@company.com

# Create metric alert (high CPU)
az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group rg-myapp-prod \
  --scopes /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.Web/sites/app-myapi-prod \
  --condition "avg CpuPercentage > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action ag-myapp-alerts \
  --severity 2

# Create metric alert (high response time)
az monitor metrics alert create \
  --name "High Response Time" \
  --resource-group rg-myapp-prod \
  --scopes /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.Web/sites/app-myapi-prod \
  --condition "avg HttpResponseTime > 5" \
  --window-size 5m \
  --action ag-myapp-alerts \
  --severity 2
```

### Log-Based Alert

```bash
# Create log alert for exceptions
az monitor scheduled-query create \
  --name "Application Exceptions Alert" \
  --resource-group rg-myapp-prod \
  --scopes /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.Insights/components/ai-myapp-prod \
  --condition "count > 10" \
  --condition-query "AppExceptions | where TimeGenerated > ago(5m)" \
  --evaluation-frequency 5m \
  --window-size 5m \
  --action-groups /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.Insights/actionGroups/ag-myapp-alerts \
  --severity 2
```

---

## Dashboards

### Create Dashboard via CLI

```bash
# Export existing dashboard
az portal dashboard show \
  --name myapp-dashboard \
  --resource-group rg-myapp-prod \
  > dashboard.json

# Create dashboard from template
az portal dashboard create \
  --name myapp-dashboard \
  --resource-group rg-myapp-prod \
  --input-path dashboard-template.json
```

### Dashboard JSON Structure

```json
{
  "properties": {
    "lenses": [
      {
        "parts": [
          {
            "position": { "x": 0, "y": 0, "colSpan": 6, "rowSpan": 4 },
            "metadata": {
              "type": "Extension/Microsoft_Azure_Monitoring/PartType/MetricsChartPart",
              "inputs": [
                {
                  "name": "resourceId",
                  "value": "/subscriptions/.../Microsoft.Web/sites/app-myapi-prod"
                },
                {
                  "name": "chartDefinition",
                  "value": {
                    "metrics": [
                      { "name": "CpuPercentage", "aggregationType": "Average" }
                    ]
                  }
                }
              ]
            }
          }
        ]
      }
    ]
  }
}
```

---

## Workbooks

Workbooks provide rich interactive reports combining text, queries, metrics, and parameters.

### Common Workbook Templates

- Application Insights Failures
- Application Performance
- Resource Health
- Cost Analysis

### Access Workbooks

```
Azure Portal > Monitor > Workbooks > Gallery
```

---

## Health Checks in .NET

### Configure Health Checks

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "db", "sql" })
    .AddAzureBlobStorage(
        builder.Configuration["Storage:ConnectionString"]!,
        name: "blob-storage",
        tags: new[] { "storage" })
    .AddUrlGroup(
        new Uri("https://external-api.com/health"),
        name: "external-api",
        tags: new[] { "external" });

var app = builder.Build();

// Map health endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Just checks if app is running
});
```

### Health Check NuGet Packages

```bash
dotnet add package AspNetCore.HealthChecks.SqlServer
dotnet add package AspNetCore.HealthChecks.AzureBlobStorage
dotnet add package AspNetCore.HealthChecks.Uris
dotnet add package AspNetCore.HealthChecks.UI.Client
```

---

## Structured Logging with Serilog

### Setup Serilog

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.ApplicationInsights
dotnet add package Serilog.Sinks.Console
```

```csharp
// Program.cs
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .WriteTo.Console()
    .WriteTo.ApplicationInsights(
        TelemetryConfiguration.Active,
        TelemetryConverter.Traces)
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();
```

### Structured Logging Usage

```csharp
public class OrderController : ControllerBase
{
    private readonly ILogger<OrderController> _logger;

    public OrderController(ILogger<OrderController> logger)
    {
        _logger = logger;
    }

    [HttpPost]
    public IActionResult CreateOrder(Order order)
    {
        _logger.LogInformation(
            "Processing order {OrderId} for customer {CustomerId} with total {Total}",
            order.Id,
            order.CustomerId,
            order.Total);

        return Ok();
    }
}
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Application Insights** | APM for applications |
| **Log Analytics** | Central log storage and queries |
| **Alerts** | Proactive notifications |
| **Dashboards** | Visual monitoring |
| **Workbooks** | Interactive reports |
| **Health Checks** | Application health status |

### Quick Commands

```bash
# Create App Insights
az monitor app-insights component create --app ai-name -g rg --application-type web

# Create Log Analytics
az monitor log-analytics workspace create --workspace-name log-name -g rg

# Enable diagnostics
az monitor diagnostic-settings create --name diag --resource <id> --workspace <workspace-id>

# Create alert
az monitor metrics alert create --name alert -g rg --scopes <id> --condition "avg CpuPercentage > 80"
```
