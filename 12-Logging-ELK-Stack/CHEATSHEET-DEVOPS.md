# ELK Stack Cheat Sheet for DevOps Engineers

## Quick Reference - Stack Components

| Component | Purpose | Port |
|-----------|---------|------|
| Elasticsearch | Search & storage | 9200, 9300 |
| Logstash | Log processing | 5044 |
| Kibana | Visualization | 5601 |
| Filebeat | Log shipping | - |
| Fluentd/Fluent Bit | Log collection | 24224 |

---

## Elasticsearch

### Basic Commands
```bash
# Cluster health
curl -X GET "localhost:9200/_cluster/health?pretty"

# List indices
curl -X GET "localhost:9200/_cat/indices?v"

# Index info
curl -X GET "localhost:9200/myindex/_settings?pretty"
curl -X GET "localhost:9200/myindex/_mapping?pretty"

# Search
curl -X GET "localhost:9200/myindex/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "message": "error"
    }
  }
}'

# Delete index
curl -X DELETE "localhost:9200/myindex"

# Create index with mapping
curl -X PUT "localhost:9200/myindex" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" }
    }
  }
}'
```

### Index Lifecycle Management (ILM)
```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

---

## Logstash

### Basic Pipeline
```ruby
# /etc/logstash/conf.d/pipeline.conf

input {
  beats {
    port => 5044
  }
}

filter {
  # Parse JSON logs
  json {
    source => "message"
  }

  # Parse timestamp
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }

  # Add fields based on conditions
  if [level] == "ERROR" {
    mutate {
      add_field => { "alert" => "true" }
    }
  }

  # Grok pattern for unstructured logs
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}" }
  }

  # Remove unwanted fields
  mutate {
    remove_field => [ "host", "agent" ]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

### Common Grok Patterns
```ruby
# Apache access log
%{COMBINEDAPACHELOG}

# Syslog
%{SYSLOGLINE}

# Custom patterns
%{IP:client_ip} - %{USER:user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status} %{NUMBER:bytes}

# .NET log pattern
%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:level}\] %{DATA:logger} - %{GREEDYDATA:message}
```

---

## Filebeat

### filebeat.yml
```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      app: myapp
      env: production

  - type: container
    paths:
      - /var/lib/docker/containers/*/*.log
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "filebeat-%{+yyyy.MM.dd}"

# Or output to Logstash
output.logstash:
  hosts: ["logstash:5044"]

setup.kibana:
  host: "kibana:5601"

logging.level: info
```

### Filebeat Modules
```bash
# List modules
filebeat modules list

# Enable module
filebeat modules enable nginx
filebeat modules enable system

# Load dashboards
filebeat setup --dashboards
```

---

## Kibana

### KQL (Kibana Query Language)
```
# Field search
status: 500
level: "ERROR"

# Wildcards
message: error*
host.name: web-*

# Boolean
status: 500 AND method: POST
level: ERROR OR level: FATAL
NOT status: 200

# Range
response_time > 1000
@timestamp >= "2024-01-01" AND @timestamp < "2024-02-01"

# Exists
kubernetes.pod.name: *
```

### Lucene Query Syntax
```
# Phrase search
message: "connection refused"

# Fuzzy search
message: error~

# Proximity search
message: "database error"~5

# Regex
message: /err.*/
```

---

## Docker Compose Setup

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

---

## Interview Q&A

### Q1: What is the ELK Stack?
**A:**
- **E**lasticsearch: Distributed search and analytics engine
- **L**ogstash: Data processing pipeline
- **K**ibana: Visualization and UI
Often includes Beats (lightweight shippers) = Elastic Stack

### Q2: How does log flow through ELK?
**A:**
```
Application → Filebeat → Logstash → Elasticsearch → Kibana
     ↓           ↓           ↓            ↓           ↓
  Writes      Ships      Parses/      Indexes     Visualizes
   logs       logs      Enriches      & Stores    & Queries
```

### Q3: What is the difference between Filebeat and Logstash?
**A:**
- **Filebeat**: Lightweight shipper, low resource usage, simple processing
- **Logstash**: Heavy processing, complex transformations, multiple inputs/outputs
Use Filebeat for shipping, Logstash for complex processing.

### Q4: How do you handle high log volume?
**A:**
- Use Kafka as buffer between Beats and Logstash
- Index lifecycle management (ILM)
- Hot-warm-cold architecture
- Proper shard sizing
- Use data streams

### Q5: What is an Elasticsearch shard?
**A:** Shards are units of storage:
- **Primary shards**: Original data, set at index creation
- **Replica shards**: Copies for redundancy and read scaling
Rule of thumb: Keep shards between 10-50GB

### Q6: How do you secure the ELK Stack?
**A:**
- Enable X-Pack security
- TLS for transport and HTTP
- Role-based access control (RBAC)
- API key authentication
- Audit logging

### Q7: What is Index Lifecycle Management?
**A:** Automates index management:
- **Hot**: Active writing and querying
- **Warm**: Read-only, less frequent queries
- **Cold**: Infrequent access, cheaper storage
- **Delete**: Remove old data

### Q8: How do you troubleshoot missing logs?
**A:**
1. Check Filebeat/Logstash logs
2. Verify connectivity to Elasticsearch
3. Check index patterns in Kibana
4. Verify file permissions
5. Check log file paths
6. Verify JSON parsing

### Q9: What is a Grok pattern?
**A:** Pattern matching for unstructured logs:
```ruby
%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}
```
Extracts structured fields from text.

### Q10: How do you optimize Elasticsearch performance?
**A:**
- Proper shard sizing (10-50GB)
- Use SSDs for hot nodes
- Disable swapping
- Allocate 50% memory to JVM heap (max 32GB)
- Use bulk indexing
- Optimize mappings (keyword vs text)

---

## Kubernetes Logging

### Fluent Bit DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
```

---

## Best Practices

1. **Use structured logging** - JSON format
2. **Index lifecycle management** - Automate data lifecycle
3. **Proper shard sizing** - 10-50GB per shard
4. **Use index templates** - Consistent mappings
5. **Buffer with Kafka** - Handle spikes
6. **Monitor the stack** - Elasticsearch health, disk space
7. **Set retention policies** - Don't store forever
8. **Use data streams** - For time-series data
