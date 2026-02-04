# Kubernetes Monitoring with Prometheus

## Kubernetes Monitoring Overview

```
┌─────────────────────────────────────────────────────────────┐
│              KUBERNETES MONITORING STACK                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Kubernetes Cluster                   │   │
│  │                                                     │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  Node   │  │  Node   │  │  Node   │            │   │
│  │  │ Exporter│  │ Exporter│  │ Exporter│            │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘            │   │
│  │       │            │            │                  │   │
│  │  ┌─────────────────────────────────────────────┐  │   │
│  │  │                 cAdvisor                     │  │   │
│  │  │            (Container metrics)              │  │   │
│  │  └──────────────────┬──────────────────────────┘  │   │
│  │                     │                              │   │
│  │  ┌─────────────────────────────────────────────┐  │   │
│  │  │            kube-state-metrics               │  │   │
│  │  │         (Kubernetes object metrics)         │  │   │
│  │  └──────────────────┬──────────────────────────┘  │   │
│  └─────────────────────┼──────────────────────────────┘   │
│                        ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Prometheus                         │   │
│  └─────────────────────┬───────────────────────────────┘   │
│                        ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Grafana                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## kube-prometheus-stack

The recommended way to deploy monitoring on Kubernetes.

### Install with Helm

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123

# Check status
kubectl get pods -n monitoring
```

### Custom Values

```yaml
# values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests:
        memory: 1Gi
        cpu: 500m
      limits:
        memory: 2Gi
        cpu: 1000m

    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

    # Service discovery
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

grafana:
  adminPassword: secure-password
  persistence:
    enabled: true
    size: 10Gi

  # Pre-configured dashboards
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'default'
          folder: ''
          type: file
          options:
            path: /var/lib/grafana/dashboards

  # Default data source
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://prometheus-kube-prometheus-prometheus:9090
          isDefault: true

# Node Exporter
nodeExporter:
  enabled: true

# kube-state-metrics
kubeStateMetrics:
  enabled: true

# Prometheus Operator
prometheusOperator:
  resources:
    requests:
      memory: 100Mi
      cpu: 100m
```

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f values.yaml
```

---

## ServiceMonitor and PodMonitor

### ServiceMonitor

Monitor services that expose /metrics endpoint.

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-api
  namespace: monitoring
  labels:
    release: prometheus  # Must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: myapp-api
  namespaceSelector:
    matchNames:
      - myapp
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

### Service with Metrics Port

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api
  namespace: myapp
  labels:
    app: myapp-api
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
  selector:
    app: myapp-api
```

### PodMonitor

Monitor pods directly (no service needed).

```yaml
# podmonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-workers
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp-worker
  namespaceSelector:
    matchNames:
      - myapp
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## Monitoring Your Applications

### Application Deployment with Metrics

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp-api
  template:
    metadata:
      labels:
        app: myapp-api
      annotations:
        # For pod annotation-based discovery
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: api
          image: myapp-api:latest
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
```

---

## Key Metrics to Monitor

### Cluster-Level Metrics

```promql
# Node count
count(kube_node_info)

# Node ready status
kube_node_status_condition{condition="Ready", status="true"}

# Total pods
sum(kube_pod_status_phase)

# Pods by phase
sum by (phase) (kube_pod_status_phase)
```

### Node Metrics

```promql
# CPU usage by node
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage by node
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Network I/O
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

### Pod/Container Metrics

```promql
# Container CPU usage
sum by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{image!=""}[5m])) * 100

# Container memory usage
sum by (namespace, pod, container) (container_memory_usage_bytes{image!=""})

# Container restarts
sum by (namespace, pod, container) (kube_pod_container_status_restarts_total)

# Pod resource requests vs usage
# CPU
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total[5m]))
/
sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})

# Memory
sum by (namespace, pod) (container_memory_usage_bytes)
/
sum by (namespace, pod) (kube_pod_container_resource_requests{resource="memory"})
```

### Deployment Metrics

```promql
# Deployment replicas status
kube_deployment_status_replicas_available / kube_deployment_spec_replicas

# Unavailable replicas
kube_deployment_status_replicas_unavailable > 0

# Deployment generation mismatch
kube_deployment_status_observed_generation != kube_deployment_metadata_generation
```

### Namespace Metrics

```promql
# Pods per namespace
sum by (namespace) (kube_pod_info)

# Resource usage per namespace
sum by (namespace) (container_memory_usage_bytes)
sum by (namespace) (rate(container_cpu_usage_seconds_total[5m]))

# Resource quota usage
kube_resourcequota{type="used"} / kube_resourcequota{type="hard"}
```

---

## Alert Rules for Kubernetes

```yaml
# kubernetes-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: kubernetes-pods
      rules:
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total[1h]) > 5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"

        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="true"} == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"

        - alert: ContainerOOMKilled
          expr: |
            kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: "Container {{ $labels.container }} in {{ $labels.namespace }}/{{ $labels.pod }} OOM killed"

    - name: kubernetes-nodes
      rules:
        - alert: NodeNotReady
          expr: |
            kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} is not ready"

        - alert: NodeMemoryPressure
          expr: |
            kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Node {{ $labels.node }} has memory pressure"

        - alert: NodeDiskPressure
          expr: |
            kube_node_status_condition{condition="DiskPressure",status="true"} == 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Node {{ $labels.node }} has disk pressure"

    - name: kubernetes-resources
      rules:
        - alert: HighCPUUsage
          expr: |
            (sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{image!=""}[5m]))
            /
            sum by (namespace, pod) (kube_pod_container_resource_limits{resource="cpu"})) > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} CPU usage > 90%"

        - alert: HighMemoryUsage
          expr: |
            (sum by (namespace, pod) (container_memory_usage_bytes{image!=""})
            /
            sum by (namespace, pod) (kube_pod_container_resource_limits{resource="memory"})) > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} memory usage > 90%"

        - alert: PVCAlmostFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is > 85% full"

    - name: kubernetes-deployments
      rules:
        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas != kube_deployment_status_replicas_available
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} replicas mismatch"

        - alert: StatefulSetReplicasMismatch
          expr: |
            kube_statefulset_status_replicas_ready != kube_statefulset_status_replicas
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} replicas mismatch"
```

---

## Access Monitoring Services

### Port Forwarding

```bash
# Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Alertmanager
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
```

### Ingress

```yaml
# monitoring-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: monitoring-basic-auth
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80

    - host: prometheus.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-prometheus
                port:
                  number: 9090
```

---

## Pre-Built Grafana Dashboards

### Included with kube-prometheus-stack

- Kubernetes / API server
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace (Pods)
- Kubernetes / Compute Resources / Node (Pods)
- Kubernetes / Compute Resources / Pod
- Kubernetes / Networking / Cluster
- Kubernetes / Networking / Namespace
- Node Exporter Full
- Prometheus Overview

### Import Additional Dashboards

| ID | Name |
|----|------|
| 315 | Kubernetes cluster monitoring |
| 6417 | Kubernetes Cluster |
| 8588 | Kubernetes Deployment Statefulset |
| 13770 | Kubernetes / Views / Global |

---

## AKS-Specific Monitoring

### Enable Container Insights

```bash
az aks enable-addons \
  --name aks-myapp-prod \
  --resource-group rg-myapp-prod \
  --addons monitoring
```

### Combined Monitoring

```bash
# Install Prometheus on AKS
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set grafana.adminPassword=admin123

# Azure Monitor + Prometheus work together
# Azure Monitor: Platform-level insights
# Prometheus: Application-level metrics
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **kube-prometheus-stack** | Complete monitoring stack |
| **ServiceMonitor** | Monitor services |
| **PodMonitor** | Monitor pods directly |
| **PrometheusRule** | Define alert rules |
| **kube-state-metrics** | K8s object metrics |
| **Node Exporter** | Node-level metrics |

### Quick Commands

```bash
# Install
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Check targets
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090/targets

# View alerts
kubectl get prometheusrules -n monitoring
```
