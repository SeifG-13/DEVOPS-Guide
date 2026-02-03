# DNS (Domain Name System)

## What is DNS?

DNS translates human-readable domain names into IP addresses.

```
www.google.com  →  DNS  →  142.250.190.4
(Domain Name)           (IP Address)
```

Without DNS, you'd need to remember IP addresses for every website!

## DNS Hierarchy

```
                        ┌─────────┐
                        │  Root   │    (.)
                        │ Servers │
                        └────┬────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼────┐          ┌────▼────┐          ┌────▼────┐
   │  .com   │          │  .org   │          │  .net   │
   │   TLD   │          │   TLD   │          │   TLD   │
   └────┬────┘          └─────────┘          └─────────┘
        │
   ┌────▼────┐
   │ google  │  (Authoritative)
   │  .com   │
   └────┬────┘
        │
   ┌────▼────┐
   │  www    │  (Host record)
   └─────────┘
```

### DNS Components

| Component | Description |
|-----------|-------------|
| Root Servers | Top of hierarchy (13 worldwide) |
| TLD Servers | Top-Level Domain (.com, .org, .net) |
| Authoritative | Has actual DNS records for domain |
| Recursive Resolver | Queries other servers on behalf of client |
| Cache | Stores recent lookups |

## DNS Resolution Process

```
1. User types www.example.com

2. Browser checks cache
   └─► Found? Return IP
   └─► Not found? Continue

3. OS checks hosts file (/etc/hosts)
   └─► Found? Return IP
   └─► Not found? Continue

4. Query recursive resolver (ISP/8.8.8.8)
   └─► Found in cache? Return IP
   └─► Not found? Continue

5. Resolver queries Root server
   └─► Returns TLD server address (.com)

6. Resolver queries TLD server
   └─► Returns authoritative server address

7. Resolver queries Authoritative server
   └─► Returns IP address

8. Resolver caches and returns IP to client

9. Browser connects to IP address
```

### Diagram

```
Client              Resolver            Root           TLD        Authoritative
  │                    │                  │              │              │
  │─── www.example.com►│                  │              │              │
  │                    │─── . (root)? ───►│              │              │
  │                    │◄── .com TLD ─────│              │              │
  │                    │                  │              │              │
  │                    │─── .com? ────────────────────►  │              │
  │                    │◄── example.com NS ──────────────│              │
  │                    │                  │              │              │
  │                    │─── www.example.com? ─────────────────────────► │
  │                    │◄── 93.184.216.34 ──────────────────────────────│
  │◄── 93.184.216.34 ──│                  │              │              │
  │                    │                  │              │              │
```

## DNS Record Types

### Common Records

| Type | Name | Purpose | Example |
|------|------|---------|---------|
| A | Address | Maps domain to IPv4 | example.com → 93.184.216.34 |
| AAAA | IPv6 Address | Maps domain to IPv6 | example.com → 2606:2800:220:1:... |
| CNAME | Canonical Name | Alias to another domain | www → example.com |
| MX | Mail Exchange | Mail server for domain | Priority 10 mail.example.com |
| TXT | Text | Arbitrary text data | SPF, DKIM verification |
| NS | Name Server | Authoritative DNS servers | ns1.example.com |
| SOA | Start of Authority | Primary NS, zone info | Serial, refresh, retry |
| PTR | Pointer | Reverse DNS (IP to domain) | 34.216.184.93 → example.com |
| SRV | Service | Service location | _http._tcp.example.com |
| CAA | Cert Authority Auth | Allowed CAs for domain | letsencrypt.org |

### Record Examples

```bash
# A Record
example.com.    300    IN    A    93.184.216.34

# AAAA Record
example.com.    300    IN    AAAA    2606:2800:220:1:248:1893:25c8:1946

# CNAME Record
www.example.com.    300    IN    CNAME    example.com.

# MX Record (with priority)
example.com.    300    IN    MX    10 mail.example.com.
example.com.    300    IN    MX    20 backup-mail.example.com.

# TXT Record (SPF)
example.com.    300    IN    TXT    "v=spf1 include:_spf.google.com ~all"

# NS Record
example.com.    86400    IN    NS    ns1.example.com.
example.com.    86400    IN    NS    ns2.example.com.
```

## DNS Query Tools

### dig (Recommended)

```bash
# Basic query
dig example.com

# Query specific record type
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com ANY

# Short answer
dig +short example.com

# Use specific DNS server
dig @8.8.8.8 example.com

# Reverse lookup
dig -x 93.184.216.34

# Trace full resolution path
dig +trace example.com

# No recursion (query authoritative only)
dig +norecurse example.com

# Show all sections
dig +all example.com
```

### nslookup

```bash
# Basic lookup
nslookup example.com

# Query specific DNS server
nslookup example.com 8.8.8.8

# Query specific record type
nslookup -type=MX example.com
nslookup -type=NS example.com
nslookup -type=TXT example.com

# Reverse lookup
nslookup 93.184.216.34

# Interactive mode
nslookup
> server 8.8.8.8
> set type=MX
> example.com
> exit
```

### host

```bash
# Basic lookup
host example.com

# Specific record type
host -t MX example.com
host -t NS example.com
host -t TXT example.com

# Reverse lookup
host 93.184.216.34

# Use specific DNS server
host example.com 8.8.8.8

# Verbose output
host -v example.com
```

## DNS Configuration

### /etc/hosts (Local DNS)

```bash
# /etc/hosts
127.0.0.1       localhost
192.168.1.100   myserver.local myserver
10.0.0.50       database.internal
```

Checked before DNS servers.

### /etc/resolv.conf (DNS Servers)

```bash
# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
search example.com internal.example.com
options timeout:2 attempts:3
```

| Option | Description |
|--------|-------------|
| nameserver | DNS server IP |
| search | Domain search list |
| domain | Default domain |
| options | Various settings |

### systemd-resolved

```bash
# Check status
systemctl status systemd-resolved

# View current DNS settings
resolvectl status

# Flush DNS cache
resolvectl flush-caches

# Query with resolved
resolvectl query example.com
```

## Common DNS Servers

### Public DNS Servers

| Provider | Primary | Secondary |
|----------|---------|-----------|
| Google | 8.8.8.8 | 8.8.4.4 |
| Cloudflare | 1.1.1.1 | 1.0.0.1 |
| Quad9 | 9.9.9.9 | 149.112.112.112 |
| OpenDNS | 208.67.222.222 | 208.67.220.220 |

### DNS over HTTPS (DoH) / DNS over TLS (DoT)

```
Regular DNS:     UDP port 53 (unencrypted)
DNS over TLS:    TCP port 853 (encrypted)
DNS over HTTPS:  TCP port 443 (encrypted)
```

## TTL (Time To Live)

TTL specifies how long a DNS record should be cached.

```bash
example.com.    300    IN    A    93.184.216.34
                 │
                 └── TTL: 300 seconds (5 minutes)
```

| TTL | Use Case |
|-----|----------|
| 60-300 | Frequently changing |
| 3600 | Standard |
| 86400 | Stable records |

**Before migration:** Lower TTL in advance!

## DNS Troubleshooting

### Common Issues

| Issue | Check |
|-------|-------|
| No resolution | DNS server reachable? |
| Slow resolution | High TTL in cache? |
| Wrong IP | Propagation delay? |
| Intermittent | Multiple NS with different records? |

### Troubleshooting Commands

```bash
# Check DNS resolution
dig example.com
nslookup example.com

# Check specific DNS server
dig @8.8.8.8 example.com

# Check propagation
dig @ns1.example.com example.com
dig @ns2.example.com example.com

# Trace resolution path
dig +trace example.com

# Check reverse DNS
dig -x 93.184.216.34

# Flush local cache
# Ubuntu
sudo systemd-resolve --flush-caches
# macOS
sudo dscacheutil -flushcache
```

## Quick Reference

| Record | Purpose | Example |
|--------|---------|---------|
| A | IPv4 address | 93.184.216.34 |
| AAAA | IPv6 address | 2606:2800:... |
| CNAME | Alias | www → example.com |
| MX | Mail server | mail.example.com |
| TXT | Text/verification | SPF, DKIM |
| NS | Name servers | ns1.example.com |
| PTR | Reverse lookup | IP → domain |

| Tool | Usage |
|------|-------|
| `dig` | `dig example.com` |
| `nslookup` | `nslookup example.com` |
| `host` | `host example.com` |

---

**Previous:** [04-tcp-udp.md](04-tcp-udp.md) | **Next:** [06-dhcp.md](06-dhcp.md)
