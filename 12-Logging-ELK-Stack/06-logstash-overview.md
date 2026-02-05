# Logstash Overview

## What is Logstash?

Logstash is a data processing pipeline that ingests, transforms, and outputs data.

```
┌─────────────────────────────────────────────────────────────┐
│                    LOGSTASH PIPELINE                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐      │
│  │   INPUTS    │──▶│   FILTERS   │──▶│   OUTPUTS   │      │
│  │             │   │             │   │             │      │
│  │ • beats     │   │ • grok      │   │ • elastic   │      │
│  │ • file      │   │ • mutate    │   │ • file      │      │
│  │ • http      │   │ • date      │   │ • stdout    │      │
│  │ • kafka     │   │ • json      │   │ • kafka     │      │
│  │ • jdbc      │   │ • geoip     │   │ • http      │      │
│  └─────────────┘   └─────────────┘   └─────────────┘      │
│                                                             │
│  Each event flows through:                                 │
│  Input → Codec → Filter → Codec → Output                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Installation

### Docker

```bash
docker run -d \
  --name logstash \
  -p 5044:5044 \
  -p 9600:9600 \
  -v $(pwd)/pipeline:/usr/share/logstash/pipeline \
  docker.elastic.co/logstash/logstash:8.11.0
```

### Linux Package

```bash
# Install
sudo apt-get install logstash

# Config directory: /etc/logstash/
# Pipeline config: /etc/logstash/conf.d/
```

### Verify Installation

```bash
# Check version
logstash --version

# Test config
logstash -t -f /path/to/config.conf

# Run with config
logstash -f /path/to/config.conf
```

---

## Pipeline Configuration

### Basic Structure

```ruby
# /etc/logstash/conf.d/pipeline.conf

input {
  # Data sources
}

filter {
  # Transformations
}

output {
  # Destinations
}
```

### Simple Example

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse JSON if message is JSON
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

---

## Input Plugins

### Beats Input

```ruby
input {
  beats {
    port => 5044
    ssl => false
    # With SSL:
    # ssl => true
    # ssl_certificate => "/path/to/cert.crt"
    # ssl_key => "/path/to/key.key"
  }
}
```

### File Input

```ruby
input {
  file {
    path => "/var/log/app/*.log"
    start_position => "beginning"  # or "end"
    sincedb_path => "/dev/null"    # For testing (re-read files)
    codec => "json"                 # If logs are JSON
  }
}
```

### HTTP Input

```ruby
input {
  http {
    port => 8080
    codec => "json"
  }
}

# Send data:
# curl -X POST http://localhost:8080 -d '{"message":"test"}'
```

### Kafka Input

```ruby
input {
  kafka {
    bootstrap_servers => "kafka:9092"
    topics => ["logs"]
    group_id => "logstash-consumer"
    codec => "json"
    consumer_threads => 3
  }
}
```

### JDBC Input (Database)

```ruby
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydb"
    jdbc_user => "user"
    jdbc_password => "password"
    schedule => "* * * * *"  # Every minute
    statement => "SELECT * FROM logs WHERE updated_at > :sql_last_value"
    use_column_value => true
    tracking_column => "updated_at"
  }
}
```

### Syslog Input

```ruby
input {
  syslog {
    port => 514
    type => "syslog"
  }
}
```

---

## Output Plugins

### Elasticsearch Output

```ruby
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"

    # With authentication
    user => "elastic"
    password => "changeme"

    # With custom template
    template => "/path/to/template.json"
    template_name => "logs"
    template_overwrite => true

    # Bulk settings
    flush_size => 500
    idle_flush_time => 1
  }
}
```

### Conditional Output

```ruby
output {
  if [level] == "ERROR" {
    elasticsearch {
      hosts => ["http://elasticsearch:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
  } else {
    elasticsearch {
      hosts => ["http://elasticsearch:9200"]
      index => "logs-%{+YYYY.MM.dd}"
    }
  }
}
```

### Multiple Outputs

```ruby
output {
  # Always send to Elasticsearch
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }

  # Also send errors to file
  if [level] == "ERROR" {
    file {
      path => "/var/log/logstash/errors-%{+YYYY-MM-dd}.log"
      codec => "json_lines"
    }
  }

  # Debug output
  stdout {
    codec => rubydebug
  }
}
```

### File Output

```ruby
output {
  file {
    path => "/var/log/processed/%{service}-%{+YYYY-MM-dd}.log"
    codec => "json_lines"
  }
}
```

### Kafka Output

```ruby
output {
  kafka {
    bootstrap_servers => "kafka:9092"
    topic_id => "processed-logs"
    codec => "json"
  }
}
```

---

## Codecs

Codecs decode input and encode output.

```ruby
# JSON codec
input {
  file {
    path => "/var/log/app.log"
    codec => "json"
  }
}

# JSON lines (one JSON per line)
input {
  file {
    path => "/var/log/app.log"
    codec => "json_lines"
  }
}

# Plain text (default)
input {
  file {
    path => "/var/log/app.log"
    codec => "plain"
  }
}

# Multiline (for stack traces)
input {
  file {
    path => "/var/log/app.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"
      negate => true
      what => "previous"
    }
  }
}

# Output codec
output {
  stdout {
    codec => rubydebug  # Pretty print for debugging
  }
}
```

---

## Conditionals

```ruby
filter {
  # If condition
  if [level] == "ERROR" {
    mutate {
      add_tag => ["error"]
    }
  }

  # If-else
  if [status] >= 500 {
    mutate { add_field => { "severity" => "error" } }
  } else if [status] >= 400 {
    mutate { add_field => { "severity" => "warning" } }
  } else {
    mutate { add_field => { "severity" => "info" } }
  }

  # Multiple conditions
  if [level] == "ERROR" and [environment] == "production" {
    mutate { add_tag => ["alert"] }
  }

  # Check field existence
  if [user][id] {
    mutate { add_tag => ["has_user"] }
  }

  # Regex match
  if [message] =~ /Exception/ {
    mutate { add_tag => ["exception"] }
  }

  # In list
  if [level] in ["ERROR", "FATAL", "CRITICAL"] {
    mutate { add_tag => ["severe"] }
  }

  # Not condition
  if [environment] != "development" {
    # Production processing
  }
}
```

---

## Field References

```ruby
filter {
  # Access top-level field
  if [message] == "test" { }

  # Access nested field
  if [user][email] == "admin@example.com" { }

  # Use field in string (sprintf)
  mutate {
    add_field => { "full_name" => "%{[user][first]} %{[user][last]}" }
  }

  # Dynamic field names
  mutate {
    rename => { "[data][%{field_name}]" => "[processed][value]" }
  }
}
```

---

## Configuration Files

### logstash.yml

```yaml
# /etc/logstash/logstash.yml
path.data: /var/lib/logstash
path.logs: /var/log/logstash

# Pipeline settings
pipeline.workers: 4
pipeline.batch.size: 125
pipeline.batch.delay: 50

# Monitoring
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]
```

### pipelines.yml (Multiple Pipelines)

```yaml
# /etc/logstash/pipelines.yml
- pipeline.id: main
  path.config: "/etc/logstash/conf.d/main.conf"
  pipeline.workers: 4

- pipeline.id: errors
  path.config: "/etc/logstash/conf.d/errors.conf"
  pipeline.workers: 2
```

---

## Testing Pipeline

```bash
# Test configuration syntax
logstash -t -f /path/to/pipeline.conf

# Run with debug output
logstash -f /path/to/pipeline.conf --log.level=debug

# Test with stdin/stdout
# pipeline.conf:
input { stdin { codec => json } }
filter { }
output { stdout { codec => rubydebug } }

# Then type JSON and see output
echo '{"message":"test","level":"INFO"}' | logstash -f pipeline.conf
```

---

## Complete Example

```ruby
# /etc/logstash/conf.d/complete.conf

input {
  beats {
    port => 5044
  }
}

filter {
  # Parse JSON message
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
      target => "parsed"
    }
    mutate {
      remove_field => ["message"]
    }
    # Move parsed fields to root
    ruby {
      code => "
        event.get('parsed').each { |k, v| event.set(k, v) }
        event.remove('parsed')
      "
    }
  }

  # Parse timestamp
  date {
    match => ["timestamp", "ISO8601", "yyyy-MM-dd HH:mm:ss"]
    target => "@timestamp"
  }

  # Add environment from hostname
  if [host][name] =~ /^prod-/ {
    mutate { add_field => { "environment" => "production" } }
  } else if [host][name] =~ /^staging-/ {
    mutate { add_field => { "environment" => "staging" } }
  } else {
    mutate { add_field => { "environment" => "development" } }
  }

  # Enrich with GeoIP
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geo"
    }
  }

  # Remove unnecessary fields
  mutate {
    remove_field => ["agent", "ecs", "input", "log"]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "%{[service]}-%{+YYYY.MM.dd}"
  }
}
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Input** | Ingest data from sources |
| **Filter** | Transform and enrich |
| **Output** | Send to destinations |
| **Codec** | Decode/encode data format |

### Common Plugins

| Input | Filter | Output |
|-------|--------|--------|
| beats | grok | elasticsearch |
| file | mutate | file |
| http | date | stdout |
| kafka | json | kafka |
| jdbc | geoip | http |
