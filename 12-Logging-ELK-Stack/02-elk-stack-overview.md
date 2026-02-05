# ELK Stack Overview

## What is the ELK Stack?

The ELK Stack is a collection of three open-source products for log management:

```
┌─────────────────────────────────────────────────────────────┐
│                      ELK STACK                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  E - Elasticsearch                                          │
│      Search and analytics engine                            │
│      Stores and indexes logs                                │
│                                                             │
│  L - Logstash                                               │
│      Data processing pipeline                               │
│      Ingests, transforms, outputs logs                      │
│                                                             │
│  K - Kibana                                                 │
│      Visualization platform                                 │
│      Dashboards, search UI, management                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Elastic Stack (Modern ELK)

The modern "Elastic Stack" includes additional components:

```
┌─────────────────────────────────────────────────────────────┐
│                    ELASTIC STACK                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                      BEATS                           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ Filebeat │ │Metricbeat│ │Packetbeat│  ...      │   │
│  │  └─────┬────┘ └────┬─────┘ └────┬─────┘           │   │
│  │        │           │            │                  │   │
│  └────────┼───────────┼────────────┼──────────────────┘   │
│           │           │            │                       │
│           └───────────┼────────────┘                       │
│                       ▼                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    LOGSTASH                          │   │
│  │              (Optional processing)                   │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                  │
│                         ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  ELASTICSEARCH                       │   │
│  │                (Storage & Search)                    │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                  │
│                         ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     KIBANA                           │   │
│  │                 (Visualization)                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Details

### Elasticsearch

```
┌────────────────────────────────────────────────────────────┐
│                    ELASTICSEARCH                           │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  What it does:                                             │
│  • Stores log data in indices                             │
│  • Full-text search capabilities                          │
│  • Real-time analytics                                    │
│  • Distributed and scalable                               │
│                                                            │
│  Key Features:                                             │
│  • RESTful API                                            │
│  • JSON document storage                                  │
│  • Inverted index for fast search                         │
│  • Automatic sharding and replication                     │
│  • Aggregations for analytics                             │
│                                                            │
│  Default Port: 9200 (HTTP), 9300 (Transport)              │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Logstash

```
┌────────────────────────────────────────────────────────────┐
│                      LOGSTASH                              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Pipeline:  INPUT → FILTER → OUTPUT                       │
│                                                            │
│  INPUT (where data comes from):                           │
│  • file      - Read from files                            │
│  • beats     - Receive from Beats                         │
│  • kafka     - Consume from Kafka                         │
│  • http      - HTTP endpoint                              │
│  • jdbc      - Database polling                           │
│                                                            │
│  FILTER (transform data):                                 │
│  • grok      - Parse unstructured logs                    │
│  • mutate    - Modify fields                              │
│  • date      - Parse timestamps                           │
│  • geoip     - Add geolocation                            │
│  • json      - Parse JSON                                 │
│                                                            │
│  OUTPUT (where data goes):                                │
│  • elasticsearch - Index in ES                            │
│  • file          - Write to file                          │
│  • kafka         - Send to Kafka                          │
│  • stdout        - Console output                         │
│                                                            │
│  Default Port: 5044 (Beats input)                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Kibana

```
┌────────────────────────────────────────────────────────────┐
│                       KIBANA                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  What it does:                                             │
│  • Web UI for Elasticsearch                               │
│  • Search and explore logs (Discover)                     │
│  • Create visualizations                                  │
│  • Build dashboards                                       │
│  • Manage Elasticsearch                                   │
│                                                            │
│  Key Features:                                             │
│  • Discover    - Search logs                              │
│  • Visualize   - Charts, graphs                           │
│  • Dashboard   - Combine visualizations                   │
│  • Lens        - Drag-and-drop viz builder               │
│  • Dev Tools   - Console for ES queries                   │
│  • Management  - Index patterns, ILM                      │
│  • Alerting    - Rule-based alerts                        │
│                                                            │
│  Default Port: 5601                                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Beats Family

```
┌────────────────────────────────────────────────────────────┐
│                        BEATS                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Lightweight data shippers:                               │
│                                                            │
│  Filebeat     - Log files                                 │
│                 Tail logs, ship to ES/Logstash           │
│                                                            │
│  Metricbeat   - System and service metrics               │
│                 CPU, memory, Docker, K8s                  │
│                                                            │
│  Packetbeat   - Network data                              │
│                 HTTP, DNS, MySQL protocols               │
│                                                            │
│  Heartbeat    - Uptime monitoring                         │
│                 HTTP, TCP, ICMP probes                    │
│                                                            │
│  Auditbeat    - Audit data                                │
│                 File integrity, system events            │
│                                                            │
│  Winlogbeat   - Windows Event logs                        │
│                 Security, Application, System logs       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Data Flow Architecture

### Simple Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 SIMPLE ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Applications                                               │
│  ┌─────┐ ┌─────┐ ┌─────┐                                  │
│  │App 1│ │App 2│ │App 3│                                  │
│  └──┬──┘ └──┬──┘ └──┬──┘                                  │
│     │       │       │                                       │
│     └───────┼───────┘                                       │
│             │ Filebeat                                      │
│             ▼                                               │
│     ┌───────────────┐                                      │
│     │ Elasticsearch │                                      │
│     └───────┬───────┘                                      │
│             │                                               │
│             ▼                                               │
│     ┌───────────────┐                                      │
│     │    Kibana     │                                      │
│     └───────────────┘                                      │
│                                                             │
│  Best for: Small deployments, simple log formats           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Production Architecture

```
┌─────────────────────────────────────────────────────────────┐
│               PRODUCTION ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Applications                                               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                 │
│  │App 1│ │App 2│ │App 3│ │App 4│ │App 5│                 │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘                 │
│     │       │       │       │       │                       │
│     └───────┴───────┼───────┴───────┘                       │
│                     │ Filebeat (on each host)               │
│                     ▼                                       │
│     ┌───────────────────────────────────┐                  │
│     │             Kafka                  │ (Buffer)        │
│     │         (Message Queue)            │                  │
│     └───────────────┬───────────────────┘                  │
│                     │                                       │
│                     ▼                                       │
│     ┌───────────────────────────────────┐                  │
│     │           Logstash                 │ (Processing)    │
│     │     (Parse, Transform, Enrich)     │                  │
│     └───────────────┬───────────────────┘                  │
│                     │                                       │
│                     ▼                                       │
│     ┌───────────────────────────────────┐                  │
│     │    Elasticsearch Cluster           │ (Storage)       │
│     │  ┌────────┐ ┌────────┐ ┌────────┐│                  │
│     │  │ Node 1 │ │ Node 2 │ │ Node 3 ││                  │
│     │  └────────┘ └────────┘ └────────┘│                  │
│     └───────────────┬───────────────────┘                  │
│                     │                                       │
│                     ▼                                       │
│     ┌───────────────────────────────────┐                  │
│     │            Kibana                  │ (Visualization) │
│     │        (Load Balanced)             │                  │
│     └───────────────────────────────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## ELK vs Alternatives

### Comparison

| Feature | ELK Stack | Loki + Grafana | Splunk | Datadog |
|---------|-----------|----------------|--------|---------|
| **Cost** | Free (OSS) | Free (OSS) | Expensive | Per-host pricing |
| **Setup** | Complex | Simple | Easy | SaaS |
| **Full-text Search** | Excellent | Limited | Excellent | Good |
| **Scalability** | High | High | High | High |
| **Learning Curve** | Moderate | Low | Moderate | Low |
| **Log Parsing** | Flexible | Label-based | Flexible | Auto-detect |
| **Storage** | High | Low | High | SaaS |

### When to Use ELK

```
✓ Need full-text search across logs
✓ Complex log parsing requirements
✓ Large-scale log analytics
✓ On-premise requirements
✓ Custom dashboards and visualizations
✓ Integration with existing Elastic ecosystem
```

### When to Consider Alternatives

```
Loki:
  - Kubernetes-native logging
  - Already using Grafana
  - Simple label-based queries sufficient

Splunk:
  - Enterprise support required
  - Advanced security analytics (SIEM)
  - Budget allows

Datadog/Cloud Services:
  - Want managed solution
  - Don't want to manage infrastructure
```

---

## Licensing

### Open Source (OSS)

```
Elasticsearch, Logstash, Kibana:
- Apache 2.0 / SSPL (Server Side Public License)
- Free to use
- Some features require subscription
```

### Elastic License (Basic - Free)

```
Included features:
- Core Elasticsearch
- Core Kibana
- Beats
- Basic security (TLS)
- Index lifecycle management
- Alerting (Watcher)
```

### Paid Tiers (Gold, Platinum, Enterprise)

```
Additional features:
- Machine Learning
- Advanced security (RBAC, SSO)
- Cross-cluster replication
- Searchable snapshots
- Enterprise support
```

---

## Resource Requirements

### Minimum (Development)

```
Elasticsearch: 2GB RAM, 2 CPU, 20GB disk
Logstash:      1GB RAM, 1 CPU
Kibana:        512MB RAM, 1 CPU
Filebeat:      128MB RAM
```

### Recommended (Production)

```
Elasticsearch (per node):
  - 16-32GB RAM (half for JVM heap)
  - 4-8 CPU cores
  - SSD storage
  - 3+ nodes for HA

Logstash:
  - 4GB RAM
  - 2-4 CPU cores
  - Scale horizontally

Kibana:
  - 2GB RAM
  - 2 CPU cores
  - Can run multiple instances
```

---

## Default Ports

```
┌────────────────────────────────────────────────────────────┐
│                    DEFAULT PORTS                           │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Component        │ Port  │ Protocol   │ Purpose          │
│  ─────────────────┼───────┼────────────┼──────────────────│
│  Elasticsearch    │ 9200  │ HTTP       │ REST API         │
│  Elasticsearch    │ 9300  │ TCP        │ Node transport   │
│  Kibana           │ 5601  │ HTTP       │ Web UI           │
│  Logstash         │ 5044  │ TCP        │ Beats input      │
│  Logstash         │ 9600  │ HTTP       │ Monitoring API   │
│  Filebeat         │ -     │ -          │ Agent (no port)  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Summary

| Component | Role | Port |
|-----------|------|------|
| **Elasticsearch** | Storage & Search | 9200 |
| **Logstash** | Processing Pipeline | 5044 |
| **Kibana** | Visualization UI | 5601 |
| **Filebeat** | Log Shipper | Agent |
| **Metricbeat** | Metrics Shipper | Agent |

### Quick Reference

```
ELK = Elasticsearch + Logstash + Kibana
Elastic Stack = ELK + Beats

Simple:    App → Filebeat → Elasticsearch → Kibana
With Proc: App → Filebeat → Logstash → Elasticsearch → Kibana
Buffered:  App → Filebeat → Kafka → Logstash → ES → Kibana
```
