# Deployment Strategies

## Deployment Strategy Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                 Deployment Strategies Comparison                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Strategy         Risk    Downtime   Rollback   Complexity    │
│   ─────────────────────────────────────────────────────────     │
│   Recreate         High    Yes        Slow       Low           │
│   Rolling          Medium  No         Medium     Medium        │
│   Blue-Green       Low     No         Fast       Medium        │
│   Canary           Low     No         Fast       High          │
│   A/B Testing      Low     No         Fast       High          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Recreate Deployment

Stops all old instances before starting new ones.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Recreate Deployment                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Time ─────────────────────────────────────────────────────►   │
│                                                                  │
│   Before:    [v1] [v1] [v1]                                     │
│                    ↓                                            │
│   Stopping:  [ - ] [ - ] [ - ]   ← Downtime                    │
│                    ↓                                            │
│   After:     [v2] [v2] [v2]                                     │
│                                                                  │
│   Use when:                                                     │
│   • Application doesn't support running multiple versions       │
│   • Development/test environments                               │
│   • Stateful applications with data migration                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Recreate

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2
```

## Rolling Deployment

Gradually replaces old instances with new ones.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Rolling Deployment                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Time ─────────────────────────────────────────────────────►   │
│                                                                  │
│   Step 1:    [v1] [v1] [v1] [v1]                               │
│   Step 2:    [v2] [v1] [v1] [v1]                               │
│   Step 3:    [v2] [v2] [v1] [v1]                               │
│   Step 4:    [v2] [v2] [v2] [v1]                               │
│   Step 5:    [v2] [v2] [v2] [v2]                               │
│                                                                  │
│   Benefits:                                                     │
│   • Zero downtime                                               │
│   • Gradual transition                                          │
│   • Can stop if issues detected                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired
      maxUnavailable: 0  # Zero downtime
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Jenkins Rolling Deployment

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                script {
                    def replicas = 4
                    def batchSize = 1

                    for (int i = 0; i < replicas; i += batchSize) {
                        sh """
                            kubectl set image deployment/myapp \
                                myapp=myapp:${BUILD_NUMBER} \
                                --record
                        """
                        sh "kubectl rollout status deployment/myapp"
                    }
                }
            }
        }
    }
}
```

## Blue-Green Deployment

Runs two identical environments, switching traffic instantly.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Blue-Green Deployment                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    Load Balancer                                │
│                         │                                       │
│            ┌────────────┴────────────┐                         │
│            │                         │                         │
│            ▼                         ▼                         │
│   ┌─────────────────┐     ┌─────────────────┐                 │
│   │   Blue (v1)     │     │   Green (v2)    │                 │
│   │   [v1] [v1]     │     │   [v2] [v2]     │                 │
│   │   ← ACTIVE      │     │   ← STANDBY     │                 │
│   └─────────────────┘     └─────────────────┘                 │
│                                                                  │
│   After switch:                                                 │
│   ┌─────────────────┐     ┌─────────────────┐                 │
│   │   Blue (v1)     │     │   Green (v2)    │                 │
│   │   ← STANDBY     │     │   ← ACTIVE      │                 │
│   └─────────────────┘     └─────────────────┘                 │
│                                                                  │
│   Benefits:                                                     │
│   • Instant rollback                                           │
│   • Test in production before switch                           │
│   • Zero downtime                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Blue-Green

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:v1
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v2
---
# service.yaml - Switch by changing selector
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: green  # Switch: blue ↔ green
  ports:
  - port: 80
    targetPort: 8080
```

### Blue-Green with Script

```bash
#!/bin/bash
# blue-green-deploy.sh

CURRENT=$(kubectl get svc myapp -o jsonpath='{.spec.selector.version}')
NEW_VERSION=$1

if [ "$CURRENT" = "blue" ]; then
    TARGET="green"
else
    TARGET="blue"
fi

echo "Current: $CURRENT, Deploying to: $TARGET"

# Deploy new version
kubectl set image deployment/myapp-$TARGET myapp=myapp:$NEW_VERSION
kubectl rollout status deployment/myapp-$TARGET

# Run smoke tests
./smoke-tests.sh myapp-$TARGET

if [ $? -eq 0 ]; then
    # Switch traffic
    kubectl patch svc myapp -p "{\"spec\":{\"selector\":{\"version\":\"$TARGET\"}}}"
    echo "Switched traffic to $TARGET"
else
    echo "Smoke tests failed, aborting"
    exit 1
fi
```

## Canary Deployment

Gradually shifts traffic to new version.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Canary Deployment                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Stage 1: 5% traffic to v2                                     │
│   ┌─────────────────────────────────────┐                       │
│   │ [v1] [v1] [v1] [v1] [v1] ... [v2]   │  95% → v1, 5% → v2   │
│   └─────────────────────────────────────┘                       │
│                                                                  │
│   Stage 2: 25% traffic to v2                                    │
│   ┌─────────────────────────────────────┐                       │
│   │ [v1] [v1] [v1] [v2] [v2] [v2] [v2]  │  75% → v1, 25% → v2  │
│   └─────────────────────────────────────┘                       │
│                                                                  │
│   Stage 3: 100% traffic to v2                                   │
│   ┌─────────────────────────────────────┐                       │
│   │ [v2] [v2] [v2] [v2] [v2] [v2] [v2]  │  100% → v2           │
│   └─────────────────────────────────────┘                       │
│                                                                  │
│   Monitor at each stage:                                        │
│   • Error rates                                                 │
│   • Response times                                              │
│   • Business metrics                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Canary with Ingress

```yaml
# Canary deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: myapp:v2
---
# Canary service
apiVersion: v1
kind: Service
metadata:
  name: myapp-canary
spec:
  selector:
    app: myapp
    version: canary
  ports:
  - port: 80
---
# Nginx Ingress canary
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% traffic
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
```

### Istio Canary

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: stable
    labels:
      version: v1
  - name: canary
    labels:
      version: v2
```

### Argo Rollouts Canary

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 5m}
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 80
      - pause: {duration: 5m}
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 2
  selector:
    matchLabels:
      app: myapp
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2
```

## A/B Testing

Route traffic based on user characteristics.

```
┌─────────────────────────────────────────────────────────────────┐
│                    A/B Testing Deployment                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User Request                                                  │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────────┐                                          │
│   │  Load Balancer  │                                          │
│   │  + A/B Logic    │                                          │
│   └────────┬────────┘                                          │
│            │                                                    │
│   ┌────────┴────────┐                                          │
│   │                 │                                          │
│   ▼                 ▼                                          │
│   Header?        Cookie?                                        │
│   Region?        User ID?                                       │
│   │                 │                                          │
│   ▼                 ▼                                          │
│   [Version A]    [Version B]                                   │
│                                                                  │
│   Examples:                                                     │
│   • New users → v2, existing → v1                              │
│   • EU region → v2, US → v1                                    │
│   • Mobile → v2, Desktop → v1                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Istio A/B Testing

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  # Route based on header
  - match:
    - headers:
        x-user-type:
          exact: "beta"
    route:
    - destination:
        host: myapp
        subset: v2
  # Route based on cookie
  - match:
    - headers:
        cookie:
          regex: ".*user_group=experiment.*"
    route:
    - destination:
        host: myapp
        subset: v2
  # Default route
  - route:
    - destination:
        host: myapp
        subset: v1
```

### Nginx A/B Testing

```nginx
# nginx.conf
upstream myapp_v1 {
    server myapp-v1:8080;
}

upstream myapp_v2 {
    server myapp-v2:8080;
}

map $cookie_ab_test $backend {
    default myapp_v1;
    "variant_b" myapp_v2;
}

server {
    location / {
        proxy_pass http://$backend;
    }
}
```

## Feature Flags

Control features without deployments.

```yaml
# Feature flag configuration
features:
  new_checkout:
    enabled: true
    rollout_percentage: 25
    user_whitelist:
      - user123
      - user456
  dark_mode:
    enabled: false
```

### LaunchDarkly Integration

```javascript
// Application code
const LaunchDarkly = require('launchdarkly-node-server-sdk');
const client = LaunchDarkly.init(process.env.LD_SDK_KEY);

app.get('/checkout', async (req, res) => {
  const user = { key: req.userId };
  const useNewCheckout = await client.variation('new-checkout', user, false);

  if (useNewCheckout) {
    return res.render('checkout-v2');
  }
  return res.render('checkout-v1');
});
```

## Rollback Strategies

### Kubernetes Rollback

```bash
# View rollout history
kubectl rollout history deployment/myapp

# Rollback to previous
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2

# Check rollback status
kubectl rollout status deployment/myapp
```

### Automated Rollback

```yaml
# Argo Rollouts with auto-rollback
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      analysis:
        templates:
        - templateName: error-rate
        args:
        - name: service-name
          value: myapp
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
spec:
  metrics:
  - name: error-rate
    interval: 1m
    successCondition: result[0] < 0.05
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

## Complete Deployment Pipeline

```yaml
# GitHub Actions
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myregistry/myapp:${{ github.sha }}

      - name: Deploy Canary
        run: |
          kubectl set image deployment/myapp-canary \
            myapp=myregistry/myapp:${{ github.sha }}
          kubectl rollout status deployment/myapp-canary

      - name: Smoke Tests
        run: ./scripts/smoke-tests.sh

      - name: Promote to 25%
        run: |
          kubectl patch ingress myapp-canary \
            -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"25"}}}'

      - name: Wait and Monitor
        run: sleep 300 && ./scripts/check-metrics.sh

      - name: Full Rollout
        run: |
          kubectl set image deployment/myapp \
            myapp=myregistry/myapp:${{ github.sha }}
          kubectl rollout status deployment/myapp

      - name: Cleanup Canary
        run: kubectl scale deployment/myapp-canary --replicas=0
```

## Quick Reference

### Strategy Comparison

| Strategy | Downtime | Risk | Rollback | Cost |
|----------|----------|------|----------|------|
| Recreate | Yes | High | Slow | Low |
| Rolling | No | Medium | Medium | Low |
| Blue-Green | No | Low | Instant | 2x infra |
| Canary | No | Low | Fast | +10-20% |

### Kubernetes Commands

| Action | Command |
|--------|---------|
| Rolling update | `kubectl set image deployment/app app=img:v2` |
| Check status | `kubectl rollout status deployment/app` |
| Rollback | `kubectl rollout undo deployment/app` |
| History | `kubectl rollout history deployment/app` |
| Pause | `kubectl rollout pause deployment/app` |
| Resume | `kubectl rollout resume deployment/app` |

---

**Previous:** [19-containerized-builds.md](19-containerized-builds.md) | **Next:** [21-kubernetes-deployments.md](21-kubernetes-deployments.md)
