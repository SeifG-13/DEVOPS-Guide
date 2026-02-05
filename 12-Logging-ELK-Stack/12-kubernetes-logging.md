# Kubernetes Logging with ELK

## Kubernetes Logging Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              KUBERNETES LOGGING STACK                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Kubernetes Cluster                    │   │
│  │                                                     │   │
│  │  Node 1           Node 2           Node 3          │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │   │
│  │  │ Pod  Pod    │ │ Pod  Pod    │ │ Pod  Pod    │  │   │
│  │  │  │    │     │ │  │    │     │ │  │    │     │  │   │
│  │  │  ▼    ▼     │ │  ▼    ▼     │ │  ▼    ▼     │  │   │
│  │  │ /var/log/   │ │ /var/log/   │ │ /var/log/   │  │   │
│  │  │ containers/ │ │ containers/ │ │ containers/ │  │   │
│  │  │      │      │ │      │      │ │      │      │  │   │
│  │  │  Filebeat   │ │  Filebeat   │ │  Filebeat   │  │   │
│  │  │ (DaemonSet) │ │ (DaemonSet) │ │ (DaemonSet) │  │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘  │   │
│  └─────────┼───────────────┼───────────────┼─────────┘   │
│            └───────────────┼───────────────┘              │
│                            ▼                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Elasticsearch Cluster                   │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Kibana                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Log Sources in Kubernetes

```
┌────────────────────────────────────────────────────────────┐
│                    LOG SOURCES                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Container Logs (stdout/stderr)                           │
│  └── /var/log/containers/*.log                            │
│  └── /var/lib/docker/containers/*/*.log                   │
│                                                            │
│  Kubernetes System Logs                                   │
│  └── /var/log/kube-apiserver.log                         │
│  └── /var/log/kube-scheduler.log                         │
│  └── /var/log/kube-controller-manager.log                │
│                                                            │
│  Node Logs                                                │
│  └── /var/log/syslog                                     │
│  └── /var/log/messages                                   │
│                                                            │
│  Audit Logs                                               │
│  └── /var/log/kubernetes/audit.log                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Deploy ELK on Kubernetes

### Using ECK (Elastic Cloud on Kubernetes)

```bash
# Install ECK operator
kubectl create -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml

# Create logging namespace
kubectl create namespace logging
```

### Elasticsearch Deployment

```yaml
# elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: logging
spec:
  version: 8.11.0
  nodeSets:
    - name: default
      count: 3
      config:
        node.store.allow_mmap: false
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 2Gi
                  cpu: 1
                limits:
                  memory: 4Gi
                  cpu: 2
              env:
                - name: ES_JAVA_OPTS
                  value: "-Xms2g -Xmx2g"
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 100Gi
            storageClassName: managed-premium  # AKS
```

### Kibana Deployment

```yaml
# kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: logging
spec:
  version: 8.11.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
  podTemplate:
    spec:
      containers:
        - name: kibana
          resources:
            requests:
              memory: 1Gi
              cpu: 500m
            limits:
              memory: 2Gi
              cpu: 1
```

---

## Filebeat DaemonSet

### ConfigMap

```yaml
# filebeat-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
  labels:
    app: filebeat
data:
  filebeat.yml: |
    filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
        processors:
          - add_kubernetes_metadata:
              host: ${NODE_NAME}
              matchers:
                - logs_path:
                    logs_path: "/var/log/containers/"

    filebeat.autodiscover:
      providers:
        - type: kubernetes
          node: ${NODE_NAME}
          hints.enabled: true
          hints.default_config:
            type: container
            paths:
              - /var/log/containers/*${data.kubernetes.container.id}.log

    processors:
      - add_cloud_metadata: ~
      - add_host_metadata: ~
      - drop_event:
          when:
            or:
              - contains:
                  kubernetes.namespace: "kube-system"
              - contains:
                  message: "healthcheck"

    output.elasticsearch:
      hosts: ["https://elasticsearch-es-http.logging.svc:9200"]
      username: "elastic"
      password: "${ELASTIC_PASSWORD}"
      ssl.certificate_authorities:
        - /etc/certificate/ca.crt
      index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

    setup.kibana:
      host: "https://kibana-kb-http.logging.svc:5601"
      ssl.certificate_authorities:
        - /etc/certificate/ca.crt
```

### DaemonSet

```yaml
# filebeat-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: filebeat
          image: docker.elastic.co/beats/filebeat:8.11.0
          args:
            - "-c"
            - "/etc/filebeat.yml"
            - "-e"
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: ELASTIC_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: elasticsearch-es-elastic-user
                  key: elastic
          securityContext:
            runAsUser: 0
          resources:
            limits:
              memory: 200Mi
              cpu: 200m
            requests:
              memory: 100Mi
              cpu: 100m
          volumeMounts:
            - name: config
              mountPath: /etc/filebeat.yml
              readOnly: true
              subPath: filebeat.yml
            - name: data
              mountPath: /usr/share/filebeat/data
            - name: varlogcontainers
              mountPath: /var/log/containers
              readOnly: true
            - name: varlogpods
              mountPath: /var/log/pods
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: elasticsearch-certs
              mountPath: /etc/certificate
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: filebeat-config
        - name: data
          hostPath:
            path: /var/lib/filebeat-data
            type: DirectoryOrCreate
        - name: varlogcontainers
          hostPath:
            path: /var/log/containers
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: elasticsearch-certs
          secret:
            secretName: elasticsearch-es-http-certs-public
```

### ServiceAccount and RBAC

```yaml
# filebeat-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: logging

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
rules:
  - apiGroups: [""]
    resources:
      - namespaces
      - pods
      - nodes
    verbs: ["get", "watch", "list"]
  - apiGroups: ["apps"]
    resources:
      - replicasets
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: logging
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
```

---

## Application Pod Configuration

### Enable Logging with Annotations

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp
spec:
  template:
    metadata:
      annotations:
        # Filebeat autodiscover hints
        co.elastic.logs/enabled: "true"
        co.elastic.logs/json.keys_under_root: "true"
        co.elastic.logs/json.add_error_key: "true"
        co.elastic.logs/json.message_key: "message"
      labels:
        app: myapp-api
    spec:
      containers:
        - name: api
          image: myapp-api:latest
          # ... container config
```

### JSON Logging Configuration

```yaml
# For .NET apps - ensure JSON logging format
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  appsettings.json: |
    {
      "Serilog": {
        "WriteTo": [
          {
            "Name": "Console",
            "Args": {
              "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
            }
          }
        ]
      }
    }
```

---

## AKS-Specific Configuration

### Azure Managed Identity for Elasticsearch

```yaml
# Use Azure Disk for storage
volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-premium
      resources:
        requests:
          storage: 100Gi
```

### Ingress for Kibana

```yaml
# kibana-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: logging
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: kibana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kibana-kb-http
                port:
                  number: 5601
  tls:
    - hosts:
        - kibana.example.com
      secretName: kibana-tls
```

---

## Helm Installation (Alternative)

```bash
# Add Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=3 \
  --set resources.requests.memory=2Gi \
  --set volumeClaimTemplate.resources.requests.storage=100Gi

# Install Kibana
helm install kibana elastic/kibana \
  --namespace logging \
  --set elasticsearchHosts="http://elasticsearch-master:9200"

# Install Filebeat
helm install filebeat elastic/filebeat \
  --namespace logging \
  --set daemonset.enabled=true \
  --set daemonset.filebeatConfig."filebeat\.yml"="
filebeat.inputs:
- type: container
  paths:
    - /var/log/containers/*.log
  processors:
  - add_kubernetes_metadata:
      host: \${NODE_NAME}
output.elasticsearch:
  hosts: [\"http://elasticsearch-master:9200\"]
"
```

---

## Index Lifecycle Management

```yaml
# ilm-policy.yaml (apply via Kibana Dev Tools)
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "7d",
        "actions": {
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## Useful Queries

```bash
# Logs by namespace
GET filebeat-*/_search
{
  "query": {
    "term": { "kubernetes.namespace": "myapp" }
  }
}

# Errors in last hour
GET filebeat-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "level": "error" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}

# Logs by pod
GET filebeat-*/_search
{
  "query": {
    "term": { "kubernetes.pod.name": "myapp-api-xxx" }
  }
}
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **ECK Operator** | Manage ES/Kibana on K8s |
| **Filebeat DaemonSet** | Collect logs from all nodes |
| **Autodiscover** | Dynamic pod log collection |
| **ILM** | Manage index lifecycle |

### Deploy Commands

```bash
# Deploy ECK
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml

# Deploy stack
kubectl apply -f elasticsearch.yaml
kubectl apply -f kibana.yaml
kubectl apply -f filebeat-rbac.yaml
kubectl apply -f filebeat-configmap.yaml
kubectl apply -f filebeat-daemonset.yaml

# Get Kibana password
kubectl get secret elasticsearch-es-elastic-user -n logging -o jsonpath='{.data.elastic}' | base64 -d

# Port forward
kubectl port-forward svc/kibana-kb-http -n logging 5601:5601
```
