# Elasticsearch Basics

## Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│              ELASTICSEARCH CONCEPTS                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Cluster                                                    │
│  └── Collection of nodes working together                   │
│                                                             │
│  Node                                                       │
│  └── Single Elasticsearch instance                          │
│                                                             │
│  Index                                                      │
│  └── Collection of documents (like a database)              │
│                                                             │
│  Document                                                   │
│  └── JSON object stored in an index (like a row)            │
│                                                             │
│  Shard                                                      │
│  └── Subdivision of an index for distribution               │
│                                                             │
│  Replica                                                    │
│  └── Copy of a shard for redundancy                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Index Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    INDEX STRUCTURE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Index: logs-2024.01.15                                    │
│  ├── Shard 0 (Primary)                                     │
│  │   └── Replica 0                                         │
│  ├── Shard 1 (Primary)                                     │
│  │   └── Replica 1                                         │
│  └── Shard 2 (Primary)                                     │
│      └── Replica 2                                         │
│                                                             │
│  Documents distributed across shards based on _id hash     │
│                                                             │
│  Shards spread across nodes:                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐          │
│  │   Node 1   │  │   Node 2   │  │   Node 3   │          │
│  │ S0p, S1r   │  │ S1p, S2r   │  │ S2p, S0r   │          │
│  └────────────┘  └────────────┘  └────────────┘          │
│                                                             │
│  p = primary, r = replica                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Documents

### Document Structure

```json
{
  "_index": "logs-2024.01.15",
  "_id": "abc123xyz",
  "_source": {
    "@timestamp": "2024-01-15T10:23:45.000Z",
    "message": "User logged in",
    "level": "INFO",
    "service": "auth-service",
    "user": {
      "id": "12345",
      "email": "john@example.com"
    }
  }
}
```

### Metadata Fields

| Field | Description |
|-------|-------------|
| `_index` | Index where document is stored |
| `_id` | Unique document identifier |
| `_source` | Original JSON document |
| `_version` | Document version number |
| `_score` | Relevance score (in search results) |

---

## Mappings

Mappings define the schema for documents in an index.

### Dynamic Mapping (Automatic)

```bash
# Elasticsearch infers types from first document
PUT logs-2024.01.15/_doc/1
{
  "message": "Hello",      # → text
  "count": 42,             # → long
  "price": 19.99,          # → float
  "active": true,          # → boolean
  "timestamp": "2024-01-15T10:00:00Z"  # → date
}
```

### Explicit Mapping

```bash
# Create index with explicit mapping
PUT logs-2024.01.15
{
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "message": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "level": {
        "type": "keyword"
      },
      "service": {
        "type": "keyword"
      },
      "response_time": {
        "type": "float"
      },
      "user": {
        "properties": {
          "id": { "type": "keyword" },
          "email": { "type": "keyword" }
        }
      },
      "tags": {
        "type": "keyword"
      },
      "location": {
        "type": "geo_point"
      }
    }
  }
}
```

### Common Field Types

| Type | Use Case | Example |
|------|----------|---------|
| `text` | Full-text search | Log messages |
| `keyword` | Exact matches, aggregations | Status codes, IDs |
| `long/integer` | Numeric values | Counts |
| `float/double` | Decimal numbers | Response times |
| `boolean` | True/false | Flags |
| `date` | Timestamps | @timestamp |
| `geo_point` | Coordinates | Location data |
| `ip` | IP addresses | Client IPs |

---

## Index Settings

```bash
# Create index with settings
PUT logs-2024.01.15
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s",
    "index.lifecycle.name": "logs-policy"
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "message": { "type": "text" }
    }
  }
}
```

### Key Settings

| Setting | Description | Default |
|---------|-------------|---------|
| `number_of_shards` | Primary shards (cannot change) | 1 |
| `number_of_replicas` | Replica count (can change) | 1 |
| `refresh_interval` | How often to make docs searchable | 1s |

---

## CRUD Operations

### Create Document

```bash
# Auto-generate ID
POST logs-2024.01.15/_doc
{
  "@timestamp": "2024-01-15T10:23:45.000Z",
  "message": "User logged in",
  "level": "INFO"
}

# Specify ID
PUT logs-2024.01.15/_doc/my-custom-id
{
  "@timestamp": "2024-01-15T10:23:45.000Z",
  "message": "User logged in",
  "level": "INFO"
}

# Create only if not exists
PUT logs-2024.01.15/_create/my-id
{
  "message": "New document"
}
```

### Read Document

```bash
# Get by ID
GET logs-2024.01.15/_doc/abc123

# Get only _source
GET logs-2024.01.15/_source/abc123

# Check if exists
HEAD logs-2024.01.15/_doc/abc123
```

### Update Document

```bash
# Partial update
POST logs-2024.01.15/_update/abc123
{
  "doc": {
    "level": "ERROR",
    "processed": true
  }
}

# Update with script
POST logs-2024.01.15/_update/abc123
{
  "script": {
    "source": "ctx._source.counter += params.count",
    "params": {
      "count": 1
    }
  }
}
```

### Delete Document

```bash
# Delete by ID
DELETE logs-2024.01.15/_doc/abc123

# Delete by query
POST logs-2024.01.15/_delete_by_query
{
  "query": {
    "range": {
      "@timestamp": {
        "lt": "2024-01-01"
      }
    }
  }
}
```

---

## Bulk Operations

```bash
# Bulk indexing (efficient for many documents)
POST _bulk
{"index": {"_index": "logs-2024.01.15"}}
{"@timestamp": "2024-01-15T10:00:00Z", "message": "Log 1"}
{"index": {"_index": "logs-2024.01.15"}}
{"@timestamp": "2024-01-15T10:00:01Z", "message": "Log 2"}
{"index": {"_index": "logs-2024.01.15"}}
{"@timestamp": "2024-01-15T10:00:02Z", "message": "Log 3"}

# Bulk with different operations
POST _bulk
{"index": {"_index": "logs", "_id": "1"}}
{"message": "Create or replace"}
{"create": {"_index": "logs", "_id": "2"}}
{"message": "Create only"}
{"update": {"_index": "logs", "_id": "1"}}
{"doc": {"updated": true}}
{"delete": {"_index": "logs", "_id": "3"}}
```

---

## Index Management

### List Indices

```bash
# List all indices
GET _cat/indices?v

# List indices matching pattern
GET _cat/indices/logs-*?v

# Get index details
GET logs-2024.01.15
```

### Index Operations

```bash
# Create index
PUT my-index

# Delete index
DELETE my-index

# Close index (saves resources)
POST my-index/_close

# Open index
POST my-index/_open

# Refresh index (make docs searchable)
POST my-index/_refresh

# Force merge (optimize)
POST my-index/_forcemerge?max_num_segments=1
```

### Index Aliases

```bash
# Create alias
POST _aliases
{
  "actions": [
    { "add": { "index": "logs-2024.01.15", "alias": "logs-current" }}
  ]
}

# Alias for multiple indices
POST _aliases
{
  "actions": [
    { "add": { "index": "logs-2024.01.*", "alias": "logs-january" }}
  ]
}

# Switch alias (zero-downtime)
POST _aliases
{
  "actions": [
    { "remove": { "index": "logs-2024.01.14", "alias": "logs-current" }},
    { "add": { "index": "logs-2024.01.15", "alias": "logs-current" }}
  ]
}

# Write alias (for indexing)
POST _aliases
{
  "actions": [
    { "add": { "index": "logs-2024.01.15", "alias": "logs-write", "is_write_index": true }}
  ]
}
```

---

## Index Templates

### Create Index Template

```bash
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": {
          "type": "text",
          "fields": {
            "keyword": { "type": "keyword" }
          }
        },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" },
        "host": {
          "properties": {
            "name": { "type": "keyword" },
            "ip": { "type": "ip" }
          }
        }
      }
    },
    "aliases": {
      "logs": {}
    }
  }
}
```

### Component Templates

```bash
# Reusable mapping component
PUT _component_template/logs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" }
      }
    }
  }
}

# Reusable settings component
PUT _component_template/logs-settings
{
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    }
  }
}

# Compose template from components
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "composed_of": ["logs-mappings", "logs-settings"]
}
```

---

## Cluster Health

```bash
# Cluster health
GET _cluster/health

# Response:
{
  "cluster_name": "my-cluster",
  "status": "green",           # green/yellow/red
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 10,
  "active_shards": 20,
  "unassigned_shards": 0
}

# Health status meanings:
# green  - All shards allocated
# yellow - Primary shards OK, some replicas missing
# red    - Some primary shards missing
```

### Cluster Info

```bash
# Node info
GET _cat/nodes?v

# Shard allocation
GET _cat/shards?v

# Cluster stats
GET _cluster/stats

# Pending tasks
GET _cluster/pending_tasks
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Index** | Collection of documents |
| **Document** | JSON object stored in index |
| **Mapping** | Schema definition |
| **Shard** | Horizontal partition |
| **Replica** | Copy for redundancy |
| **Alias** | Virtual index name |
| **Template** | Auto-apply settings to new indices |

### Quick Commands

```bash
# List indices
GET _cat/indices?v

# Create document
POST index/_doc
{ "field": "value" }

# Search
GET index/_search

# Delete index
DELETE index

# Check health
GET _cluster/health
```
