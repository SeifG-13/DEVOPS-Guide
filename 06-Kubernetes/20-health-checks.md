# Kubernetes Health Checks

## Types of Probes

```
┌─────────────────────────────────────────────────────────────────┐
│                    Probe Types                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Liveness Probe                                                │
│   └── Is the container alive?                                   │
│   └── Failed → Container is restarted                          │
│                                                                  │
│   Readiness Probe                                               │
│   └── Is the container ready to serve traffic?                 │
│   └── Failed → Removed from Service endpoints                  │
│                                                                  │
│   Startup Probe                                                 │
│   └── Has the container started successfully?                  │
│   └── Failed → Container is restarted                          │
│   └── Disables liveness/readiness until successful             │
│                                                                  │
│   Timeline:                                                     │
│   Start ──► Startup Probe ──► Liveness + Readiness Probes      │
│             (initialization)    (continuous)                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Probe Mechanisms

### HTTP GET Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: http-probe-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
```

### TCP Socket Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tcp-probe-pod
spec:
  containers:
  - name: redis
    image: redis:7
    ports:
    - containerPort: 6379
    livenessProbe:
      tcpSocket:
        port: 6379
      initialDelaySeconds: 15
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 6379
      initialDelaySeconds: 5
      periodSeconds: 5
```

### Exec Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: exec-probe-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

### gRPC Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grpc-probe-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 50051
    livenessProbe:
      grpc:
        port: 50051
        service: health
      initialDelaySeconds: 10
      periodSeconds: 10
```

## Probe Configuration

### All Parameters

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15  # Wait before first probe
  periodSeconds: 10        # How often to probe
  timeoutSeconds: 3        # Timeout for probe
  successThreshold: 1      # Successes to mark healthy
  failureThreshold: 3      # Failures to mark unhealthy
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `initialDelaySeconds` | 0 | Delay before first probe |
| `periodSeconds` | 10 | Time between probes |
| `timeoutSeconds` | 1 | Probe timeout |
| `successThreshold` | 1 | Successes required |
| `failureThreshold` | 3 | Failures before action |

## Liveness Probe

Detects if container is in a broken state and needs restart.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
```

### When to Use Liveness

```
✓ Application hangs but process is running
✓ Deadlock detection
✓ Memory leaks causing unresponsiveness
✓ Application in unrecoverable state

✗ Don't use for startup delays (use startupProbe)
✗ Don't use for dependency checks (use readinessProbe)
```

## Readiness Probe

Determines if container can receive traffic.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1
      failureThreshold: 3
```

### When to Use Readiness

```
✓ Application needs time to load data
✓ Temporary unavailability (heavy load)
✓ Dependency not available
✓ Database connection not ready

Effect:
• Failed readiness → Pod removed from Service endpoints
• No traffic sent to unready pods
• Pod NOT restarted (unlike liveness failure)
```

## Startup Probe

For slow-starting containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 30  # 30 * 10 = 300 seconds max startup
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
```

### Startup Probe Behavior

```
┌─────────────────────────────────────────────────────────────────┐
│                  Startup Probe Behavior                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Container Start                                               │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                   Startup Probe                          │  │
│   │                                                          │  │
│   │   Liveness and Readiness probes are DISABLED            │  │
│   │   until startup probe succeeds                          │  │
│   │                                                          │  │
│   │   Success → Enable liveness/readiness                   │  │
│   │   Failure → Restart container                           │  │
│   └─────────────────────────────────────────────────────────┘  │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │       Liveness + Readiness Probes (enabled)             │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Complete Example

```yaml
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
        ports:
        - containerPort: 8080

        # Startup probe - for slow starting apps
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 30

        # Liveness probe - restart if unhealthy
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3

        # Readiness probe - stop traffic if not ready
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

## Health Endpoint Examples

### Basic Health Endpoint (Go)

```go
func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}

func readyHandler(w http.ResponseWriter, r *http.Request) {
    // Check dependencies
    if dbConnected && cacheReady {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Ready"))
    } else {
        w.WriteHeader(http.StatusServiceUnavailable)
        w.Write([]byte("Not Ready"))
    }
}
```

### Health Endpoint (Python Flask)

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/healthz')
def health():
    return jsonify({"status": "healthy"}), 200

@app.route('/ready')
def ready():
    if check_database() and check_cache():
        return jsonify({"status": "ready"}), 200
    return jsonify({"status": "not ready"}), 503
```

### Health Endpoint (Node.js)

```javascript
app.get('/healthz', (req, res) => {
  res.status(200).send('OK');
});

app.get('/ready', async (req, res) => {
  try {
    await checkDatabase();
    await checkRedis();
    res.status(200).send('Ready');
  } catch (error) {
    res.status(503).send('Not Ready');
  }
});
```

## Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                  Health Check Best Practices                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Liveness Probe:                                               │
│   ✓ Keep it simple and fast                                    │
│   ✓ Don't check dependencies                                   │
│   ✓ Only check if app is fundamentally broken                  │
│   ✗ Don't include database connectivity                        │
│                                                                  │
│   Readiness Probe:                                              │
│   ✓ Check all dependencies                                     │
│   ✓ Verify app can handle requests                             │
│   ✓ Use shorter intervals than liveness                        │
│                                                                  │
│   Startup Probe:                                                │
│   ✓ Use for slow-starting apps (Java, ML models)              │
│   ✓ Give enough time with failureThreshold × periodSeconds    │
│                                                                  │
│   General:                                                      │
│   ✓ Always set probes in production                            │
│   ✓ Test probe behavior before deployment                      │
│   ✓ Monitor probe failures in alerting                         │
│   ✓ Set appropriate timeouts                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Debugging Probes

```bash
# Check pod events for probe failures
kubectl describe pod mypod

# Events:
#   Type     Reason     Message
#   ----     ------     -------
#   Warning  Unhealthy  Liveness probe failed: ...
#   Warning  Unhealthy  Readiness probe failed: ...

# Test probe endpoint manually
kubectl exec mypod -- curl -v http://localhost:8080/healthz

# Check container restarts
kubectl get pods
# RESTARTS column shows liveness failures

# Check ready status
kubectl get pods
# READY column shows readiness status (1/1 or 0/1)
```

## Quick Reference

### Probe Types

| Probe | Purpose | Action on Failure |
|-------|---------|-------------------|
| Liveness | Is container alive? | Restart container |
| Readiness | Can serve traffic? | Remove from endpoints |
| Startup | Has container started? | Restart container |

### Probe Mechanisms

| Mechanism | Use Case |
|-----------|----------|
| HTTP GET | Web applications |
| TCP Socket | Databases, TCP services |
| Exec | Custom scripts, file checks |
| gRPC | gRPC services |

### Key Parameters

| Parameter | Description |
|-----------|-------------|
| `initialDelaySeconds` | Wait before first probe |
| `periodSeconds` | Time between probes |
| `timeoutSeconds` | Probe timeout |
| `failureThreshold` | Failures before action |
| `successThreshold` | Successes to mark healthy |

---

**Previous:** [19-resource-management.md](19-resource-management.md) | **Next:** [21-troubleshooting.md](21-troubleshooting.md)
