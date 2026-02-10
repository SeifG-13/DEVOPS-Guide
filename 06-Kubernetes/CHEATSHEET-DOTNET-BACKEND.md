# Kubernetes Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET on Kubernetes

### Basic .NET Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  labels:
    app: myapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
    spec:
      containers:
        - name: myapi
          image: myregistry/myapi:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

### Service for .NET API
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapi-service
spec:
  type: ClusterIP
  selector:
    app: myapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

## Configuration Management

### ConfigMap for appsettings
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapi-config
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Information"
  AllowedHosts: "*"
  appsettings.Production.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information"
        }
      },
      "Features": {
        "EnableSwagger": false
      }
    }
```

### Secret for Connection Strings
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapi-secrets
type: Opaque
stringData:
  ConnectionStrings__DefaultConnection: "Server=sql-server;Database=mydb;User=sa;Password=secret"
  JWT__Secret: "your-super-secret-key-here"
```

### Using ConfigMap/Secret in Deployment
```yaml
spec:
  containers:
    - name: myapi
      image: myregistry/myapi:1.0.0
      envFrom:
        - configMapRef:
            name: myapi-config
        - secretRef:
            name: myapi-secrets
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: myapi-config
        items:
          - key: appsettings.Production.json
            path: appsettings.Production.json
```

### Reading Config in .NET
```csharp
// Program.cs
builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddJsonFile("config/appsettings.Production.json", optional: true)
    .AddEnvironmentVariables();

// Environment variable format for nested config
// ConnectionStrings__DefaultConnection -> ConnectionStrings:DefaultConnection
```

---

## Health Checks

### .NET Health Check Configuration
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(redisConnectionString)
    .AddUrlGroup(new Uri("https://external-api/health"), "external-api");

// Separate endpoints for K8s probes
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Just check if app responds
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

### Kubernetes Probes
```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30  # 30 * 5 = 150s max startup
```

---

## Complete Deployment Example

### Full .NET Microservice Stack
```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapi-config
  namespace: myapp
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: myapi-secrets
  namespace: myapp
type: Opaque
stringData:
  ConnectionStrings__DefaultConnection: "Server=sql;Database=db;User=sa;Password=secret"
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
    spec:
      containers:
        - name: myapi
          image: myregistry/myapi:1.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapi-config
            - secretRef:
                name: myapi-secrets
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: myapi-service
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: myapi
  ports:
    - port: 80
      targetPort: 8080
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapi-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapi-service
                port:
                  number: 80
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapi-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapi
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Interview Q&A

### Q1: How do you deploy a .NET API to Kubernetes?
**A:**
1. Containerize with Docker (multi-stage build)
2. Push image to container registry
3. Create Kubernetes manifests (Deployment, Service, ConfigMap, Secret)
4. Apply with `kubectl apply -f`
5. Configure Ingress for external access
6. Set up health checks and resource limits

### Q2: How do you handle configuration in Kubernetes for .NET?
**A:**
- **ConfigMaps**: Non-sensitive config (environment, feature flags)
- **Secrets**: Sensitive data (connection strings, API keys)
- Mount as environment variables or files
- .NET reads via `AddEnvironmentVariables()` or file-based config

### Q3: How do you implement health checks for .NET in Kubernetes?
**A:**
```csharp
// Separate liveness (app alive) and readiness (ready for traffic)
builder.Services.AddHealthChecks()
    .AddCheck("liveness", () => HealthCheckResult.Healthy())
    .AddDbContextCheck<AppDbContext>("readiness");
```
Configure Kubernetes probes to hit respective endpoints.

### Q4: How do you handle database migrations in Kubernetes?
**A:**
Options:
1. **Init Container**: Run migrations before app starts
2. **Kubernetes Job**: Separate migration job
3. **Startup Check**: App checks/applies migrations on startup
4. **CI/CD Pipeline**: Run migrations as deployment step

```yaml
initContainers:
  - name: migrations
    image: myregistry/myapi:1.0.0
    command: ["dotnet", "ef", "database", "update"]
```

### Q5: How do you scale a .NET application in Kubernetes?
**A:**
- **Manual**: `kubectl scale deployment myapi --replicas=5`
- **HPA**: Auto-scale based on CPU/memory/custom metrics
- **KEDA**: Event-driven autoscaling (queue length, etc.)
- Design app for horizontal scaling (stateless, distributed cache)

### Q6: How do you debug a .NET pod that won't start?
**A:**
```bash
kubectl describe pod <pod>              # Check events
kubectl logs <pod>                      # Application logs
kubectl logs <pod> --previous           # Previous crash logs
kubectl exec -it <pod> -- /bin/sh       # Shell access

# Common issues:
# - ImagePullBackOff: Registry auth, image name
# - CrashLoopBackOff: App startup error
# - Pending: Resource constraints
```

### Q7: How do you handle secrets securely in Kubernetes?
**A:**
- Use Kubernetes Secrets (base64, but not encrypted by default)
- Enable encryption at rest for etcd
- Use external secret management (Azure Key Vault, AWS Secrets Manager)
- Sealed Secrets for GitOps
- RBAC to restrict secret access

### Q8: What resources does a typical .NET API need?
**A:**
```yaml
resources:
  requests:
    memory: "128Mi"    # Minimum for small API
    cpu: "100m"
  limits:
    memory: "512Mi"    # Depends on workload
    cpu: "500m"
```
Monitor actual usage and adjust. .NET 8+ has better memory efficiency.

---

## Useful kubectl Commands for .NET

```bash
# View logs
kubectl logs -f deployment/myapi -n myapp

# Shell into pod
kubectl exec -it deployment/myapi -n myapp -- /bin/sh

# Port forward for debugging
kubectl port-forward svc/myapi-service 5000:80 -n myapp

# Check environment variables
kubectl exec deployment/myapi -n myapp -- env | grep ASPNET

# Restart deployment (rolling)
kubectl rollout restart deployment/myapi -n myapp

# View configuration
kubectl get configmap myapi-config -n myapp -o yaml
kubectl get secret myapi-secrets -n myapp -o yaml
```

---

## Best Practices

1. **Use health checks** - Separate liveness and readiness probes
2. **Set resource limits** - Prevent memory/CPU starvation
3. **Externalize config** - ConfigMaps for config, Secrets for sensitive data
4. **Use namespaces** - Isolate environments (dev, staging, prod)
5. **Enable HPA** - Auto-scale based on demand
6. **Graceful shutdown** - Handle SIGTERM in .NET
7. **Structured logging** - JSON logs for aggregation
8. **Use Ingress** - Centralized routing and TLS termination

### Graceful Shutdown in .NET
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Host.UseDefaultServiceProvider((context, options) =>
{
    options.ValidateOnBuild = true;
});

var app = builder.Build();

// Handle shutdown
app.Lifetime.ApplicationStopping.Register(() =>
{
    // Cleanup code
});

// Configure shutdown timeout
builder.WebHost.UseShutdownTimeout(TimeSpan.FromSeconds(30));
```
