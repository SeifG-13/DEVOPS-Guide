# Proxy & Reverse Proxy

## What is a Proxy?

A proxy is an intermediary server between clients and other servers.

## Forward Proxy

Sits between clients and the internet, acting on behalf of **clients**.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│    Internal Network                              Internet         │
│                                                                   │
│    ┌────────┐                                   ┌──────────┐     │
│    │Client 1│──┐                          ┌────►│ Server A │     │
│    └────────┘  │     ┌────────────┐       │     └──────────┘     │
│    ┌────────┐  ├────►│  Forward   │───────┤     ┌──────────┐     │
│    │Client 2│──┤     │   Proxy    │       ├────►│ Server B │     │
│    └────────┘  │     └────────────┘       │     └──────────┘     │
│    ┌────────┐  │                          │     ┌──────────┐     │
│    │Client 3│──┘                          └────►│ Server C │     │
│    └────────┘                                   └──────────┘     │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Forward Proxy Use Cases

| Use Case | Description |
|----------|-------------|
| Privacy | Hide client IP from destination |
| Access Control | Control which sites users can access |
| Caching | Cache frequently accessed content |
| Bypass Restrictions | Access geo-blocked content |
| Monitoring | Log and monitor user traffic |
| Bandwidth Savings | Cache reduces external traffic |

### Forward Proxy Examples

- Squid
- Privoxy
- Corporate proxy servers
- VPN (acts as proxy)

## Reverse Proxy

Sits between the internet and backend servers, acting on behalf of **servers**.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│    Internet                              Internal Network         │
│                                                                   │
│    ┌────────┐                                   ┌──────────┐     │
│    │Client 1│──┐                          ┌────►│ Server 1 │     │
│    └────────┘  │     ┌────────────┐       │     └──────────┘     │
│    ┌────────┐  ├────►│  Reverse   │───────┤     ┌──────────┐     │
│    │Client 2│──┤     │   Proxy    │       ├────►│ Server 2 │     │
│    └────────┘  │     └────────────┘       │     └──────────┘     │
│    ┌────────┐  │                          │     ┌──────────┐     │
│    │Client 3│──┘                          └────►│ Server 3 │     │
│    └────────┘                                   └──────────┘     │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Reverse Proxy Use Cases

| Use Case | Description |
|----------|-------------|
| Load Balancing | Distribute traffic across servers |
| SSL Termination | Handle HTTPS at proxy |
| Caching | Cache static content |
| Compression | Compress responses |
| Security | Hide backend infrastructure |
| DDoS Protection | Filter malicious traffic |
| URL Rewriting | Modify request URLs |
| A/B Testing | Route to different versions |

### Reverse Proxy Examples

- Nginx
- HAProxy
- Apache (mod_proxy)
- Traefik
- Caddy
- AWS ALB/CloudFront

## Forward vs Reverse Proxy

| Feature | Forward Proxy | Reverse Proxy |
|---------|---------------|---------------|
| Acts for | Clients | Servers |
| Client knows | Proxy exists | Thinks talking to server |
| Server sees | Proxy IP | Proxy IP |
| Common use | Corporate, privacy | Web servers, CDN |
| Placement | Near clients | Near servers |

## Nginx Reverse Proxy

### Basic Configuration

```nginx
# /etc/nginx/sites-available/example.conf

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Multiple Backends

```nginx
server {
    listen 80;
    server_name example.com;

    # API requests
    location /api/ {
        proxy_pass http://localhost:3000/;
    }

    # Static files
    location /static/ {
        proxy_pass http://localhost:8080/;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://localhost:9000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Default
    location / {
        proxy_pass http://localhost:4000;
    }
}
```

### With SSL Termination

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### Caching

```nginx
# Define cache zone
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_cache my_cache;
        proxy_cache_valid 200 60m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
        add_header X-Cache-Status $upstream_cache_status;
        proxy_pass http://localhost:3000;
    }

    # Bypass cache for specific paths
    location /api/ {
        proxy_cache_bypass 1;
        proxy_pass http://localhost:3000;
    }
}
```

## Apache Reverse Proxy

```apache
# Enable modules
# a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests

<VirtualHost *:80>
    ServerName example.com

    ProxyPreserveHost On
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/

    # Load balancing
    <Proxy "balancer://mycluster">
        BalancerMember http://localhost:3001
        BalancerMember http://localhost:3002
        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPass /api balancer://mycluster
    ProxyPassReverse /api balancer://mycluster
</VirtualHost>
```

## HAProxy Reverse Proxy

```haproxy
# /etc/haproxy/haproxy.cfg

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.pem

    # Redirect HTTP to HTTPS
    http-request redirect scheme https unless { ssl_fc }

    # Route based on path
    acl is_api path_beg /api
    acl is_static path_beg /static

    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 192.168.1.10:3000 check
    server web2 192.168.1.11:3000 check

backend api_servers
    balance leastconn
    server api1 192.168.1.20:4000 check
    server api2 192.168.1.21:4000 check

backend static_servers
    balance roundrobin
    server static1 192.168.1.30:8080 check
```

## Traefik (Container-Native)

```yaml
# docker-compose.yml with Traefik

version: '3'

services:
  traefik:
    image: traefik:v2.9
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  webapp:
    image: myapp
    labels:
      - "traefik.http.routers.webapp.rule=Host(`example.com`)"
      - "traefik.http.services.webapp.loadbalancer.server.port=3000"

  api:
    image: myapi
    labels:
      - "traefik.http.routers.api.rule=Host(`example.com`) && PathPrefix(`/api`)"
      - "traefik.http.services.api.loadbalancer.server.port=4000"
```

## Common Headers

### Headers Set by Proxy

| Header | Purpose |
|--------|---------|
| X-Real-IP | Original client IP |
| X-Forwarded-For | Chain of proxies |
| X-Forwarded-Proto | Original protocol (http/https) |
| X-Forwarded-Host | Original Host header |
| X-Forwarded-Port | Original port |

### Reading Client IP in Application

```
# With proxy, check headers in this order:
1. X-Forwarded-For (first IP in list)
2. X-Real-IP
3. Remote address (will be proxy IP)
```

## Squid (Forward Proxy)

```bash
# Install
sudo apt install squid

# Configuration
sudo vim /etc/squid/squid.conf
```

```squid
# /etc/squid/squid.conf

# Listen port
http_port 3128

# Access control
acl localnet src 192.168.1.0/24
http_access allow localnet
http_access deny all

# Cache settings
cache_dir ufs /var/spool/squid 1000 16 256
maximum_object_size 100 MB

# Logging
access_log /var/log/squid/access.log
```

## Quick Reference

| Type | Acts For | Hides | Use Case |
|------|----------|-------|----------|
| Forward Proxy | Clients | Client IP | Corporate access control |
| Reverse Proxy | Servers | Server IP | Load balancing, SSL |

| Software | Type |
|----------|------|
| Nginx | Reverse |
| HAProxy | Reverse |
| Traefik | Reverse |
| Squid | Forward |
| Apache mod_proxy | Both |

---

**Previous:** [10-load-balancing.md](10-load-balancing.md) | **Next:** [12-vpn-basics.md](12-vpn-basics.md)
