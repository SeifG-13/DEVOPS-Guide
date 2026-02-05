# Elasticsearch Queries

## Query DSL Overview

Elasticsearch uses a JSON-based Query DSL (Domain Specific Language) for searching.

```
┌─────────────────────────────────────────────────────────────┐
│                    QUERY STRUCTURE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  GET index/_search                                          │
│  {                                                          │
│    "query": { ... },        // Search criteria              │
│    "sort": [ ... ],         // Ordering                     │
│    "from": 0,               // Pagination start             │
│    "size": 10,              // Results per page             │
│    "_source": [ ... ],      // Fields to return             │
│    "aggs": { ... }          // Aggregations                 │
│  }                                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Basic Queries

### Match All

```bash
# Return all documents
GET logs-*/_search
{
  "query": {
    "match_all": {}
  }
}
```

### Match (Full-Text Search)

```bash
# Full-text search on a field
GET logs-*/_search
{
  "query": {
    "match": {
      "message": "error connection failed"
    }
  }
}

# Match with operator
GET logs-*/_search
{
  "query": {
    "match": {
      "message": {
        "query": "error connection",
        "operator": "and"  # Both terms must match
      }
    }
  }
}
```

### Match Phrase

```bash
# Exact phrase match
GET logs-*/_search
{
  "query": {
    "match_phrase": {
      "message": "connection refused"
    }
  }
}

# With slop (allow words between)
GET logs-*/_search
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "user logged",
        "slop": 2  # Allow 2 words between
      }
    }
  }
}
```

### Term (Exact Match)

```bash
# Exact value match (keyword fields)
GET logs-*/_search
{
  "query": {
    "term": {
      "level": "ERROR"
    }
  }
}

# Multiple values
GET logs-*/_search
{
  "query": {
    "terms": {
      "level": ["ERROR", "WARN"]
    }
  }
}
```

---

## Range Queries

```bash
# Numeric range
GET logs-*/_search
{
  "query": {
    "range": {
      "response_time": {
        "gte": 1000,
        "lt": 5000
      }
    }
  }
}

# Date range
GET logs-*/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2024-01-15T00:00:00",
        "lte": "2024-01-15T23:59:59"
      }
    }
  }
}

# Relative date range
GET logs-*/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-1h",
        "lte": "now"
      }
    }
  }
}

# Date math
GET logs-*/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-7d/d",    # 7 days ago, rounded to day
        "lt": "now/d"          # Today, rounded to day
      }
    }
  }
}
```

---

## Boolean Queries

Combine multiple queries with boolean logic.

```bash
GET logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        # All must match (AND)
        { "match": { "message": "error" } },
        { "term": { "service": "api" } }
      ],
      "should": [
        # At least one should match (OR)
        { "term": { "level": "ERROR" } },
        { "term": { "level": "FATAL" } }
      ],
      "must_not": [
        # None must match (NOT)
        { "term": { "environment": "development" } }
      ],
      "filter": [
        # Must match, but doesn't affect score
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}
```

### Boolean Query Examples

```bash
# Find errors in production from last hour
GET logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "level": "ERROR" } }
      ],
      "filter": [
        { "term": { "environment": "production" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}

# Find login failures excluding test users
GET logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "login failed" } }
      ],
      "must_not": [
        { "wildcard": { "user.email": "*@test.com" } }
      ]
    }
  }
}
```

---

## Wildcard and Regex

```bash
# Wildcard query
GET logs-*/_search
{
  "query": {
    "wildcard": {
      "host.name": "web-server-*"
    }
  }
}

# Regex query
GET logs-*/_search
{
  "query": {
    "regexp": {
      "path": "/api/v[0-9]+/users.*"
    }
  }
}

# Prefix query
GET logs-*/_search
{
  "query": {
    "prefix": {
      "service": "auth-"
    }
  }
}
```

---

## Exists Query

```bash
# Documents where field exists
GET logs-*/_search
{
  "query": {
    "exists": {
      "field": "error.stack_trace"
    }
  }
}

# Documents where field does NOT exist
GET logs-*/_search
{
  "query": {
    "bool": {
      "must_not": {
        "exists": {
          "field": "error.stack_trace"
        }
      }
    }
  }
}
```

---

## Sorting

```bash
# Sort by timestamp descending
GET logs-*/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "@timestamp": "desc" }
  ]
}

# Multiple sort criteria
GET logs-*/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "level": "asc" },
    { "@timestamp": "desc" }
  ]
}

# Sort by relevance score (default)
GET logs-*/_search
{
  "query": { "match": { "message": "error" } },
  "sort": [
    "_score",
    { "@timestamp": "desc" }
  ]
}
```

---

## Pagination

```bash
# Basic pagination
GET logs-*/_search
{
  "query": { "match_all": {} },
  "from": 0,    # Start from first result
  "size": 10    # Return 10 results
}

# Page 2
GET logs-*/_search
{
  "query": { "match_all": {} },
  "from": 10,   # Skip first 10
  "size": 10
}

# Deep pagination with search_after (recommended)
GET logs-*/_search
{
  "query": { "match_all": {} },
  "size": 10,
  "sort": [
    { "@timestamp": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2024-01-15T10:00:00.000Z", "abc123"]
}
```

---

## Source Filtering

```bash
# Include specific fields
GET logs-*/_search
{
  "query": { "match_all": {} },
  "_source": ["@timestamp", "message", "level"]
}

# Exclude fields
GET logs-*/_search
{
  "query": { "match_all": {} },
  "_source": {
    "excludes": ["raw_message", "full_stack_trace"]
  }
}

# Disable _source (metadata only)
GET logs-*/_search
{
  "query": { "match_all": {} },
  "_source": false
}
```

---

## Aggregations

### Bucket Aggregations

```bash
# Count by level
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "logs_by_level": {
      "terms": {
        "field": "level",
        "size": 10
      }
    }
  }
}

# Histogram by time
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "logs_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "hour"
      }
    }
  }
}

# Range buckets
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "response_time_ranges": {
      "range": {
        "field": "response_time",
        "ranges": [
          { "to": 100 },
          { "from": 100, "to": 500 },
          { "from": 500, "to": 1000 },
          { "from": 1000 }
        ]
      }
    }
  }
}
```

### Metric Aggregations

```bash
# Statistics
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "response_stats": {
      "stats": {
        "field": "response_time"
      }
    }
  }
}

# Percentiles
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "latency_percentiles": {
      "percentiles": {
        "field": "response_time",
        "percents": [50, 90, 95, 99]
      }
    }
  }
}

# Average
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "avg_response_time": {
      "avg": {
        "field": "response_time"
      }
    }
  }
}
```

### Nested Aggregations

```bash
# Errors by service with average response time
GET logs-*/_search
{
  "size": 0,
  "query": {
    "term": { "level": "ERROR" }
  },
  "aggs": {
    "by_service": {
      "terms": {
        "field": "service",
        "size": 10
      },
      "aggs": {
        "avg_response": {
          "avg": { "field": "response_time" }
        },
        "error_count": {
          "value_count": { "field": "_id" }
        }
      }
    }
  }
}

# Logs over time by level
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "hour"
      },
      "aggs": {
        "by_level": {
          "terms": {
            "field": "level"
          }
        }
      }
    }
  }
}
```

---

## Common Log Queries

### Find Errors

```bash
# All errors in last hour
GET logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "terms": { "level": ["ERROR", "FATAL"] } }
      ],
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "sort": [{ "@timestamp": "desc" }]
}
```

### Search by Trace ID

```bash
# Find all logs for a request
GET logs-*/_search
{
  "query": {
    "term": {
      "trace.id": "abc-123-xyz"
    }
  },
  "sort": [{ "@timestamp": "asc" }]
}
```

### Error Rate by Service

```bash
GET logs-*/_search
{
  "size": 0,
  "query": {
    "range": { "@timestamp": { "gte": "now-24h" } }
  },
  "aggs": {
    "by_service": {
      "terms": { "field": "service" },
      "aggs": {
        "total": { "value_count": { "field": "_id" } },
        "errors": {
          "filter": { "term": { "level": "ERROR" } }
        }
      }
    }
  }
}
```

### Slow Requests

```bash
GET logs-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "response_time": { "gte": 1000 } } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "sort": [{ "response_time": "desc" }],
  "size": 100
}
```

### Top Error Messages

```bash
GET logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [{ "term": { "level": "ERROR" } }],
      "filter": [{ "range": { "@timestamp": { "gte": "now-24h" } } }]
    }
  },
  "aggs": {
    "top_errors": {
      "terms": {
        "field": "message.keyword",
        "size": 20
      }
    }
  }
}
```

---

## Query String Syntax

```bash
# Simple query string (Kibana search bar)
GET logs-*/_search
{
  "query": {
    "query_string": {
      "query": "level:ERROR AND service:api"
    }
  }
}

# Query string examples:
# level:ERROR
# message:"connection refused"
# level:(ERROR OR WARN)
# @timestamp:[2024-01-15 TO 2024-01-16]
# response_time:>1000
# message:error*
# _exists_:error.stack
```

---

## Summary

| Query Type | Use Case |
|------------|----------|
| `match` | Full-text search |
| `term` | Exact value match |
| `range` | Numeric/date ranges |
| `bool` | Combine queries |
| `exists` | Field existence |
| `wildcard` | Pattern matching |

### Quick Reference

```bash
# Full-text search
{ "match": { "message": "error" } }

# Exact match
{ "term": { "level": "ERROR" } }

# Date range
{ "range": { "@timestamp": { "gte": "now-1h" } } }

# Boolean combination
{ "bool": { "must": [...], "filter": [...] } }

# Aggregation
{ "aggs": { "name": { "terms": { "field": "level" } } } }
```
