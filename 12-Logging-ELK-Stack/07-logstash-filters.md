# Logstash Filters

## Filter Plugins Overview

Filters transform and enrich events as they pass through the pipeline.

```
┌─────────────────────────────────────────────────────────────┐
│                    COMMON FILTERS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  grok      - Parse unstructured text into fields           │
│  mutate    - Modify fields (rename, remove, convert)       │
│  date      - Parse dates into @timestamp                   │
│  json      - Parse JSON strings                            │
│  geoip     - Add geographic data from IP addresses         │
│  kv        - Parse key=value pairs                         │
│  dissect   - Fast pattern-based parsing                    │
│  ruby      - Custom Ruby code                              │
│  drop      - Drop events                                   │
│  clone     - Duplicate events                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Grok Filter

Grok uses regular expressions with named patterns to parse unstructured text.

### Basic Syntax

```ruby
filter {
  grok {
    match => { "message" => "%{PATTERN:field_name}" }
  }
}
```

### Common Grok Patterns

```
# Base patterns
%{WORD}           # Single word
%{NUMBER}         # Integer or float
%{INT}            # Integer
%{POSINT}         # Positive integer
%{IP}             # IP address
%{HOSTNAME}       # Hostname
%{URIPATH}        # URI path
%{QUOTEDSTRING}   # "quoted string"
%{GREEDYDATA}     # Any characters (greedy)
%{DATA}           # Any characters (non-greedy)

# Combined patterns
%{COMBINEDAPACHELOG}  # Apache access log
%{SYSLOGBASE}         # Syslog prefix
%{TIMESTAMP_ISO8601}  # ISO 8601 timestamp
%{HTTPDATE}           # HTTP date format
%{LOGLEVEL}           # Log level (INFO, ERROR, etc.)
```

### Grok Examples

#### Apache Access Log

```ruby
# Log format:
# 192.168.1.1 - - [15/Jan/2024:10:23:45 +0000] "GET /api/users HTTP/1.1" 200 1234

filter {
  grok {
    match => {
      "message" => "%{COMBINEDAPACHELOG}"
    }
  }
}

# Extracted fields:
# clientip: 192.168.1.1
# verb: GET
# request: /api/users
# response: 200
# bytes: 1234
```

#### Custom Application Log

```ruby
# Log format:
# 2024-01-15 10:23:45.123 [INFO] [auth-service] User 12345 logged in from 192.168.1.1

filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:level}\] \[%{DATA:service}\] %{GREEDYDATA:log_message}"
    }
  }
}

# More specific parsing:
filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:level}\] \[%{DATA:service}\] User %{INT:user_id} logged in from %{IP:client_ip}"
    }
  }
}
```

#### Nginx Error Log

```ruby
# Log format:
# 2024/01/15 10:23:45 [error] 12345#0: *67890 open() "/var/www/missing.html" failed

filter {
  grok {
    match => {
      "message" => "%{DATA:timestamp} \[%{LOGLEVEL:level}\] %{POSINT:pid}#%{NUMBER:tid}: \*%{NUMBER:connection_id} %{GREEDYDATA:error_message}"
    }
  }
}
```

### Multiple Patterns

```ruby
filter {
  grok {
    match => {
      "message" => [
        "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:level}\] %{GREEDYDATA:msg}",
        "%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:host} %{GREEDYDATA:msg}",
        "%{GREEDYDATA:msg}"  # Fallback
      ]
    }
  }
}
```

### Custom Patterns

```ruby
# Define custom patterns
filter {
  grok {
    patterns_dir => ["/etc/logstash/patterns"]
    match => { "message" => "%{MYPATTERN:field}" }
  }
}

# /etc/logstash/patterns/custom
# MYPATTERN [A-Z]{3}-[0-9]{4}
# ORDER_ID ORD-[0-9]+
# USER_EMAIL [a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}
```

### Grok Debugger

Test patterns at: https://grokdebugger.com/ or Kibana Dev Tools

---

## Mutate Filter

Modify fields in events.

### Rename Fields

```ruby
filter {
  mutate {
    rename => {
      "hostname" => "[host][name]"
      "clientip" => "[client][ip]"
    }
  }
}
```

### Remove Fields

```ruby
filter {
  mutate {
    remove_field => ["message", "agent", "ecs", "log"]
  }
}
```

### Add Fields

```ruby
filter {
  mutate {
    add_field => {
      "environment" => "production"
      "[event][processed]" => "true"
    }
  }
}
```

### Convert Types

```ruby
filter {
  mutate {
    convert => {
      "response_code" => "integer"
      "response_time" => "float"
      "is_error" => "boolean"
    }
  }
}
```

### String Operations

```ruby
filter {
  mutate {
    # Lowercase
    lowercase => ["level", "service"]

    # Uppercase
    uppercase => ["country_code"]

    # Strip whitespace
    strip => ["message"]

    # Replace
    gsub => [
      # Replace slashes with dashes
      "path", "/", "-",
      # Remove quotes
      "message", "\"", ""
    ]

    # Split into array
    split => { "tags" => "," }

    # Join array
    join => { "tags" => "," }

    # Update field
    update => { "status" => "processed" }

    # Replace field (create if not exists)
    replace => { "message" => "%{level}: %{original_message}" }
  }
}
```

### Copy Fields

```ruby
filter {
  mutate {
    copy => {
      "message" => "original_message"
      "[user][email]" => "[contact][email]"
    }
  }
}
```

### Merge Arrays

```ruby
filter {
  mutate {
    merge => {
      "all_tags" => "new_tags"
    }
  }
}
```

---

## Date Filter

Parse dates and set @timestamp.

```ruby
filter {
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }
}

# Multiple formats
filter {
  date {
    match => [
      "timestamp",
      "ISO8601",
      "yyyy-MM-dd HH:mm:ss",
      "yyyy-MM-dd HH:mm:ss.SSS",
      "dd/MMM/yyyy:HH:mm:ss Z",
      "UNIX",
      "UNIX_MS"
    ]
    target => "@timestamp"
    timezone => "UTC"
  }
}
```

### Date Formats

```
ISO8601               2024-01-15T10:23:45.123Z
yyyy-MM-dd HH:mm:ss   2024-01-15 10:23:45
dd/MMM/yyyy:HH:mm:ss  15/Jan/2024:10:23:45
UNIX                  1705315425 (seconds)
UNIX_MS               1705315425123 (milliseconds)
```

---

## JSON Filter

Parse JSON strings into fields.

```ruby
# Parse message field
filter {
  json {
    source => "message"
    target => "parsed"  # Optional: put in nested field
  }
}

# Parse and merge to root
filter {
  json {
    source => "message"
    # No target = merge to root
  }
}

# Handle parse errors
filter {
  json {
    source => "message"
    skip_on_invalid_json => true
    tag_on_failure => ["_jsonparsefailure"]
  }
}
```

---

## GeoIP Filter

Add geographic information from IP addresses.

```ruby
filter {
  geoip {
    source => "client_ip"
    target => "geo"
    # Optional: specify database
    # database => "/path/to/GeoLite2-City.mmdb"
  }
}

# Result:
# geo.country_name: "United States"
# geo.city_name: "San Francisco"
# geo.location.lat: 37.7749
# geo.location.lon: -122.4194
```

### GeoIP with specific fields

```ruby
filter {
  geoip {
    source => "client_ip"
    target => "geo"
    fields => ["country_name", "city_name", "location"]
  }
}
```

---

## KV Filter

Parse key=value pairs.

```ruby
# Input: "user=john status=active count=5"
filter {
  kv {
    source => "message"
    field_split => " "
    value_split => "="
  }
}

# Result:
# user: john
# status: active
# count: 5

# With prefix
filter {
  kv {
    source => "query_string"
    field_split => "&"
    value_split => "="
    prefix => "param_"
  }
}
```

---

## Dissect Filter

Fast parsing with fixed delimiters (faster than grok).

```ruby
# Input: "2024-01-15 10:23:45 INFO auth-service User logged in"
filter {
  dissect {
    mapping => {
      "message" => "%{timestamp} %{level} %{service} %{log_message}"
    }
  }
}

# With modifiers
filter {
  dissect {
    mapping => {
      # %{} = normal field
      # %{+} = append to existing field
      # %{?} = skip field (don't store)
      # %{&} = use as key name
      "message" => "%{timestamp} %{+timestamp} %{level} %{?skip} %{log_message}"
    }
  }
}
```

---

## Ruby Filter

Custom Ruby code for complex transformations.

```ruby
filter {
  ruby {
    code => "
      # Access event
      message = event.get('message')

      # Set fields
      event.set('message_length', message.length)
      event.set('processed_at', Time.now.utc.iso8601)

      # Conditional logic
      if event.get('status').to_i >= 500
        event.set('is_error', true)
      else
        event.set('is_error', false)
      end

      # Remove field
      event.remove('temp_field')

      # Parse nested JSON
      require 'json'
      if event.get('data')
        parsed = JSON.parse(event.get('data'))
        event.set('[parsed][data]', parsed)
      end
    "
  }
}
```

### External Ruby file

```ruby
filter {
  ruby {
    path => "/etc/logstash/scripts/transform.rb"
    script_params => { "threshold" => 100 }
  }
}

# /etc/logstash/scripts/transform.rb
def filter(event)
  threshold = event.get('[script_params][threshold]')
  if event.get('value').to_i > threshold
    event.set('above_threshold', true)
  end
  [event]
end
```

---

## Drop Filter

Remove events from the pipeline.

```ruby
filter {
  # Drop health check logs
  if [message] =~ /healthcheck/ {
    drop { }
  }

  # Drop based on field
  if [level] == "DEBUG" {
    drop { }
  }

  # Drop by percentage (sampling)
  if [type] == "metrics" {
    drop {
      percentage => 90  # Drop 90%, keep 10%
    }
  }
}
```

---

## Clone Filter

Duplicate events.

```ruby
filter {
  clone {
    clones => ["copy_for_archive"]
  }
}

output {
  if [type] == "copy_for_archive" {
    s3 { ... }
  } else {
    elasticsearch { ... }
  }
}
```

---

## Aggregate Filter

Combine multiple events.

```ruby
filter {
  aggregate {
    task_id => "%{session_id}"
    code => "
      map['events'] ||= []
      map['events'] << event.get('message')
      map['count'] ||= 0
      map['count'] += 1
    "
    push_map_as_event_on_timeout => true
    timeout => 120
  }
}
```

---

## Complete Filter Pipeline

```ruby
filter {
  # Step 1: Parse JSON if applicable
  if [message] =~ /^\{/ {
    json {
      source => "message"
      skip_on_invalid_json => true
    }
  } else {
    # Step 2: Parse with grok for non-JSON
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:level}\] \[%{DATA:service}\] %{GREEDYDATA:log_message}"
      }
      tag_on_failure => ["_grokparsefailure"]
    }
  }

  # Step 3: Parse timestamp
  if [timestamp] {
    date {
      match => ["timestamp", "ISO8601", "yyyy-MM-dd HH:mm:ss.SSS"]
      target => "@timestamp"
      remove_field => ["timestamp"]
    }
  }

  # Step 4: Normalize level
  mutate {
    uppercase => ["level"]
  }

  # Step 5: Add GeoIP for client IPs
  if [client_ip] and [client_ip] != "127.0.0.1" {
    geoip {
      source => "client_ip"
      target => "geo"
    }
  }

  # Step 6: Calculate response time category
  if [response_time] {
    ruby {
      code => "
        rt = event.get('response_time').to_f
        if rt < 100
          event.set('response_category', 'fast')
        elsif rt < 500
          event.set('response_category', 'normal')
        elsif rt < 1000
          event.set('response_category', 'slow')
        else
          event.set('response_category', 'very_slow')
        end
      "
    }
  }

  # Step 7: Drop debug in production
  if [level] == "DEBUG" and [environment] == "production" {
    drop { }
  }

  # Step 8: Clean up
  mutate {
    remove_field => ["agent", "ecs", "input", "log", "host"]
  }
}
```

---

## Summary

| Filter | Use Case |
|--------|----------|
| `grok` | Parse unstructured text |
| `mutate` | Modify fields |
| `date` | Parse timestamps |
| `json` | Parse JSON |
| `geoip` | Add location from IP |
| `kv` | Parse key=value |
| `dissect` | Fast delimiter parsing |
| `ruby` | Custom logic |
| `drop` | Remove events |
