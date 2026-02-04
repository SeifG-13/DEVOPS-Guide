# Kubernetes ConfigMaps

## What is a ConfigMap?

A ConfigMap stores non-confidential configuration data as key-value pairs.

```
┌─────────────────────────────────────────────────────────────────┐
│                     ConfigMap Concept                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ConfigMap (app-config)                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  data:                                                   │   │
│   │    DATABASE_HOST: "mysql.default"                       │   │
│   │    DATABASE_PORT: "3306"                                │   │
│   │    LOG_LEVEL: "info"                                    │   │
│   │    app.properties: |                                    │   │
│   │      server.port=8080                                   │   │
│   │      debug.enabled=true                                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│              ┌─────────────┼─────────────┐                      │
│              ▼             ▼             ▼                      │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│   │  Env Vars    │ │ Volume Mount │ │  Command Args │           │
│   │  in Pod      │ │  as Files    │ │  in Pod      │           │
│   └──────────────┘ └──────────────┘ └──────────────┘           │
│                                                                  │
│   Use Cases:                                                    │
│   • Application configuration                                   │
│   • Environment-specific settings                               │
│   • Configuration files                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Creating ConfigMaps

### From Literal Values

```bash
# Create from literals
kubectl create configmap app-config \
    --from-literal=DATABASE_HOST=mysql.default \
    --from-literal=DATABASE_PORT=3306 \
    --from-literal=LOG_LEVEL=info

# View ConfigMap
kubectl get configmap app-config -o yaml
```

### From Files

```bash
# Create from file
kubectl create configmap app-config --from-file=config.properties

# Create from multiple files
kubectl create configmap app-config \
    --from-file=app.properties \
    --from-file=database.properties

# Create from directory
kubectl create configmap app-config --from-file=./configs/

# Create with specific key name
kubectl create configmap app-config --from-file=myconfig=config.properties
```

### From YAML

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value pairs
  DATABASE_HOST: "mysql.default"
  DATABASE_PORT: "3306"
  LOG_LEVEL: "info"

  # Multi-line configuration file
  app.properties: |
    server.port=8080
    server.name=myapp
    debug.enabled=false

  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
        }
    }
```

```bash
kubectl apply -f configmap.yaml
```

### Binary Data

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: binary-config
binaryData:
  # Base64 encoded binary data
  logo.png: iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==
```

## Using ConfigMaps in Pods

### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Single key from ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_PORT
```

### All Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config
    # Optional prefix
    - configMapRef:
        name: app-config
        prefix: APP_
```

### As Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

Files created in `/etc/config/`:
- `/etc/config/DATABASE_HOST`
- `/etc/config/DATABASE_PORT`
- `/etc/config/app.properties`

### Mount Specific Keys

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: application.properties
      - key: nginx.conf
        path: nginx/nginx.conf
```

### Mount to Specific Path

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf  # Mount single file, not directory
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
```

### Set File Permissions

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    defaultMode: 0644  # File permissions
    items:
    - key: script.sh
      path: script.sh
      mode: 0755  # Executable
```

## ConfigMap with Deployment

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  config.json: |
    {
      "database": {
        "host": "db.example.com",
        "port": 5432
      },
      "cache": {
        "enabled": true,
        "ttl": 3600
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: webapp:v1
        envFrom:
        - configMapRef:
            name: webapp-config
        volumeMounts:
        - name: config
          mountPath: /app/config
      volumes:
      - name: config
        configMap:
          name: webapp-config
          items:
          - key: config.json
            path: config.json
```

## Updating ConfigMaps

### Update Methods

```bash
# Edit ConfigMap
kubectl edit configmap app-config

# Replace ConfigMap
kubectl create configmap app-config --from-file=config.properties --dry-run=client -o yaml | kubectl apply -f -

# Patch ConfigMap
kubectl patch configmap app-config --type merge -p '{"data":{"LOG_LEVEL":"debug"}}'
```

### Update Behavior

```
┌─────────────────────────────────────────────────────────────────┐
│                ConfigMap Update Behavior                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Method                    Behavior                            │
│   ─────────────────────────────────────────────────────         │
│   Environment Variables     Pod restart required                │
│   Volume Mounts            Auto-updated (may take ~1 minute)    │
│   subPath Mounts           NOT auto-updated                     │
│                                                                  │
│   To force pod restart after ConfigMap update:                  │
│   kubectl rollout restart deployment/myapp                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Immutable ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true  # Cannot be modified after creation
data:
  DATABASE_HOST: "mysql.default"
```

Benefits of immutable ConfigMaps:
- Protects against accidental updates
- Improves cluster performance (no watches)

## Managing ConfigMaps

```bash
# List ConfigMaps
kubectl get configmaps
kubectl get cm

# Describe ConfigMap
kubectl describe configmap app-config

# Get ConfigMap YAML
kubectl get configmap app-config -o yaml

# Delete ConfigMap
kubectl delete configmap app-config

# Delete from file
kubectl delete -f configmap.yaml
```

## Best Practices

### Naming and Organization

```yaml
# Use descriptive names
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-frontend-config
  labels:
    app: webapp
    component: frontend
    environment: production
data:
  # ...
```

### Separate Configuration by Purpose

```yaml
# Database configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  host: "postgres.default"
  port: "5432"
  database: "myapp"
---
# Application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  log_level: "info"
  cache_enabled: "true"
---
# Feature flags
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  new_ui: "true"
  beta_features: "false"
```

### Version ConfigMaps

```yaml
# Include version in name for rollback capability
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
data:
  # New configuration
```

```yaml
# Reference specific version in Deployment
volumes:
- name: config
  configMap:
    name: app-config-v2
```

## Practical Examples

### Nginx Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 80;
            location / {
                root /usr/share/nginx/html;
                index index.html;
            }
            location /api {
                proxy_pass http://api-service:8080;
            }
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
```

### Application Properties

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-config
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/mydb
        username: myuser
      jpa:
        hibernate:
          ddl-auto: update
    server:
      port: 8080
    logging:
      level:
        root: INFO
        com.myapp: DEBUG
```

## Quick Reference

### Creation Methods

| Method | Command |
|--------|---------|
| Literal | `kubectl create cm name --from-literal=key=value` |
| File | `kubectl create cm name --from-file=file.txt` |
| Directory | `kubectl create cm name --from-file=./configs/` |
| YAML | `kubectl apply -f configmap.yaml` |

### Usage Methods

| Method | Use Case |
|--------|----------|
| `env.valueFrom` | Single environment variable |
| `envFrom` | All keys as env vars |
| `volume` | Mount as files |
| `subPath` | Mount single file |

### Key Points

- ConfigMaps store non-sensitive configuration
- Use Secrets for sensitive data
- Volume mounts auto-update, env vars don't
- Consider immutable ConfigMaps for stability
- Version ConfigMaps for easy rollbacks

---

**Previous:** [09-namespaces.md](09-namespaces.md) | **Next:** [11-secrets.md](11-secrets.md)
