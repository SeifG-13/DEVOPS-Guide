# NAT & Port Forwarding

## What is NAT?

NAT (Network Address Translation) modifies IP addresses in packet headers as they pass through a router/firewall.

```
Private Network                    Router/NAT                    Internet
192.168.1.0/24

┌────────────┐                  ┌──────────────┐               ┌──────────┐
│ 192.168.1.10 │────────────────►│              │               │          │
└────────────┘                  │   Public IP   │───────────────►│  Server  │
┌────────────┐                  │  203.0.113.1  │               │          │
│ 192.168.1.20 │────────────────►│              │               └──────────┘
└────────────┘                  └──────────────┘
```

### Why NAT?

- **IPv4 exhaustion**: Limited public IPs
- **Security**: Hides internal network structure
- **Flexibility**: Private IPs can be reused

## NAT Types

### 1. SNAT (Source NAT)

Changes the **source IP** of outgoing packets.

```
Internal → Router → Internet

Packet before NAT:
  Src: 192.168.1.10  Dst: 8.8.8.8

Packet after SNAT:
  Src: 203.0.113.1   Dst: 8.8.8.8
       └── Public IP
```

**Use case**: Allow internal hosts to access internet.

### 2. DNAT (Destination NAT)

Changes the **destination IP** of incoming packets.

```
Internet → Router → Internal

Packet before NAT:
  Src: 8.8.8.8  Dst: 203.0.113.1

Packet after DNAT:
  Src: 8.8.8.8  Dst: 192.168.1.100
                     └── Internal server
```

**Use case**: Port forwarding, load balancing.

### 3. PAT / NAPT (Port Address Translation)

Maps multiple private IPs to one public IP using different ports.

```
┌────────────────────────────────────────────────────────────────┐
│                     PAT Translation Table                       │
├────────────────────────┬───────────────────────────────────────┤
│  Internal              │  External                              │
├────────────────────────┼───────────────────────────────────────┤
│  192.168.1.10:52000    │  203.0.113.1:10001                    │
│  192.168.1.20:52001    │  203.0.113.1:10002                    │
│  192.168.1.10:52002    │  203.0.113.1:10003                    │
└────────────────────────┴───────────────────────────────────────┘
```

### 4. Masquerade

Dynamic SNAT - uses whatever IP is on the outgoing interface. Useful when public IP is assigned via DHCP.

## NAT with iptables

### Enable IP Forwarding

```bash
# Temporary
echo 1 > /proc/sys/net/ipv4/ip_forward

# Permanent
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

### SNAT (Static Public IP)

```bash
# Outgoing traffic from 192.168.1.0/24 gets source IP 203.0.113.1
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j SNAT --to-source 203.0.113.1
```

### Masquerade (Dynamic Public IP)

```bash
# Outgoing traffic uses whatever IP is on eth0
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

### DNAT (Port Forwarding)

```bash
# Forward port 80 to internal server
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80

# Also need to allow forwarding
iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 80 -j ACCEPT
```

## Port Forwarding

### What is Port Forwarding?

Redirects traffic from one port on the router to a specific internal host/port.

```
Internet                        Router                       Internal
                               203.0.113.1
  User ──── port 80 ──────────►│       │──── port 8080 ──────► 192.168.1.100
                               │       │
  User ──── port 443 ─────────►│       │──── port 443 ───────► 192.168.1.101
                               │       │
  User ──── port 22 ──────────►│       │──── port 22 ────────► 192.168.1.102
```

### Port Forwarding with iptables

```bash
# Forward external:80 to internal:80
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80
iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 80 -j ACCEPT

# Forward external:2222 to internal:22 (SSH on different port)
iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 192.168.1.100:22
iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 22 -j ACCEPT

# Forward range of ports
iptables -t nat -A PREROUTING -p tcp --dport 3000:3100 -j DNAT --to-destination 192.168.1.100

# Forward UDP
iptables -t nat -A PREROUTING -p udp --dport 53 -j DNAT --to-destination 192.168.1.100:53
```

### Complete NAT Router Example

```bash
#!/bin/bash

# Variables
WAN_IF="eth0"
LAN_IF="eth1"
LAN_NET="192.168.1.0/24"
WEB_SERVER="192.168.1.100"

# Enable forwarding
sysctl -w net.ipv4.ip_forward=1

# Flush existing rules
iptables -F
iptables -t nat -F

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow LAN to WAN
iptables -A FORWARD -i $LAN_IF -o $WAN_IF -j ACCEPT

# Masquerade outgoing traffic
iptables -t nat -A POSTROUTING -s $LAN_NET -o $WAN_IF -j MASQUERADE

# Port forward HTTP to web server
iptables -t nat -A PREROUTING -i $WAN_IF -p tcp --dport 80 -j DNAT --to-destination $WEB_SERVER:80
iptables -A FORWARD -p tcp -d $WEB_SERVER --dport 80 -j ACCEPT

# Port forward HTTPS
iptables -t nat -A PREROUTING -i $WAN_IF -p tcp --dport 443 -j DNAT --to-destination $WEB_SERVER:443
iptables -A FORWARD -p tcp -d $WEB_SERVER --dport 443 -j ACCEPT
```

## Viewing NAT Rules

```bash
# List NAT table rules
iptables -t nat -L -n -v

# List with line numbers
iptables -t nat -L -n --line-numbers

# Show PREROUTING chain
iptables -t nat -L PREROUTING -n -v

# Show POSTROUTING chain
iptables -t nat -L POSTROUTING -n -v

# Show connection tracking
cat /proc/net/nf_conntrack
conntrack -L
```

## NAT with firewalld (RHEL/CentOS)

### Enable Masquerading

```bash
# Enable masquerade on zone
firewall-cmd --zone=public --add-masquerade --permanent

# Check masquerade status
firewall-cmd --zone=public --query-masquerade
```

### Port Forwarding

```bash
# Forward port 80 to internal server
firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toaddr=192.168.1.100 --permanent

# Forward with port change
firewall-cmd --zone=public --add-forward-port=port=2222:proto=tcp:toaddr=192.168.1.100:toport=22 --permanent

# Reload
firewall-cmd --reload

# List forward ports
firewall-cmd --zone=public --list-forward-ports
```

## NAT with UFW

```bash
# Enable IP forwarding in /etc/ufw/sysctl.conf
net.ipv4.ip_forward=1

# Add NAT rules in /etc/ufw/before.rules (before *filter)
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
COMMIT

# Port forward (add to before.rules)
*nat
:PREROUTING ACCEPT [0:0]
-A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80
COMMIT

# Restart UFW
ufw disable && ufw enable
```

## NAT Types (Consumer/Gaming Context)

| Type | Description | Compatibility |
|------|-------------|---------------|
| Full Cone | Any external host can send to mapped port | Open |
| Restricted Cone | Only hosts client contacted can reply | Moderate |
| Port Restricted | Host + port must match | Moderate |
| Symmetric | Different mapping per destination | Strict |

## Troubleshooting NAT

### Common Issues

| Issue | Check |
|-------|-------|
| No internet from LAN | IP forwarding enabled? |
| Port forward not working | FORWARD chain allows traffic? |
| Intermittent connectivity | Connection tracking table full? |
| Hairpin NAT | Need special rules for internal access |

### Debugging Commands

```bash
# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward

# Check NAT rules
iptables -t nat -L -n -v

# Check FORWARD rules
iptables -L FORWARD -n -v

# Monitor NAT connections
watch -n 1 'conntrack -L | head -20'

# Check for dropped packets
iptables -L -n -v | grep DROP

# Trace packet path
iptables -t raw -A PREROUTING -p tcp --dport 80 -j TRACE
dmesg | tail
```

## Quick Reference

| NAT Type | Direction | Use Case |
|----------|-----------|----------|
| SNAT | Outbound | Fixed public IP |
| Masquerade | Outbound | Dynamic public IP |
| DNAT | Inbound | Port forwarding |

| Command | Purpose |
|---------|---------|
| `iptables -t nat -L -n -v` | List NAT rules |
| `-j SNAT --to-source IP` | Source NAT |
| `-j MASQUERADE` | Dynamic SNAT |
| `-j DNAT --to-destination IP:port` | Destination NAT |

---

**Previous:** [08-firewalls-iptables.md](08-firewalls-iptables.md) | **Next:** [10-load-balancing.md](10-load-balancing.md)
