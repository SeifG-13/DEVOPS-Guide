# Networking Cheat Sheet for DevOps Engineers

## Quick Reference - OSI Model

| Layer | Name | Protocols/Concepts | Devices |
|-------|------|-------------------|---------|
| 7 | Application | HTTP, HTTPS, DNS, FTP, SSH | - |
| 6 | Presentation | SSL/TLS, Encryption | - |
| 5 | Session | Sessions, Sockets | - |
| 4 | Transport | TCP, UDP | Load Balancers |
| 3 | Network | IP, ICMP, Routing | Routers |
| 2 | Data Link | Ethernet, MAC, ARP | Switches |
| 1 | Physical | Cables, Signals | Hubs |

---

## Essential Networking Commands

### IP Configuration
```bash
ip addr                         # Show IP addresses
ip addr add 192.168.1.10/24 dev eth0   # Add IP
ip link set eth0 up             # Enable interface
ip route                        # Show routing table
ip route add default via 192.168.1.1   # Add default gateway
```

### DNS
```bash
dig example.com                 # DNS lookup
dig +short example.com          # Short answer
dig @8.8.8.8 example.com        # Query specific DNS
nslookup example.com            # Alternative DNS lookup
host example.com                # Simple DNS lookup
cat /etc/resolv.conf            # DNS configuration
```

### Connectivity Testing
```bash
ping -c 4 google.com            # Test connectivity
traceroute google.com           # Trace packet path
mtr google.com                  # Combined ping+traceroute
telnet host 80                  # Test TCP port
nc -zv host 80                  # Netcat port check
curl -v http://example.com      # HTTP request verbose
```

### Port & Connection Analysis
```bash
ss -tulnp                       # All listening ports
ss -s                           # Socket statistics
netstat -tulnp                  # Legacy port listing
lsof -i :80                     # Process using port 80
nmap -sT host                   # Port scan
```

### Firewall (iptables/nftables)
```bash
# iptables
iptables -L -n -v               # List rules
iptables -A INPUT -p tcp --dport 80 -j ACCEPT   # Allow port 80
iptables -A INPUT -s 10.0.0.0/8 -j DROP         # Block subnet
iptables-save > /etc/iptables/rules.v4          # Save rules

# UFW (Ubuntu)
ufw status                      # Show status
ufw allow 22                    # Allow SSH
ufw allow from 10.0.0.0/8       # Allow subnet
ufw enable                      # Enable firewall
```

### Packet Capture
```bash
tcpdump -i eth0                 # Capture on interface
tcpdump -i eth0 port 80         # Capture port 80
tcpdump -i eth0 host 10.0.0.1   # Capture specific host
tcpdump -w capture.pcap         # Write to file
tshark -i eth0 -f "port 443"    # Wireshark CLI
```

### Network Configuration Files
```bash
/etc/hosts                      # Local DNS
/etc/resolv.conf                # DNS servers
/etc/network/interfaces         # Debian network config
/etc/netplan/*.yaml             # Ubuntu netplan
/etc/sysconfig/network-scripts/ # RHEL network scripts
```

---

## Key Protocols & Ports

| Port | Protocol | Service |
|------|----------|---------|
| 20, 21 | TCP | FTP |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 27017 | TCP | MongoDB |

---

## Subnetting Quick Reference

### CIDR Notation
| CIDR | Subnet Mask | Usable Hosts |
|------|-------------|--------------|
| /32 | 255.255.255.255 | 1 |
| /30 | 255.255.255.252 | 2 |
| /24 | 255.255.255.0 | 254 |
| /16 | 255.255.0.0 | 65,534 |
| /8 | 255.0.0.0 | 16,777,214 |

### Private IP Ranges
| Class | Range | CIDR |
|-------|-------|------|
| A | 10.0.0.0 - 10.255.255.255 | 10.0.0.0/8 |
| B | 172.16.0.0 - 172.31.255.255 | 172.16.0.0/12 |
| C | 192.168.0.0 - 192.168.255.255 | 192.168.0.0/16 |

---

## Interview Q&A

### Q1: What is the difference between TCP and UDP?
**A:**
- **TCP**: Connection-oriented, reliable, ordered delivery, flow control, slower (HTTP, SSH, FTP)
- **UDP**: Connectionless, unreliable, no ordering, faster (DNS, DHCP, streaming)

### Q2: Explain the TCP 3-way handshake
**A:**
1. **SYN**: Client sends synchronize request
2. **SYN-ACK**: Server acknowledges and sends its own SYN
3. **ACK**: Client acknowledges, connection established

### Q3: What is NAT and why is it used?
**A:** Network Address Translation translates private IPs to public IPs. Used for:
- IPv4 address conservation
- Security (hides internal network)
- Types: SNAT (source), DNAT (destination), PAT (port-based)

### Q4: What is the difference between a router and a switch?
**A:**
- **Switch**: Layer 2, forwards based on MAC address, creates VLANs
- **Router**: Layer 3, forwards based on IP address, connects different networks

### Q5: How does DNS work?
**A:**
1. Client queries recursive resolver
2. Resolver queries root nameserver
3. Root refers to TLD nameserver (.com)
4. TLD refers to authoritative nameserver
5. Authoritative returns IP address
6. Resolver caches and returns to client

### Q6: What is a VLAN?
**A:** Virtual LAN - logically segments a physical network. Benefits:
- Security isolation
- Broadcast domain reduction
- Simplified management
- Trunk ports carry multiple VLANs (802.1Q tagging)

### Q7: Explain load balancing algorithms
**A:**
- **Round Robin**: Sequential distribution
- **Least Connections**: Route to least busy server
- **IP Hash**: Consistent routing based on client IP
- **Weighted**: Based on server capacity
- **Health-based**: Only to healthy servers

### Q8: What is the difference between L4 and L7 load balancing?
**A:**
- **L4**: Based on IP/port, faster, TCP/UDP level
- **L7**: Based on content (URL, headers), smarter routing, HTTP level

### Q9: How do you troubleshoot network connectivity issues?
**A:**
1. Check physical/link: `ip link`
2. Check IP config: `ip addr`
3. Check local connectivity: `ping gateway`
4. Check DNS: `dig domain`
5. Check remote: `ping/traceroute remote`
6. Check ports: `ss -tulnp`, `telnet host port`
7. Check firewall: `iptables -L`

### Q10: What is a CDN?
**A:** Content Delivery Network - distributed servers that cache content closer to users. Benefits:
- Reduced latency
- Offload origin server
- DDoS protection
- Global availability

---

## Network Troubleshooting Flowchart

```
Can't reach remote host?
│
├─ Check local interface: ip addr
│   └─ No IP? → Check DHCP or static config
│
├─ Ping gateway: ping 192.168.1.1
│   └─ Fails? → Check cables/switch/VLAN
│
├─ Ping external IP: ping 8.8.8.8
│   └─ Fails? → Check routing: ip route
│
├─ Ping domain: ping google.com
│   └─ Fails? → DNS issue: check /etc/resolv.conf
│
└─ Port unreachable?
    └─ Check firewall: iptables -L, ss -tulnp
```

---

## Best Practices

1. **Segment networks** - Use VLANs and subnets for isolation
2. **Principle of least privilege** - Only open required ports
3. **Use private IPs** - NAT for internet access
4. **Implement redundancy** - Multiple paths, load balancers
5. **Monitor network** - Prometheus, Grafana, SNMP
6. **Document topology** - Keep network diagrams updated
7. **Encrypt traffic** - TLS everywhere, VPNs for sensitive data
8. **Regular security audits** - Port scans, vulnerability assessments
