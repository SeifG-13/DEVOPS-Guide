# Filebeat Setup

## What is Filebeat?

Filebeat is a lightweight log shipper that forwards logs to Elasticsearch or Logstash.

```
┌─────────────────────────────────────────────────────────────┐
│                      FILEBEAT                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                    Filebeat                           │ │
│  │                                                       │ │
│  │  ┌─────────────┐    ┌─────────────┐                 │ │
│  │  │  Harvesters │───▶│   Spooler   │                 │ │
│  │  │ (read files)│    │ (aggregate) │                 │ │
│  │  └─────────────┘    └──────┬──────┘                 │ │
│  │                            │                         │ │
│  │                            ▼                         │ │
│  │                    ┌─────────────┐                  │ │
│  │                    │   Output    │                  │ │
│  │                    └─────────────┘                  │ │
│  └───────────────────────────────────────────────────────┘ │
│                            │                                │
│                            ▼                                │
│         ┌─────────────────────────────────┐                │
│         │  Elasticsearch / Logstash       │                │
│         └─────────────────────────────────┘                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Installation

### Docker

```bash
docker run -d \
  --name filebeat \
  --user root \
  -v $(pwd)/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro \
  -v /var/log:/var/log:ro \
  -v /var/lib/docker/containers:/var/lib/docker/containers:ro \
  docker.elastic.co/beats/filebeat:8.11.0
```

### Linux Package

```bash
# Download and install
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.11.0-amd64.deb
sudo dpkg -i filebeat-8.11.0-amd64.deb

# Or via apt
sudo apt-get install filebeat

# Start service
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

### Verify Installation

```bash
# Check version
filebeat version

# Test config
filebeat test config -c /etc/filebeat/filebeat.yml

# Test output connectivity
filebeat test output -c /etc/filebeat/filebeat.yml
```

---

## Basic Configuration

### filebeat.yml

```yaml
# /etc/filebeat/filebeat.yml

# ============================== Filebeat inputs ==============================
filebeat.inputs:

  # Log files input
  - type: log
    enabled: true
    paths:
      - /var/log/*.log
      - /var/log/app/*.log
    exclude_files: ['\.gz$']

# ============================== Outputs ======================================

# Output to Elasticsearch directly
output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

# Or output to Logstash
# output.logstash:
#   hosts: ["logstash:5044"]

# ============================== Kibana =======================================
setup.kibana:
  host: "http://kibana:5601"

# ============================== Logging ======================================
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

---

## Input Types

### Log Input (File-based)

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log

    # Include/exclude patterns
    include_lines: ['^ERROR', '^WARN']
    exclude_lines: ['^DEBUG']
    exclude_files: ['\.gz$', '\.zip$']

    # Multiline for stack traces
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after

    # Add fields
    fields:
      service: myapp
      environment: production
    fields_under_root: true

    # Tags
    tags: ["app-logs"]
```

### Container Input (Docker)

```yaml
filebeat.inputs:
  - type: container
    enabled: true
    paths:
      - '/var/lib/docker/containers/*/*.log'

    # Parse container logs
    stream: all  # stdout, stderr, or all

    # Add container metadata
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"
```

### Filestream Input (Recommended for new setups)

```yaml
filebeat.inputs:
  - type: filestream
    enabled: true
    id: app-logs
    paths:
      - /var/log/app/*.log

    # Prospector settings
    prospector.scanner.check_interval: 10s

    # Fingerprint for file identity
    file_identity.native: ~
```

### Syslog Input

```yaml
filebeat.inputs:
  - type: syslog
    enabled: true
    protocol.udp:
      host: "0.0.0.0:9514"
```

### TCP/UDP Input

```yaml
filebeat.inputs:
  - type: tcp
    enabled: true
    host: "0.0.0.0:9000"

  - type: udp
    enabled: true
    host: "0.0.0.0:9001"
```

---

## Multiline Handling

### Stack Traces (Java)

```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/app/*.log
    multiline.type: pattern
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after

# Example log:
# 2024-01-15 10:23:45 ERROR Exception occurred
#     at com.example.Service.method(Service.java:42)
#     at com.example.Main.main(Main.java:10)
# 2024-01-15 10:23:46 INFO Next log entry
```

### Python Tracebacks

```yaml
multiline.type: pattern
multiline.pattern: '^Traceback|^  File|^    |^\w+Error:'
multiline.negate: false
multiline.match: after
```

### JSON with newlines

```yaml
multiline.type: pattern
multiline.pattern: '^\{'
multiline.negate: true
multiline.match: after
```

---

## Processors

### Add Fields

```yaml
processors:
  - add_fields:
      target: ''
      fields:
        environment: production
        datacenter: us-east-1
```

### Add Host Metadata

```yaml
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

### Drop Events

```yaml
processors:
  - drop_event:
      when:
        contains:
          message: "healthcheck"

  - drop_event:
      when:
        equals:
          level: "DEBUG"
```

### Rename Fields

```yaml
processors:
  - rename:
      fields:
        - from: "hostname"
          to: "host.name"
```

### Parse JSON

```yaml
processors:
  - decode_json_fields:
      fields: ["message"]
      target: ""
      overwrite_keys: true
      add_error_key: true
```

### Dissect

```yaml
processors:
  - dissect:
      tokenizer: "%{timestamp} %{level} %{service} %{message}"
      field: "message"
      target_prefix: ""
```

### Script (JavaScript)

```yaml
processors:
  - script:
      lang: javascript
      source: >
        function process(event) {
          var msg = event.Get("message");
          if (msg.includes("error")) {
            event.Put("is_error", true);
          }
          return event;
        }
```

---

## Output Configuration

### Elasticsearch Output

```yaml
output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]

  # Authentication
  username: "elastic"
  password: "changeme"
  # Or API key:
  # api_key: "id:api_key"

  # Index configuration
  index: "logs-%{[fields.service]}-%{+yyyy.MM.dd}"

  # SSL/TLS
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]

  # Performance tuning
  bulk_max_size: 500
  worker: 2

  # ILM
  setup.ilm.enabled: true
  setup.ilm.rollover_alias: "filebeat"
  setup.ilm.pattern: "{now/d}-000001"
```

### Logstash Output

```yaml
output.logstash:
  hosts: ["logstash:5044"]

  # Load balancing
  loadbalance: true

  # SSL
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]

  # Compression
  compression_level: 3
```

### Kafka Output

```yaml
output.kafka:
  hosts: ["kafka1:9092", "kafka2:9092"]
  topic: "logs-%{[fields.service]}"
  partition.round_robin:
    reachable_only: true
  required_acks: 1
  compression: gzip
```

---

## Modules

### Enable Modules

```bash
# List available modules
filebeat modules list

# Enable module
filebeat modules enable nginx
filebeat modules enable system

# Disable module
filebeat modules disable nginx
```

### Module Configuration

```yaml
# /etc/filebeat/modules.d/nginx.yml
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log*"]
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log*"]
```

### System Module

```yaml
# /etc/filebeat/modules.d/system.yml
- module: system
  syslog:
    enabled: true
    var.paths: ["/var/log/syslog*"]
  auth:
    enabled: true
    var.paths: ["/var/log/auth.log*"]
```

### Common Modules

| Module | Purpose |
|--------|---------|
| system | Syslog, auth logs |
| nginx | Nginx access/error |
| apache | Apache logs |
| mysql | MySQL slow/error logs |
| postgresql | PostgreSQL logs |
| docker | Docker daemon logs |
| kubernetes | K8s audit logs |

---

## Kubernetes Deployment

### DaemonSet

```yaml
# filebeat-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
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
          securityContext:
            runAsUser: 0
          volumeMounts:
            - name: config
              mountPath: /etc/filebeat.yml
              readOnly: true
              subPath: filebeat.yml
            - name: data
              mountPath: /usr/share/filebeat/data
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: filebeat-config
        - name: data
          hostPath:
            path: /var/lib/filebeat-data
            type: DirectoryOrCreate
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
data:
  filebeat.yml: |
    filebeat.inputs:
      - type: container
        paths:
          - /var/lib/docker/containers/*/*.log
        processors:
          - add_kubernetes_metadata:
              host: ${NODE_NAME}
              matchers:
                - logs_path:
                    logs_path: "/var/lib/docker/containers/"

    output.elasticsearch:
      hosts: ["elasticsearch:9200"]
      index: "filebeat-%{+yyyy.MM.dd}"

    setup.kibana:
      host: "kibana:5601"
```

---

## Complete Configuration Example

```yaml
# /etc/filebeat/filebeat.yml

filebeat.inputs:
  # Application logs
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    fields:
      service: myapp
      environment: production
    fields_under_root: true
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after
    processors:
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

  # Nginx logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      service: nginx
      log_type: access
    fields_under_root: true

# Global processors
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - drop_event:
      when:
        contains:
          message: "healthcheck"

# Output
output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "logs-%{[fields.service]}-%{+yyyy.MM.dd}"

# Kibana setup
setup.kibana:
  host: "http://kibana:5601"

# Logging
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Inputs** | Define log sources |
| **Processors** | Transform events |
| **Outputs** | Send to ES/Logstash |
| **Modules** | Pre-built configs |

### Quick Commands

```bash
# Test config
filebeat test config

# Test output
filebeat test output

# Setup dashboards
filebeat setup --dashboards

# Run
filebeat -e
```
