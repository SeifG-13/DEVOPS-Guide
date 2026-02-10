# Docker Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET Dockerfile

### Basic .NET Web API Dockerfile
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApi.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
USER app
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### Optimized Multi-Stage Build
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj files first (better layer caching)
COPY ["src/MyApi/MyApi.csproj", "src/MyApi/"]
COPY ["src/MyApi.Core/MyApi.Core.csproj", "src/MyApi.Core/"]
COPY ["src/MyApi.Infrastructure/MyApi.Infrastructure.csproj", "src/MyApi.Infrastructure/"]

# Restore
RUN dotnet restore "src/MyApi/MyApi.csproj"

# Copy everything and build
COPY . .
WORKDIR "/src/src/MyApi"
RUN dotnet publish -c Release -o /app/publish --no-restore

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy published app
COPY --from=build /app/publish .

# Set user and expose port
USER appuser
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

---

## .NET Docker Images

### Official .NET Images
| Image | Purpose | Size |
|-------|---------|------|
| `mcr.microsoft.com/dotnet/sdk:8.0` | Build/development | ~700MB |
| `mcr.microsoft.com/dotnet/aspnet:8.0` | ASP.NET runtime | ~210MB |
| `mcr.microsoft.com/dotnet/runtime:8.0` | Console apps | ~190MB |
| `mcr.microsoft.com/dotnet/aspnet:8.0-alpine` | Smaller runtime | ~100MB |
| `mcr.microsoft.com/dotnet/runtime-deps:8.0` | Self-contained | ~10MB |

### Choosing the Right Image
```dockerfile
# For Web APIs
FROM mcr.microsoft.com/dotnet/aspnet:8.0

# For minimal size (Alpine)
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine

# For self-contained apps
FROM mcr.microsoft.com/dotnet/runtime-deps:8.0-alpine
```

---

## Docker Compose for .NET

### Full Stack Example
```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/MyApi/Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=mydb;User=sa;Password=YourStrong@Password;TrustServerCertificate=true
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Password
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "YourStrong@Password" -Q "SELECT 1"
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  sqldata:

networks:
  app-network:
    driver: bridge
```

### Development Compose (with hot reload)
```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "5000:5000"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1
    volumes:
      - ./src:/src
      - ~/.nuget/packages:/root/.nuget/packages:ro
    command: dotnet watch run --project /src/MyApi/MyApi.csproj
```

---

## Configuration & Environment

### Environment Variables
```dockerfile
# In Dockerfile
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
```

```yaml
# In docker-compose.yml
services:
  api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=${DB_CONNECTION}
      - JWT__Secret=${JWT_SECRET}
    env_file:
      - .env
```

### Using appsettings in Docker
```csharp
// Program.cs - Configuration priority
builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddJsonFile($"appsettings.{env}.json", optional: true)
    .AddEnvironmentVariables();  // Highest priority
```

```bash
# Override via environment
docker run -e "Logging__LogLevel__Default=Debug" myapi
```

---

## Interview Q&A

### Q1: How do you containerize a .NET application?
**A:**
1. Create multi-stage Dockerfile (SDK for build, runtime for production)
2. Use appropriate base image (aspnet for web, runtime for console)
3. Copy csproj first for layer caching
4. Run as non-root user
5. Expose correct port (8080 in .NET 8+)
6. Add health checks

### Q2: What is the difference between dotnet/sdk and dotnet/aspnet images?
**A:**
- **SDK**: Contains full .NET SDK (build tools, compilers) ~700MB
- **ASP.NET**: Runtime only for web applications ~210MB
Use SDK for building, ASP.NET runtime for production.

### Q3: How do you optimize .NET Docker image size?
**A:**
1. Multi-stage builds (build in SDK, run in runtime)
2. Use Alpine variants
3. Use self-contained with trimming for smallest size
4. Copy only needed files
5. Use .dockerignore
```dockerfile
RUN dotnet publish -c Release --self-contained true \
    /p:PublishTrimmed=true /p:PublishSingleFile=true
```

### Q4: How do you handle database migrations in Docker?
**A:**
```yaml
services:
  migrations:
    build: .
    command: dotnet ef database update
    depends_on:
      db:
        condition: service_healthy

  api:
    depends_on:
      migrations:
        condition: service_completed_successfully
```
Or use EF migration bundles in init container.

### Q5: How do you debug a .NET container?
**A:**
```bash
# Attach debugger (VS Code/Rider)
docker run -e DOTNET_ENVIRONMENT=Development \
    -e ASPNETCORE_URLS=http://+:5000 \
    -p 5000:5000 myapi

# Interactive shell
docker exec -it container /bin/sh

# View logs
docker logs -f container
```

### Q6: How do you configure Kestrel in Docker?
**A:**
```csharp
// Program.cs
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(8080);
});

// Or via environment
// ASPNETCORE_URLS=http://+:8080
```

### Q7: How do you handle secrets in .NET Docker containers?
**A:**
1. Environment variables (via Docker secrets or compose)
2. Mounted secret files
3. Azure Key Vault / AWS Secrets Manager
4. Docker secrets (Swarm mode)
```csharp
builder.Configuration.AddKeyPerFile("/run/secrets", optional: true);
```

### Q8: What port does .NET 8 use by default?
**A:** .NET 8 uses port 8080 by default (changed from 80). Configure via `ASPNETCORE_URLS` or Kestrel configuration.

---

## .dockerignore for .NET
```dockerignore
**/.git
**/.vs
**/.vscode
**/bin
**/obj
**/.dockerignore
**/Dockerfile*
**/docker-compose*
**/*.md
**/*.log
**/TestResults
**/.env*
**/appsettings.*.json
!**/appsettings.json
!**/appsettings.Production.json
```

---

## Health Checks

### In Dockerfile
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

### In .NET Code
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(connectionString);

app.MapHealthChecks("/health");
```

### In Docker Compose
```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

---

## Best Practices

### Image Building
1. **Use multi-stage builds** - Separate build and runtime
2. **Leverage layer caching** - Copy csproj files first
3. **Use Alpine images** - Smaller attack surface
4. **Run as non-root** - Security requirement
5. **Use specific version tags** - Not `latest`

### Configuration
1. **Use environment variables** - 12-factor app principles
2. **Don't bake secrets** - Use runtime injection
3. **Configure proper ports** - 8080 for .NET 8+
4. **Add health checks** - For orchestration

### Development
1. **Use docker-compose** - Orchestrate dependencies
2. **Volume mount for hot reload** - Faster development
3. **Use .dockerignore** - Reduce build context
4. **Separate dev/prod configs** - Different compose files

---

## Common Docker Commands for .NET

```bash
# Build and run
docker build -t myapi .
docker run -p 5000:8080 -e ASPNETCORE_ENVIRONMENT=Development myapi

# Debug
docker run -it --entrypoint /bin/sh myapi
docker logs -f container_name

# Compose
docker-compose up --build
docker-compose logs -f api
docker-compose exec api /bin/sh

# Clean up
docker system prune -a
docker volume prune
```
