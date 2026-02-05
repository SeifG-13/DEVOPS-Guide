# Elasticsearch Installation

## Installation Options

| Method | Best For |
|--------|----------|
| Docker | Development, quick testing |
| Docker Compose | Full ELK stack locally |
| Binary/Package | Production on VMs |
| Helm/Kubernetes | Production on K8s/AKS |
| Elastic Cloud | Managed service |

---

## Docker Installation

### Single Node (Development)

```bash
# Create network
docker network create elastic

# Run Elasticsearch
docker run -d \
  --name elasticsearch \
  --net elastic \
  -p 9200:9200 \
  -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms1g -Xmx1g" \
  -v elasticsearch-data:/usr/share/elasticsearch/data \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.0

# Verify
curl http://localhost:9200

# With security enabled (default in 8.x)
docker run -d \
  --name elasticsearch \
  --net elastic \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "ELASTIC_PASSWORD=changeme" \
  -v elasticsearch-data:/usr/share/elasticsearch/data \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.0

# Get enrollment token for Kibana
docker exec -it elasticsearch bin/elasticsearch-create-enrollment-token -s kibana
```

---

## Docker Compose (Full Stack)

### Complete ELK Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - bootstrap.memory_lock=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elastic
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200 | grep -q 'cluster_name'"]
      interval: 10s
      timeout: 10s
      retries: 120

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "5044:5044"
      - "9600:9600"
    environment:
      - "LS_JAVA_OPTS=-Xms512m -Xmx512m"
    networks:
      - elastic
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    networks:
      - elastic
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:5601/api/status | grep -q 'available'"]
      interval: 10s
      timeout: 10s
      retries: 120

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - elastic
    depends_on:
      elasticsearch:
        condition: service_healthy

networks:
  elastic:
    driver: bridge

volumes:
  elasticsearch-data:
```

### Logstash Configuration

```yaml
# logstash/config/logstash.yml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]
```

```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

### Filebeat Configuration

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'

processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  indices:
    - index: "filebeat-%{+yyyy.MM.dd}"

setup.kibana:
  host: "kibana:5601"
```

### Start the Stack

```bash
# Create directories
mkdir -p logstash/pipeline logstash/config filebeat

# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f elasticsearch

# Access Kibana: http://localhost:5601
# Access Elasticsearch: http://localhost:9200
```

---

## Binary Installation (Linux)

### Install Elasticsearch

```bash
# Import GPG key
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Install
sudo apt-get update
sudo apt-get install elasticsearch

# Or download directly
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.11.0-amd64.deb
sudo dpkg -i elasticsearch-8.11.0-amd64.deb
```

### Configure Elasticsearch

```yaml
# /etc/elasticsearch/elasticsearch.yml
cluster.name: my-cluster
node.name: node-1

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

network.host: 0.0.0.0
http.port: 9200

discovery.type: single-node  # For single node setup

# For cluster setup:
# discovery.seed_hosts: ["node1", "node2", "node3"]
# cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# Security (disable for testing only)
xpack.security.enabled: false
```

### JVM Settings

```bash
# /etc/elasticsearch/jvm.options.d/heap.options
# Set heap size (50% of available RAM, max 31GB)
-Xms4g
-Xmx4g
```

### Start Elasticsearch

```bash
# Start service
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# Check status
sudo systemctl status elasticsearch

# View logs
sudo journalctl -u elasticsearch -f

# Test
curl http://localhost:9200
```

---

## Kubernetes Installation (Helm)

### Using ECK (Elastic Cloud on Kubernetes)

```bash
# Install ECK operator
kubectl create -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml

# Verify operator
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

### Deploy Elasticsearch

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
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 50Gi
```

### Deploy Kibana

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
              memory: 512Mi
              cpu: 500m
            limits:
              memory: 1Gi
              cpu: 1
```

### Apply Resources

```bash
# Create namespace
kubectl create namespace logging

# Deploy Elasticsearch
kubectl apply -f elasticsearch.yaml

# Deploy Kibana
kubectl apply -f kibana.yaml

# Check status
kubectl get elasticsearch -n logging
kubectl get kibana -n logging

# Get password
kubectl get secret elasticsearch-es-elastic-user -n logging -o jsonpath='{.data.elastic}' | base64 -d

# Port forward Kibana
kubectl port-forward -n logging svc/kibana-kb-http 5601:5601

# Port forward Elasticsearch
kubectl port-forward -n logging svc/elasticsearch-es-http 9200:9200
```

---

## Using Helm Charts

### Elastic Helm Charts

```bash
# Add Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=3 \
  --set minimumMasterNodes=2 \
  --set resources.requests.memory=2Gi \
  --set volumeClaimTemplate.resources.requests.storage=50Gi

# Install Kibana
helm install kibana elastic/kibana \
  --namespace logging \
  --set elasticsearchHosts="http://elasticsearch-master:9200"

# Install Filebeat
helm install filebeat elastic/filebeat \
  --namespace logging \
  --set daemonset.enabled=true
```

### Custom Values

```yaml
# elasticsearch-values.yaml
replicas: 3
minimumMasterNodes: 2

resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"

volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 50Gi

esConfig:
  elasticsearch.yml: |
    xpack.security.enabled: false

extraEnvs:
  - name: ES_JAVA_OPTS
    value: "-Xms2g -Xmx2g"
```

```bash
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  -f elasticsearch-values.yaml
```

---

## Cluster Setup (Multi-Node)

### Node Configuration

```yaml
# Node 1 (Master + Data)
# /etc/elasticsearch/elasticsearch.yml
cluster.name: production-cluster
node.name: es-node-1
node.roles: [master, data]

network.host: 10.0.0.1
http.port: 9200

discovery.seed_hosts: ["10.0.0.1", "10.0.0.2", "10.0.0.3"]
cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
```

```yaml
# Node 2 (Master + Data)
cluster.name: production-cluster
node.name: es-node-2
node.roles: [master, data]

network.host: 10.0.0.2
discovery.seed_hosts: ["10.0.0.1", "10.0.0.2", "10.0.0.3"]
cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
```

### Node Roles

```yaml
# Master-only node
node.roles: [master]

# Data-only node
node.roles: [data]

# Ingest node (for pipelines)
node.roles: [ingest]

# Coordinating-only node
node.roles: []

# Combined (default)
node.roles: [master, data, ingest]
```

---

## Verify Installation

```bash
# Check cluster health
curl -X GET "localhost:9200/_cluster/health?pretty"

# Check nodes
curl -X GET "localhost:9200/_cat/nodes?v"

# Check indices
curl -X GET "localhost:9200/_cat/indices?v"

# Index a test document
curl -X POST "localhost:9200/test-index/_doc" \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello Elasticsearch"}'

# Search
curl -X GET "localhost:9200/test-index/_search?pretty"
```

---

## Summary

| Method | Command |
|--------|---------|
| **Docker** | `docker run docker.elastic.co/elasticsearch/elasticsearch:8.11.0` |
| **Compose** | `docker-compose up -d` |
| **APT** | `apt install elasticsearch` |
| **Helm** | `helm install elasticsearch elastic/elasticsearch` |
| **ECK** | `kubectl apply -f elasticsearch.yaml` |

### Default URLs

```
Elasticsearch: http://localhost:9200
Kibana:        http://localhost:5601
Logstash:      http://localhost:9600 (monitoring)
```
