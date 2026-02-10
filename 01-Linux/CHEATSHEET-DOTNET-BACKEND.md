# Linux Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET on Linux

### .NET CLI Essentials
```bash
dotnet --version                    # Check .NET version
dotnet --list-sdks                  # List installed SDKs
dotnet --list-runtimes              # List installed runtimes
dotnet new webapi -n MyApi          # Create new Web API
dotnet build                        # Build project
dotnet run                          # Run application
dotnet publish -c Release           # Publish for deployment
dotnet watch run                    # Hot reload development
```

### Running .NET Apps on Linux
```bash
# Run as background service
nohup dotnet MyApp.dll &

# Run with specific URLs
dotnet MyApp.dll --urls "http://0.0.0.0:5000"

# Run with environment
ASPNETCORE_ENVIRONMENT=Production dotnet MyApp.dll

# Check if app is running
ps aux | grep dotnet
lsof -i :5000
```

### Systemd Service for .NET App
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My .NET Web API
After=network.target

[Service]
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/dotnet /var/www/myapp/MyApp.dll
Restart=always
RestartSec=10
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
journalctl -u myapp -f              # View logs
```

### File Permissions for .NET Apps
```bash
# Set proper ownership
sudo chown -R www-data:www-data /var/www/myapp

# Set permissions (no execute needed for DLLs)
sudo chmod -R 755 /var/www/myapp
sudo chmod 644 /var/www/myapp/*.dll
sudo chmod 644 /var/www/myapp/appsettings*.json
```

### Environment Variables
```bash
# Set temporarily
export ConnectionStrings__DefaultConnection="Server=..."
export ASPNETCORE_URLS="http://+:5000"

# Set permanently in ~/.bashrc or /etc/environment
echo 'export ASPNETCORE_ENVIRONMENT=Production' >> ~/.bashrc

# In systemd service
Environment=ASPNETCORE_ENVIRONMENT=Production
EnvironmentFile=/etc/myapp/env
```

---

## Deployment Scenarios

### Self-Contained Deployment
```bash
# Publish self-contained for Linux
dotnet publish -c Release -r linux-x64 --self-contained true

# Make executable
chmod +x ./MyApp
./MyApp
```

### Framework-Dependent Deployment
```bash
# Publish framework-dependent
dotnet publish -c Release

# Requires .NET runtime on server
sudo apt install dotnet-runtime-8.0
dotnet MyApp.dll
```

### Reverse Proxy with Nginx
```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Troubleshooting .NET on Linux

### Common Commands
```bash
# Check if app is listening
ss -tulnp | grep 5000
curl -v http://localhost:5000/health

# Check logs
journalctl -u myapp --since "1 hour ago"
tail -f /var/log/myapp/app.log

# Debug startup issues
dotnet MyApp.dll --verbose

# Check dependencies
ldd MyApp                           # For self-contained
dotnet --info                       # Runtime info
```

### Memory and Performance
```bash
# Monitor .NET process
top -p $(pgrep -f "dotnet.*MyApp")

# Memory usage
ps -o pid,rss,vsz,command -p $(pgrep -f MyApp)

# Generate dump for analysis
dotnet-dump collect -p <PID>
dotnet-dump analyze <dump-file>

# CPU profiling
dotnet-trace collect -p <PID>
```

---

## Interview Q&A

### Q1: How do you deploy a .NET application to Linux?
**A:**
1. Publish app: `dotnet publish -c Release`
2. Copy to server via SCP/SFTP
3. Install .NET runtime (if framework-dependent)
4. Create systemd service for process management
5. Configure reverse proxy (Nginx/Apache)
6. Set up SSL with Let's Encrypt

### Q2: What's the difference between self-contained and framework-dependent deployment?
**A:**
- **Framework-dependent**: Smaller size, requires .NET runtime on server, easier updates
- **Self-contained**: Larger size, includes runtime, no dependencies, version isolation

### Q3: How do you manage configuration in Linux environments?
**A:**
- Environment variables (12-factor app approach)
- `appsettings.Production.json` for environment-specific settings
- `/etc/myapp/` for sensitive configs (with proper permissions)
- Secret managers (Azure Key Vault, HashiCorp Vault)

### Q4: How do you troubleshoot a .NET app that won't start on Linux?
**A:**
1. Check `journalctl -u myapp` for service logs
2. Run manually: `dotnet MyApp.dll` to see errors
3. Verify permissions on files and directories
4. Check if port is available: `ss -tulnp | grep 5000`
5. Verify environment variables are set
6. Check .NET runtime is installed: `dotnet --info`

### Q5: How do you handle logging in .NET on Linux?
**A:**
- Console logging (captured by journald for systemd services)
- File logging with Serilog/NLog to `/var/log/myapp/`
- Structured logging with JSON for ELK/Prometheus
- Configure log rotation with logrotate

### Q6: What Linux permissions does a .NET app need?
**A:**
- Read access to application directory
- Write access to logs/temp directories
- Execute permission for self-contained apps
- Network binding capability (or run behind reverse proxy)
- Non-root user for security (www-data, appuser)

### Q7: How do you secure a .NET API on Linux?
**A:**
- Run as non-root user
- Use reverse proxy (Nginx) for SSL termination
- Firewall rules (ufw/iptables)
- Disable directory listing
- Proper file permissions (644 for files, 755 for directories)
- Environment-based secrets management

---

## Essential Linux Paths for .NET

| Path | Purpose |
|------|---------|
| `/var/www/myapp/` | Application files |
| `/var/log/myapp/` | Application logs |
| `/etc/myapp/` | Configuration files |
| `/tmp/` | Temporary files |
| `/usr/share/dotnet/` | .NET SDK/Runtime |
| `~/.dotnet/` | User .NET tools |
| `~/.nuget/` | NuGet cache |

---

## Health Check Endpoint Example
```csharp
// Program.cs
app.MapHealthChecks("/health");

// Check from Linux
curl http://localhost:5000/health
watch -n 5 'curl -s http://localhost:5000/health'
```

---

## Best Practices

1. **Use systemd** - Proper process management and auto-restart
2. **Run as non-root** - Create dedicated service account
3. **Use reverse proxy** - Nginx/Apache for SSL and static files
4. **Externalize config** - Environment variables for secrets
5. **Structured logging** - JSON logs for parsing
6. **Health endpoints** - For load balancers and monitoring
7. **Graceful shutdown** - Handle SIGTERM properly
8. **Log rotation** - Prevent disk space issues
