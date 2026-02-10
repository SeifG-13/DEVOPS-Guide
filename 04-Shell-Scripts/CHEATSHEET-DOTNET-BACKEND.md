# Shell Scripting Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET CLI in Scripts

### Build & Run Scripts
```bash
#!/bin/bash
set -euo pipefail

PROJECT_DIR="/var/www/myapp"
cd "$PROJECT_DIR"

# Restore and build
dotnet restore
dotnet build --configuration Release --no-restore

# Run with environment
ASPNETCORE_ENVIRONMENT=Production dotnet run
```

### Publish Script
```bash
#!/bin/bash
set -euo pipefail

OUTPUT_DIR="./publish"
CONFIGURATION="Release"
RUNTIME="linux-x64"

# Clean previous build
rm -rf "$OUTPUT_DIR"

# Publish self-contained
dotnet publish \
    --configuration "$CONFIGURATION" \
    --runtime "$RUNTIME" \
    --self-contained true \
    --output "$OUTPUT_DIR"

echo "Published to $OUTPUT_DIR"
```

### Test Script
```bash
#!/bin/bash
set -euo pipefail

# Run all tests with coverage
dotnet test \
    --configuration Release \
    --collect:"XPlat Code Coverage" \
    --results-directory ./TestResults \
    --logger "trx;LogFileName=results.trx"

# Check exit code
if [[ $? -eq 0 ]]; then
    echo "All tests passed!"
else
    echo "Tests failed!"
    exit 1
fi
```

---

## Deployment Scripts

### Full Deployment Script
```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapi"
DEPLOY_DIR="/var/www/$APP_NAME"
SERVICE_NAME="$APP_NAME.service"
BACKUP_DIR="/var/backups/$APP_NAME"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# Backup current deployment
backup() {
    log "Creating backup..."
    mkdir -p "$BACKUP_DIR"
    if [[ -d "$DEPLOY_DIR" ]]; then
        tar -czf "$BACKUP_DIR/backup_$(date +%Y%m%d_%H%M%S).tar.gz" \
            -C "$DEPLOY_DIR" .
    fi
}

# Deploy new version
deploy() {
    log "Deploying new version..."

    # Stop service
    sudo systemctl stop "$SERVICE_NAME" || true

    # Copy new files
    cp -r ./publish/* "$DEPLOY_DIR/"

    # Set permissions
    sudo chown -R www-data:www-data "$DEPLOY_DIR"
    sudo chmod -R 755 "$DEPLOY_DIR"

    # Start service
    sudo systemctl start "$SERVICE_NAME"

    log "Deployment complete!"
}

# Health check
health_check() {
    log "Running health check..."
    local max_attempts=10
    local attempt=1

    while [[ $attempt -le $max_attempts ]]; do
        if curl -sf http://localhost:5000/health > /dev/null; then
            log "Health check passed!"
            return 0
        fi
        log "Attempt $attempt/$max_attempts - waiting..."
        sleep 3
        ((attempt++))
    done

    log "Health check failed!"
    return 1
}

# Rollback
rollback() {
    log "Rolling back..."
    local latest_backup=$(ls -t "$BACKUP_DIR"/*.tar.gz 2>/dev/null | head -1)

    if [[ -n "$latest_backup" ]]; then
        sudo systemctl stop "$SERVICE_NAME"
        rm -rf "$DEPLOY_DIR"/*
        tar -xzf "$latest_backup" -C "$DEPLOY_DIR"
        sudo systemctl start "$SERVICE_NAME"
        log "Rollback complete!"
    else
        log "No backup found!"
        exit 1
    fi
}

# Main
case "${1:-deploy}" in
    deploy)
        backup
        deploy
        if ! health_check; then
            rollback
            exit 1
        fi
        ;;
    rollback)
        rollback
        ;;
    *)
        echo "Usage: $0 {deploy|rollback}"
        exit 1
        ;;
esac
```

### Database Migration Script
```bash
#!/bin/bash
set -euo pipefail

PROJECT_DIR="/var/www/myapp"
CONNECTION_STRING="${DB_CONNECTION_STRING:-}"

if [[ -z "$CONNECTION_STRING" ]]; then
    echo "Error: DB_CONNECTION_STRING not set"
    exit 1
fi

cd "$PROJECT_DIR"

echo "Running database migrations..."
dotnet ef database update \
    --connection "$CONNECTION_STRING" \
    --project src/MyApp.Data

echo "Migrations complete!"
```

---

## Service Management Scripts

### Systemd Service Setup
```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapi"
APP_DIR="/var/www/$APP_NAME"
APP_USER="www-data"

# Create service file
cat > "/etc/systemd/system/${APP_NAME}.service" << EOF
[Unit]
Description=$APP_NAME .NET Web API
After=network.target

[Service]
WorkingDirectory=$APP_DIR
ExecStart=/usr/bin/dotnet $APP_DIR/${APP_NAME}.dll
Restart=always
RestartSec=10
User=$APP_USER
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
EOF

# Reload and enable
systemctl daemon-reload
systemctl enable "$APP_NAME"
systemctl start "$APP_NAME"

echo "Service $APP_NAME configured and started"
```

### Service Control Script
```bash
#!/bin/bash
SERVICE="myapi.service"

case "$1" in
    start)
        sudo systemctl start "$SERVICE"
        ;;
    stop)
        sudo systemctl stop "$SERVICE"
        ;;
    restart)
        sudo systemctl restart "$SERVICE"
        ;;
    status)
        sudo systemctl status "$SERVICE"
        ;;
    logs)
        sudo journalctl -u "$SERVICE" -f
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status|logs}"
        exit 1
        ;;
esac
```

---

## Environment & Configuration Scripts

### Environment Setup Script
```bash
#!/bin/bash
set -euo pipefail

# Create environment file
cat > /etc/myapp/env << 'EOF'
ASPNETCORE_ENVIRONMENT=Production
ConnectionStrings__DefaultConnection=Server=localhost;Database=mydb;...
JWT__Secret=your-secret-key
EOF

# Secure the file
chmod 600 /etc/myapp/env
chown root:www-data /etc/myapp/env

echo "Environment file created"
```

### Configuration Validation Script
```bash
#!/bin/bash
set -euo pipefail

CONFIG_FILE="appsettings.Production.json"

# Check if config exists
if [[ ! -f "$CONFIG_FILE" ]]; then
    echo "Error: $CONFIG_FILE not found"
    exit 1
fi

# Validate JSON syntax
if ! jq empty "$CONFIG_FILE" 2>/dev/null; then
    echo "Error: Invalid JSON in $CONFIG_FILE"
    exit 1
fi

# Check required keys
required_keys=("ConnectionStrings.DefaultConnection" "Logging.LogLevel.Default")

for key in "${required_keys[@]}"; do
    value=$(jq -r ".${key}" "$CONFIG_FILE")
    if [[ "$value" == "null" ]]; then
        echo "Warning: Missing required key: $key"
    fi
done

echo "Configuration validation complete"
```

---

## CI/CD Scripts

### Build Pipeline Script
```bash
#!/bin/bash
set -euo pipefail

echo "=== .NET Build Pipeline ==="

# Restore
echo "Restoring packages..."
dotnet restore

# Build
echo "Building..."
dotnet build --no-restore --configuration Release

# Test
echo "Running tests..."
dotnet test --no-build --configuration Release

# Publish
echo "Publishing..."
dotnet publish --no-build --configuration Release --output ./publish

echo "=== Build Complete ==="
```

### Docker Build Script
```bash
#!/bin/bash
set -euo pipefail

IMAGE_NAME="myapi"
VERSION="${1:-latest}"
REGISTRY="${DOCKER_REGISTRY:-}"

# Build image
docker build -t "${IMAGE_NAME}:${VERSION}" .

# Tag and push if registry is set
if [[ -n "$REGISTRY" ]]; then
    docker tag "${IMAGE_NAME}:${VERSION}" "${REGISTRY}/${IMAGE_NAME}:${VERSION}"
    docker push "${REGISTRY}/${IMAGE_NAME}:${VERSION}"
    echo "Pushed to ${REGISTRY}/${IMAGE_NAME}:${VERSION}"
fi
```

---

## Interview Q&A

### Q1: How do you automate .NET deployments with shell scripts?
**A:**
1. Build: `dotnet publish -c Release -o ./publish`
2. Stop service: `systemctl stop myapp`
3. Backup current version
4. Copy new files to deployment directory
5. Set permissions
6. Start service: `systemctl start myapp`
7. Health check to verify deployment

### Q2: How do you handle secrets in deployment scripts?
**A:**
- Use environment variables (not hardcoded)
- Read from secure files with restricted permissions
- Use Azure Key Vault / AWS Secrets Manager
- Never log or echo secrets
```bash
# Read from environment
DB_PASSWORD="${DB_PASSWORD:?Error: DB_PASSWORD not set}"

# Read from file
DB_PASSWORD=$(cat /run/secrets/db_password)
```

### Q3: How do you implement zero-downtime deployments?
**A:**
```bash
# Blue-green deployment
# Deploy to inactive environment
deploy_to "$INACTIVE_ENV"
health_check "$INACTIVE_ENV"

# Switch traffic
switch_load_balancer "$INACTIVE_ENV"

# Keep old environment for rollback
```

### Q4: How do you monitor a .NET service from a script?
**A:**
```bash
# Check if process is running
if pgrep -f "dotnet.*MyApp.dll" > /dev/null; then
    echo "App is running"
fi

# Check HTTP endpoint
if curl -sf http://localhost:5000/health; then
    echo "App is healthy"
fi

# Check systemd status
if systemctl is-active --quiet myapp; then
    echo "Service is active"
fi
```

### Q5: How do you run .NET migrations in a script?
**A:**
```bash
# Using EF Core CLI
dotnet ef database update --project src/MyApp

# Using migration bundle (recommended for production)
./efbundle --connection "$CONNECTION_STRING"
```

---

## Utility Scripts

### Log Tail with Filtering
```bash
#!/bin/bash
# Tail .NET logs and filter by level

LOG_LEVEL="${1:-Error}"

journalctl -u myapp -f | while read -r line; do
    if echo "$line" | grep -q "\"Level\":\"$LOG_LEVEL\""; then
        echo "$line"
    fi
done
```

### Memory Monitor
```bash
#!/bin/bash
# Monitor .NET app memory usage

APP_NAME="MyApp"
THRESHOLD_MB=500

while true; do
    mem=$(ps -o rss= -C dotnet | awk '{sum+=$1} END {print sum/1024}')

    if (( $(echo "$mem > $THRESHOLD_MB" | bc -l) )); then
        echo "Warning: Memory usage is ${mem}MB (threshold: ${THRESHOLD_MB}MB)"
        # Optional: restart service
    fi

    sleep 60
done
```

### Cleanup Script
```bash
#!/bin/bash
# Clean up old deployments and logs

DEPLOY_BACKUP_DIR="/var/backups/myapp"
LOG_DIR="/var/log/myapp"
DAYS_TO_KEEP=7

# Clean old backups
find "$DEPLOY_BACKUP_DIR" -name "*.tar.gz" -mtime +$DAYS_TO_KEEP -delete

# Clean old logs
find "$LOG_DIR" -name "*.log" -mtime +$DAYS_TO_KEEP -delete

# Clean temp files
rm -rf /tmp/myapp-*

echo "Cleanup complete"
```

---

## Best Practices

1. **Use set -euo pipefail** - Fail early on errors
2. **Validate inputs** - Check environment variables exist
3. **Implement health checks** - Verify deployment success
4. **Keep backups** - Always backup before deployment
5. **Log everything** - Timestamp and context for debugging
6. **Use systemd** - Proper service management
7. **Secure secrets** - Never hardcode credentials
8. **Test scripts** - Run in staging first
