# DHCP (Dynamic Host Configuration Protocol)

## What is DHCP?

DHCP automatically assigns IP addresses and network configuration to devices on a network.

```
┌──────────┐                              ┌──────────────┐
│  Client  │  ────  "I need an IP!"  ──►  │ DHCP Server  │
│ (No IP)  │  ◄───  "Here's 192.168.1.50" │              │
└──────────┘                              └──────────────┘
```

### What DHCP Provides

| Parameter | Description |
|-----------|-------------|
| IP Address | Device's network address |
| Subnet Mask | Network boundary |
| Default Gateway | Router address |
| DNS Servers | Name resolution servers |
| Lease Time | How long IP is valid |
| Domain Name | Network domain |
| NTP Servers | Time synchronization |

## DORA Process

DHCP uses a four-step process: **D**iscover, **O**ffer, **R**equest, **A**cknowledge

```
    Client                                  DHCP Server
       │                                         │
       │──────── DISCOVER (Broadcast) ──────────►│
       │         "Anyone have an IP for me?"     │
       │                                         │
       │◄─────── OFFER (Unicast/Broadcast) ──────│
       │         "How about 192.168.1.50?"       │
       │                                         │
       │──────── REQUEST (Broadcast) ───────────►│
       │         "I'll take 192.168.1.50"        │
       │                                         │
       │◄─────── ACK (Unicast) ──────────────────│
       │         "It's yours for 24 hours"       │
       │                                         │
```

### Step-by-Step

| Step | Message | Source | Destination | Port |
|------|---------|--------|-------------|------|
| 1 | DISCOVER | 0.0.0.0:68 | 255.255.255.255:67 | UDP |
| 2 | OFFER | Server:67 | 255.255.255.255:68 | UDP |
| 3 | REQUEST | 0.0.0.0:68 | 255.255.255.255:67 | UDP |
| 4 | ACK | Server:67 | Client:68 | UDP |

### DHCP Ports

| Port | Protocol | Direction |
|------|----------|-----------|
| 67 | UDP | Server (listens) |
| 68 | UDP | Client (listens) |

## DHCP Lease Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    Lease Timeline                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ├────────────────┼────────────────┼────────────────┤       │
│  0%              50%              87.5%           100%       │
│  │                │                │                │        │
│  Lease          T1               T2             Lease        │
│  Start        Renewal         Rebind          Expires        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Lease Timers

| Timer | Default | Action |
|-------|---------|--------|
| Lease Time | 24 hours | Total lease duration |
| T1 (Renewal) | 50% | Client tries to renew with same server |
| T2 (Rebind) | 87.5% | Client broadcasts for any server |
| Expiration | 100% | Client must release IP |

## DHCP Message Types

| Type | Code | Purpose |
|------|------|---------|
| DISCOVER | 1 | Client looking for server |
| OFFER | 2 | Server offering IP |
| REQUEST | 3 | Client requesting IP |
| DECLINE | 4 | Client rejecting IP |
| ACK | 5 | Server confirming |
| NAK | 6 | Server denying |
| RELEASE | 7 | Client releasing IP |
| INFORM | 8 | Client requesting config only |

## Static vs Dynamic IP

| Feature | Static IP | Dynamic IP (DHCP) |
|---------|-----------|-------------------|
| Configuration | Manual | Automatic |
| IP Changes | Never | May change |
| Best For | Servers, printers | Workstations, mobile |
| Management | Time-consuming | Easy |
| Conflicts | Possible if duplicated | Prevented by DHCP |

### DHCP Reservation

Best of both worlds: DHCP-assigned but always the same IP.

```
MAC Address: AA:BB:CC:DD:EE:FF
Reserved IP: 192.168.1.100
```

## DHCP Server Configuration

### ISC DHCP Server (Linux)

```bash
# Install
sudo apt install isc-dhcp-server

# Configuration file
sudo vim /etc/dhcp/dhcpd.conf
```

```bash
# /etc/dhcp/dhcpd.conf

# Global settings
option domain-name "example.com";
option domain-name-servers 8.8.8.8, 8.8.4.4;
default-lease-time 86400;     # 24 hours
max-lease-time 604800;        # 7 days

# Subnet definition
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option broadcast-address 192.168.1.255;
}

# Static reservation
host server1 {
    hardware ethernet AA:BB:CC:DD:EE:FF;
    fixed-address 192.168.1.50;
}
```

```bash
# Specify interface
sudo vim /etc/default/isc-dhcp-server
INTERFACESv4="eth0"

# Start service
sudo systemctl enable --now isc-dhcp-server

# View leases
cat /var/lib/dhcp/dhcpd.leases
```

### dnsmasq (Lightweight Alternative)

```bash
# Install
sudo apt install dnsmasq

# Configuration
sudo vim /etc/dnsmasq.conf
```

```bash
# /etc/dnsmasq.conf
interface=eth0
dhcp-range=192.168.1.100,192.168.1.200,24h
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,8.8.8.8,8.8.4.4

# Static reservation
dhcp-host=AA:BB:CC:DD:EE:FF,192.168.1.50
```

## DHCP Client Configuration

### Linux Client

```bash
# Release current lease
sudo dhclient -r eth0

# Request new lease
sudo dhclient eth0

# Verbose mode
sudo dhclient -v eth0

# View current lease
cat /var/lib/dhcp/dhclient.leases
```

### Netplan (Ubuntu)

```yaml
# /etc/netplan/01-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false
```

### NetworkManager

```bash
# Set to DHCP
nmcli con mod "Connection" ipv4.method auto
nmcli con up "Connection"
```

## DHCP Relay Agent

When DHCP server is on a different subnet.

```
┌──────────┐      ┌─────────────┐      ┌──────────────┐
│  Client  │ ──►  │ DHCP Relay  │ ──►  │ DHCP Server  │
│  Subnet A│      │  (Router)   │      │  Subnet B    │
└──────────┘      └─────────────┘      └──────────────┘
```

```bash
# Configure relay (on router/gateway)
sudo apt install isc-dhcp-relay

# /etc/default/isc-dhcp-relay
SERVERS="192.168.2.10"      # DHCP server IP
INTERFACES="eth0 eth1"       # Interfaces to relay
```

## Troubleshooting DHCP

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No IP received | Server down | Check server status |
| 169.254.x.x (APIPA) | No DHCP response | Check network connectivity |
| IP conflict | Duplicate assignment | Check reservations/static IPs |
| Wrong subnet | Wrong DHCP scope | Verify subnet configuration |

### Troubleshooting Commands

```bash
# Check DHCP client status
systemctl status dhclient

# View current IP
ip addr show

# Check for DHCP traffic (tcpdump)
sudo tcpdump -i eth0 port 67 or port 68

# Force DHCP renewal
sudo dhclient -r && sudo dhclient

# View DHCP leases (server)
cat /var/lib/dhcp/dhcpd.leases

# View DHCP lease (client)
cat /var/lib/dhcp/dhclient.leases

# Check DHCP server logs
journalctl -u isc-dhcp-server
```

### Capture DHCP Traffic

```bash
# Using tcpdump
sudo tcpdump -i eth0 -n port 67 or port 68 -v

# Using Wireshark filter
bootp or dhcp
```

## DHCP Security

### Threats

| Threat | Description |
|--------|-------------|
| Rogue DHCP | Unauthorized server giving bad config |
| DHCP Starvation | Attacker requests all IPs |
| DHCP Spoofing | Attacker responds faster than real server |

### Mitigation

- **DHCP Snooping**: Switch feature to filter untrusted DHCP
- **Port Security**: Limit MAC addresses per port
- **802.1X**: Network access control

## Quick Reference

| Term | Description |
|------|-------------|
| DORA | Discover, Offer, Request, Acknowledge |
| Lease | Time period IP is valid |
| Reservation | Fixed IP for specific MAC |
| Relay | Forwards DHCP across subnets |
| Pool/Scope | Range of IPs to assign |

| Command | Purpose |
|---------|---------|
| `dhclient -r` | Release IP |
| `dhclient eth0` | Request IP |
| `ip addr show` | View current IP |

---

**Previous:** [05-dns.md](05-dns.md) | **Next:** [07-http-https.md](07-http-https.md)
