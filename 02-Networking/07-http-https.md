# HTTP & HTTPS

## What is HTTP?

HTTP (Hypertext Transfer Protocol) is the foundation of data communication on the web.

```
┌────────────┐         Request          ┌────────────┐
│            │ ───────────────────────► │            │
│   Client   │    GET /index.html       │   Server   │
│  (Browser) │                          │   (Web)    │
│            │ ◄─────────────────────── │            │
└────────────┘         Response         └────────────┘
                   200 OK + HTML
```

## HTTP Versions

| Version | Year | Features |
|---------|------|----------|
| HTTP/0.9 | 1991 | GET only, no headers |
| HTTP/1.0 | 1996 | Headers, POST, status codes |
| HTTP/1.1 | 1997 | Keep-alive, chunked transfer, host header |
| HTTP/2 | 2015 | Binary, multiplexing, server push |
| HTTP/3 | 2022 | QUIC (UDP-based), faster |

## HTTP Request Structure

```
GET /api/users HTTP/1.1              ← Request Line
Host: api.example.com                ← Headers
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer token123
                                     ← Empty Line
{"name": "John"}                     ← Body (optional)
```

### Request Line Components

```
GET    /api/users?id=123    HTTP/1.1
│      │                    │
│      │                    └── Protocol Version
│      └── URI (Path + Query String)
└── HTTP Method
```

## HTTP Methods

| Method | Purpose | Body | Idempotent | Safe |
|--------|---------|------|------------|------|
| GET | Retrieve resource | No | Yes | Yes |
| POST | Create resource | Yes | No | No |
| PUT | Update/Replace resource | Yes | Yes | No |
| PATCH | Partial update | Yes | No | No |
| DELETE | Remove resource | Optional | Yes | No |
| HEAD | GET without body | No | Yes | Yes |
| OPTIONS | Show allowed methods | No | Yes | Yes |
| TRACE | Echo request | No | Yes | Yes |

### CRUD Mapping

| Operation | HTTP Method |
|-----------|-------------|
| Create | POST |
| Read | GET |
| Update | PUT / PATCH |
| Delete | DELETE |

## HTTP Response Structure

```
HTTP/1.1 200 OK                      ← Status Line
Date: Mon, 15 Jan 2024 12:00:00 GMT  ← Headers
Content-Type: application/json
Content-Length: 27
Cache-Control: max-age=3600
                                     ← Empty Line
{"id": 1, "name": "John"}            ← Body
```

## HTTP Status Codes

### Categories

| Range | Category | Description |
|-------|----------|-------------|
| 1xx | Informational | Request received, continuing |
| 2xx | Success | Request successful |
| 3xx | Redirection | Further action needed |
| 4xx | Client Error | Bad request from client |
| 5xx | Server Error | Server failed to fulfill |

### Common Status Codes

**2xx Success**
| Code | Name | Description |
|------|------|-------------|
| 200 | OK | Request successful |
| 201 | Created | Resource created |
| 204 | No Content | Success, no body returned |

**3xx Redirection**
| Code | Name | Description |
|------|------|-------------|
| 301 | Moved Permanently | Resource permanently moved |
| 302 | Found | Temporary redirect |
| 304 | Not Modified | Use cached version |
| 307 | Temporary Redirect | Preserve method |
| 308 | Permanent Redirect | Preserve method |

**4xx Client Errors**
| Code | Name | Description |
|------|------|-------------|
| 400 | Bad Request | Invalid syntax |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | Access denied |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | Method not supported |
| 408 | Request Timeout | Client too slow |
| 409 | Conflict | Request conflicts with state |
| 429 | Too Many Requests | Rate limited |

**5xx Server Errors**
| Code | Name | Description |
|------|------|-------------|
| 500 | Internal Server Error | Generic server error |
| 502 | Bad Gateway | Invalid upstream response |
| 503 | Service Unavailable | Server overloaded/maintenance |
| 504 | Gateway Timeout | Upstream timeout |

## Common HTTP Headers

### Request Headers

| Header | Purpose | Example |
|--------|---------|---------|
| Host | Target host | Host: api.example.com |
| User-Agent | Client info | User-Agent: Mozilla/5.0 |
| Accept | Accepted content types | Accept: application/json |
| Accept-Language | Preferred language | Accept-Language: en-US |
| Accept-Encoding | Accepted compression | Accept-Encoding: gzip |
| Authorization | Auth credentials | Authorization: Bearer xyz |
| Cookie | Session cookies | Cookie: session=abc123 |
| Content-Type | Body format | Content-Type: application/json |
| Content-Length | Body size | Content-Length: 348 |

### Response Headers

| Header | Purpose | Example |
|--------|---------|---------|
| Content-Type | Body format | Content-Type: text/html |
| Content-Length | Body size | Content-Length: 1234 |
| Content-Encoding | Compression used | Content-Encoding: gzip |
| Cache-Control | Caching rules | Cache-Control: max-age=3600 |
| Set-Cookie | Set client cookie | Set-Cookie: session=xyz |
| Location | Redirect URL | Location: /new-page |
| Server | Server software | Server: nginx/1.18.0 |
| X-Powered-By | Framework info | X-Powered-By: Express |

### Security Headers

| Header | Purpose |
|--------|---------|
| Strict-Transport-Security | Force HTTPS |
| Content-Security-Policy | Prevent XSS |
| X-Content-Type-Options | Prevent MIME sniffing |
| X-Frame-Options | Prevent clickjacking |
| X-XSS-Protection | XSS filter |

## HTTPS (HTTP Secure)

HTTPS = HTTP + TLS/SSL encryption

```
┌────────────┐                          ┌────────────┐
│            │  ═══ Encrypted ═══════►  │            │
│   Client   │       Channel            │   Server   │
│            │  ◄═══════════════════    │            │
└────────────┘                          └────────────┘
         Port 443                     Certificate
```

### HTTP vs HTTPS

| Feature | HTTP | HTTPS |
|---------|------|-------|
| Port | 80 | 443 |
| Encryption | None | TLS/SSL |
| URL | http:// | https:// |
| Security | None | Encrypted |
| Certificate | Not needed | Required |
| SEO | Lower ranking | Higher ranking |

### TLS Handshake (Simplified)

```
Client                                    Server
   │                                         │
   │──── Client Hello ──────────────────────►│
   │     (Supported ciphers, TLS version)    │
   │                                         │
   │◄─── Server Hello ───────────────────────│
   │     (Chosen cipher, certificate)        │
   │                                         │
   │──── Key Exchange ──────────────────────►│
   │     (Pre-master secret, encrypted)      │
   │                                         │
   │◄─── Finished ───────────────────────────│
   │                                         │
   │══════ Encrypted Communication ══════════│
```

### SSL/TLS Certificates

| Certificate Type | Validation | Use Case |
|------------------|------------|----------|
| DV (Domain) | Domain ownership | Basic websites |
| OV (Organization) | Organization verified | Business sites |
| EV (Extended) | Extensive verification | Banking, e-commerce |
| Wildcard | *.domain.com | Multiple subdomains |
| SAN | Multiple domains | Multiple domains |

### Certificate Authorities (CA)

- Let's Encrypt (free)
- DigiCert
- Comodo
- GeoTrust
- GlobalSign

## Testing HTTP/HTTPS

### curl

```bash
# Basic GET request
curl https://api.example.com

# Show headers
curl -I https://example.com
curl -i https://example.com    # Headers + body

# POST request
curl -X POST -d "name=John" https://api.example.com/users

# POST JSON
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"John"}' \
  https://api.example.com/users

# With authentication
curl -u username:password https://api.example.com
curl -H "Authorization: Bearer token123" https://api.example.com

# Follow redirects
curl -L https://example.com

# Verbose mode
curl -v https://example.com

# Check SSL certificate
curl -vI https://example.com 2>&1 | grep -A5 "Server certificate"

# Ignore SSL errors (testing only!)
curl -k https://self-signed.example.com
```

### wget

```bash
# Download file
wget https://example.com/file.zip

# Output to stdout
wget -qO- https://api.example.com
```

### OpenSSL (Certificate Check)

```bash
# Check certificate
openssl s_client -connect example.com:443

# Show certificate details
openssl s_client -connect example.com:443 | openssl x509 -text

# Check expiration
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

## HTTP in DevOps

### Health Checks

```bash
# Simple health check
curl -f http://localhost:8080/health || exit 1

# With timeout
curl --connect-timeout 5 -f http://localhost:8080/health
```

### Load Testing

```bash
# Apache Bench
ab -n 1000 -c 10 http://localhost:8080/

# wrk
wrk -t4 -c100 -d30s http://localhost:8080/
```

### SSL Certificate Monitoring

```bash
# Check expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -enddate

# Days until expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -checkend 2592000  # 30 days
```

## Quick Reference

| Method | Purpose | Idempotent |
|--------|---------|------------|
| GET | Read | Yes |
| POST | Create | No |
| PUT | Replace | Yes |
| PATCH | Update | No |
| DELETE | Remove | Yes |

| Status | Meaning |
|--------|---------|
| 2xx | Success |
| 3xx | Redirect |
| 4xx | Client Error |
| 5xx | Server Error |

| Port | Protocol |
|------|----------|
| 80 | HTTP |
| 443 | HTTPS |

---

**Previous:** [06-dhcp.md](06-dhcp.md) | **Next:** [08-firewalls-iptables.md](08-firewalls-iptables.md)
