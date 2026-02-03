# Load Balancing

## What is Load Balancing?

Load balancing distributes incoming traffic across multiple servers to ensure reliability and performance.

```
                                    ┌──────────────┐
                              ┌────►│   Server 1   │
                              │     └──────────────┘
┌──────────┐    ┌──────────┐  │     ┌──────────────┐
│ Clients  │───►│   Load   │──┼────►│   Server 2   │
└──────────┘    │ Balancer │  │     └──────────────┘
                └──────────┘  │     ┌──────────────┐
                              └────►│   Server 3   │
                                    └──────────────┘
```

## Why Load Balancing?

| Benefit | Description |
|---------|-------------|
| High Availability | If one server fails, others handle traffic |
| Scalability | Add more servers to handle growth |
| Performance | Distribute load for faster response |
| Flexibility | Perform maintenance without downtime |
| Redundancy | No single point of failure |

## Load Balancer Types

### By OSI Layer

| Layer | Name | Balances Based On | Example |
|-------|------|-------------------|---------|
| L4 | Transport | IP, Port, Protocol | HAProxy TCP mode |
| L7 | Application | URL, Headers, Cookies | Nginx, HAProxy HTTP |

### Layer 4 vs Layer 7

```
Layer 4 (Transport):
┌────────────────────────────────────────────────────────────┐
│  Can see: Source IP, Dest IP, Source Port, Dest Port      │
│  Cannot see: URL, Headers, Cookies, Content               │
│  Fast, simple, less CPU                                   │
└────────────────────────────────────────────────────────────┘

Layer 7 (Application):
┌────────────────────────────────────────────────────────────┐
│  Can see: Everything including HTTP headers, URL, content │
│  Features: URL routing, header modification, SSL termination│
│  More flexible, more CPU                                  │
└────────────────────────────────────────────────────────────┘
```

### By Deployment

| Type | Description |
|------|-------------|
| Hardware | Physical appliance (F5, Citrix) |
| Software | Application-based (Nginx, HAProxy) |
| Cloud | Provider service (AWS ALB/NLB, GCP LB) |
| DNS | DNS-based distribution |

## Load Balancing Algorithms

### Round Robin

Distributes requests sequentially.

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1  (cycle repeats)
```

**Pros**: Simple, even distribution
**Cons**: Ignores server capacity and current load

### Weighted Round Robin

Servers with higher weight get more requests.

```
Server 1 (weight=3): Gets 3 requests
Server 2 (weight=2): Gets 2 requests
Server 3 (weight=1): Gets 1 request
```

**Use case**: Servers with different capacities

### Least Connections

Sends to server with fewest active connections.

```
Server 1: 5 connections  ← New request goes here
Server 2: 10 connections
Server 3: 8 connections
```

**Use case**: When request processing time varies

### Weighted Least Connections

Combines weights with connection count.

### IP Hash

Same client IP always goes to same server.

```
Hash(Client IP) → Server selection
```

**Use case**: Session persistence without cookies

### Least Response Time

Routes to server with fastest response time.

### Random

Randomly selects a server.

## Session Persistence (Sticky Sessions)

Ensures client requests go to the same backend server.

### Methods

| Method | Description |
|--------|-------------|
| Source IP | Hash client IP |
| Cookie | Insert/read session cookie |
| URL Parameter | Session ID in URL |
| SSL Session ID | Use TLS session |

### Cookie-Based Example

```
┌────────────┐    First Request    ┌──────────┐
│   Client   │ ─────────────────►  │    LB    │ ──► Server 2
└────────────┘                     └──────────┘
                                        │
     Response: Set-Cookie: SERVERID=srv2│
◄───────────────────────────────────────┘

┌────────────┐    Next Request     ┌──────────┐
│   Client   │ Cookie: SERVERID=srv2 │    LB    │ ──► Server 2
└────────────┘ ─────────────────►  └──────────┘
```

## Health Checks

Load balancers check if backend servers are healthy.

### Health Check Types

| Type | Description |
|------|-------------|
| TCP | Can connect to port? |
| HTTP | Returns 200 OK? |
| HTTPS | SSL + HTTP check |
| Custom | Script-based check |

### Configuration Example

```
Check every 5 seconds
Server must respond in 2 seconds
3 failures = server is DOWN
2 successes = server is UP
```

## Common Load Balancers

### Nginx

```nginx
# /etc/nginx/nginx.conf

http {
    upstream backend {
        # Round robin (default)
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
        server 192.168.1.12:8080;
    }

    # Weighted
    upstream backend_weighted {
        server 192.168.1.10:8080 weight=3;
        server 192.168.1.11:8080 weight=2;
        server 192.168.1.12:8080 weight=1;
    }

    # Least connections
    upstream backend_least {
        least_conn;
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
    }

    # IP Hash (sticky)
    upstream backend_sticky {
        ip_hash;
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
    }

    # Health checks (Nginx Plus or OpenResty)
    upstream backend_health {
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
        # Backup server
        server 192.168.1.12:8080 backup;
        # Mark as down
        server 192.168.1.13:8080 down;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

### HAProxy

```bash
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 4096

defaults
    mode http
    log global
    option httplog
    option dontlognull
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200
    server server1 192.168.1.10:8080 check
    server server2 192.168.1.11:8080 check
    server server3 192.168.1.12:8080 check

# Layer 4 (TCP) load balancing
frontend tcp_front
    bind *:3306
    mode tcp
    default_backend mysql_back

backend mysql_back
    mode tcp
    balance leastconn
    server db1 192.168.1.20:3306 check
    server db2 192.168.1.21:3306 check
```

### Balance Algorithms in HAProxy

| Algorithm | Description |
|-----------|-------------|
| roundrobin | Round robin |
| leastconn | Least connections |
| source | IP hash |
| uri | URI hash |
| hdr(name) | Header hash |
| first | First available server |

## SSL/TLS Termination

Load balancer handles SSL, backends use HTTP.

```
Client ──── HTTPS ────► Load Balancer ──── HTTP ────► Servers
                              │
                        SSL Certificate
```

### Benefits

- Offload SSL processing from servers
- Centralized certificate management
- Can inspect/modify HTTP traffic

### Nginx SSL Termination

```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://backend;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

## Cloud Load Balancers

### AWS

| Service | Layer | Use Case |
|---------|-------|----------|
| ALB (Application) | L7 | HTTP/HTTPS, URL routing |
| NLB (Network) | L4 | TCP/UDP, high performance |
| CLB (Classic) | L4/L7 | Legacy |
| GWLB (Gateway) | L3 | Firewalls, IDS/IPS |

### GCP

| Service | Layer |
|---------|-------|
| HTTP(S) Load Balancer | L7 |
| TCP/UDP Load Balancer | L4 |
| Internal Load Balancer | L4 |

### Azure

| Service | Layer |
|---------|-------|
| Application Gateway | L7 |
| Load Balancer | L4 |
| Traffic Manager | DNS |
| Front Door | L7 Global |

## Quick Reference

| Algorithm | Best For |
|-----------|----------|
| Round Robin | Equal servers, short requests |
| Weighted | Different capacity servers |
| Least Connections | Varying request lengths |
| IP Hash | Session persistence |

| Load Balancer | Type |
|---------------|------|
| Nginx | L7 (L4 with stream) |
| HAProxy | L4 and L7 |
| AWS ALB | L7 |
| AWS NLB | L4 |

---

**Previous:** [09-nat-port-forwarding.md](09-nat-port-forwarding.md) | **Next:** [11-proxy-reverse-proxy.md](11-proxy-reverse-proxy.md)
