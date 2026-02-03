# Firewalls & iptables

## What is a Firewall?

A firewall controls incoming and outgoing network traffic based on security rules.

```
    Internet                    Firewall                    Internal Network
        │                          │                              │
  ──────┼──────────────────────────┼──────────────────────────────┼──────
        │    ┌─────────────────┐   │                              │
Allowed ├───►│ Rule: Allow 443 │───┼──────────────────────────────►
        │    └─────────────────┘   │                              │
        │    ┌─────────────────┐   │                              │
Blocked ├──X─│ Rule: Block 23  │   │                              │
        │    └─────────────────┘   │                              │
```

## Firewall Types

| Type | Layer | Description |
|------|-------|-------------|
| Packet Filter | 3-4 | Filters by IP, port, protocol |
| Stateful | 3-4 | Tracks connection state |
| Application | 7 | Inspects application data |
| Next-Gen (NGFW) | 3-7 | Deep packet inspection, IDS/IPS |

## Linux Firewall Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Space                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   iptables  │  │     ufw     │  │  firewalld  │          │
│  │  (command)  │  │  (Ubuntu)   │  │   (RHEL)    │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                │                 │                 │
├─────────┼────────────────┼─────────────────┼─────────────────┤
│         └────────────────┼─────────────────┘                 │
│                          │                                   │
│                    ┌─────▼─────┐                             │
│                    │ netfilter │     Kernel Space            │
│                    │ (kernel)  │                             │
│                    └───────────┘                             │
└─────────────────────────────────────────────────────────────┘
```

## iptables

### Tables and Chains

```
┌─────────────────────────────────────────────────────────────┐
│                      iptables Tables                         │
├─────────────────────────────────────────────────────────────┤
│  filter (default)  │  nat             │  mangle             │
│  ├── INPUT         │  ├── PREROUTING  │  ├── PREROUTING     │
│  ├── FORWARD       │  ├── POSTROUTING │  ├── INPUT          │
│  └── OUTPUT        │  └── OUTPUT      │  ├── FORWARD        │
│                    │                   │  ├── OUTPUT         │
│                    │                   │  └── POSTROUTING    │
└─────────────────────────────────────────────────────────────┘
```

### Tables

| Table | Purpose |
|-------|---------|
| filter | Packet filtering (default) |
| nat | Network Address Translation |
| mangle | Packet modification |
| raw | Connection tracking exemption |

### Chains

| Chain | Table | Purpose |
|-------|-------|---------|
| INPUT | filter | Incoming packets to local |
| OUTPUT | filter | Outgoing packets from local |
| FORWARD | filter | Packets passing through |
| PREROUTING | nat | Before routing decision |
| POSTROUTING | nat | After routing decision |

### Packet Flow

```
                    ┌──────────────┐
                    │   INCOMING   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  PREROUTING  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
              ┌─────┤   Routing    ├─────┐
              │     │   Decision   │     │
              │     └──────────────┘     │
        For local                  For forwarding
              │                          │
       ┌──────▼───────┐          ┌──────▼───────┐
       │    INPUT     │          │   FORWARD    │
       └──────┬───────┘          └──────┬───────┘
              │                          │
       ┌──────▼───────┐                  │
       │    Local     │                  │
       │   Process    │                  │
       └──────┬───────┘                  │
              │                          │
       ┌──────▼───────┐                  │
       │    OUTPUT    │                  │
       └──────┬───────┘                  │
              │                          │
              └──────────┬───────────────┘
                         │
                  ┌──────▼───────┐
                  │ POSTROUTING  │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │   OUTGOING   │
                  └──────────────┘
```

### iptables Syntax

```bash
iptables [-t table] COMMAND chain [match] [-j target]
```

### Commands

| Command | Description |
|---------|-------------|
| -A | Append rule to chain |
| -I | Insert rule at position |
| -D | Delete rule |
| -R | Replace rule |
| -L | List rules |
| -F | Flush (delete all rules) |
| -P | Set default policy |
| -N | Create new chain |
| -X | Delete chain |

### Match Options

| Option | Description | Example |
|--------|-------------|---------|
| -p | Protocol | -p tcp |
| -s | Source IP | -s 192.168.1.0/24 |
| -d | Destination IP | -d 10.0.0.1 |
| --sport | Source port | --sport 80 |
| --dport | Destination port | --dport 443 |
| -i | Input interface | -i eth0 |
| -o | Output interface | -o eth1 |
| -m | Match extension | -m state |

### Targets

| Target | Description |
|--------|-------------|
| ACCEPT | Allow packet |
| DROP | Silently discard |
| REJECT | Discard with error |
| LOG | Log packet |
| SNAT | Source NAT |
| DNAT | Destination NAT |
| MASQUERADE | Dynamic SNAT |

### Common iptables Commands

```bash
# List all rules
sudo iptables -L -n -v
sudo iptables -L -n --line-numbers

# List specific table
sudo iptables -t nat -L -n -v

# Set default policy
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow established connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow from specific IP
sudo iptables -A INPUT -s 192.168.1.100 -j ACCEPT

# Allow subnet
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# Block IP
sudo iptables -A INPUT -s 10.0.0.5 -j DROP

# Allow ICMP (ping)
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Allow localhost
sudo iptables -A INPUT -i lo -j ACCEPT

# Log dropped packets
sudo iptables -A INPUT -j LOG --log-prefix "DROPPED: "

# Delete rule by number
sudo iptables -D INPUT 3

# Flush all rules
sudo iptables -F

# Save rules
sudo iptables-save > /etc/iptables/rules.v4

# Restore rules
sudo iptables-restore < /etc/iptables/rules.v4
```

### Complete Firewall Example

```bash
#!/bin/bash

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F

# Set default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (port 22)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP (port 80)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow HTTPS (port 443)
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow ping
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Log and drop everything else
iptables -A INPUT -j LOG --log-prefix "DROPPED: "
iptables -A INPUT -j DROP
```

## UFW (Uncomplicated Firewall)

Ubuntu's simplified firewall interface.

```bash
# Enable/Disable
sudo ufw enable
sudo ufw disable

# Check status
sudo ufw status
sudo ufw status verbose

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow/Deny port
sudo ufw allow 22
sudo ufw allow 80/tcp
sudo ufw deny 23

# Allow service by name
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Allow from IP
sudo ufw allow from 192.168.1.100

# Allow from subnet
sudo ufw allow from 192.168.1.0/24

# Allow to specific port from IP
sudo ufw allow from 192.168.1.100 to any port 22

# Delete rule
sudo ufw delete allow 80

# Reset to defaults
sudo ufw reset

# Application profiles
sudo ufw app list
sudo ufw allow 'Nginx Full'
```

## firewalld (RHEL/CentOS)

```bash
# Start/Enable
sudo systemctl start firewalld
sudo systemctl enable firewalld

# Check status
sudo firewall-cmd --state
sudo firewall-cmd --list-all

# Get active zones
sudo firewall-cmd --get-active-zones

# Add service
sudo firewall-cmd --add-service=http
sudo firewall-cmd --add-service=http --permanent

# Add port
sudo firewall-cmd --add-port=8080/tcp
sudo firewall-cmd --add-port=8080/tcp --permanent

# Remove service/port
sudo firewall-cmd --remove-service=http
sudo firewall-cmd --remove-port=8080/tcp

# Reload
sudo firewall-cmd --reload

# Allow IP
sudo firewall-cmd --add-source=192.168.1.100
```

### firewalld Zones

| Zone | Description |
|------|-------------|
| drop | Drop all incoming |
| block | Reject incoming |
| public | Public networks (default) |
| external | External networks with NAT |
| internal | Internal networks |
| dmz | DMZ servers |
| work | Work networks |
| home | Home networks |
| trusted | Accept all |

## NAT with iptables

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Or permanently in /etc/sysctl.conf
net.ipv4.ip_forward = 1

# SNAT (Source NAT) - Masquerade
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# DNAT (Port Forwarding)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:8080
iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 8080 -j ACCEPT
```

## Quick Reference

### iptables

| Command | Purpose |
|---------|---------|
| `iptables -L -n` | List rules |
| `iptables -A INPUT ...` | Add input rule |
| `iptables -D INPUT 3` | Delete rule 3 |
| `iptables -F` | Flush all rules |
| `iptables -P INPUT DROP` | Set default policy |
| `iptables-save` | Save rules |
| `iptables-restore` | Restore rules |

### UFW

| Command | Purpose |
|---------|---------|
| `ufw enable` | Enable firewall |
| `ufw status` | Show status |
| `ufw allow 22` | Allow port |
| `ufw deny 23` | Deny port |
| `ufw allow from IP` | Allow IP |
| `ufw delete allow 80` | Delete rule |

---

**Previous:** [07-http-https.md](07-http-https.md) | **Next:** [09-nat-port-forwarding.md](09-nat-port-forwarding.md)
