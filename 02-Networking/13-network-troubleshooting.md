# Network Troubleshooting

## Troubleshooting Methodology

### OSI Layer Approach (Bottom-Up)

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Physical                                              │
│  □ Cable connected?  □ Link light on?  □ Right port?            │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Data Link                                             │
│  □ MAC address?  □ Switch port?  □ VLAN correct?                │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Network                                               │
│  □ IP address?  □ Subnet mask?  □ Gateway?  □ Routing?          │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: Transport                                             │
│  □ Port open?  □ Firewall?  □ Service listening?                │
├─────────────────────────────────────────────────────────────────┤
│  Layer 7: Application                                           │
│  □ Service running?  □ Config correct?  □ Logs?                 │
└─────────────────────────────────────────────────────────────────┘
```

### Quick Checklist

1. **Can you ping localhost?** → TCP/IP stack working
2. **Can you ping gateway?** → Local network working
3. **Can you ping 8.8.8.8?** → Internet connectivity
4. **Can you ping google.com?** → DNS working

## Diagnostic Commands

### Layer 1-2: Physical/Data Link

```bash
# Check interface status
ip link show
ethtool eth0

# View interface errors
ip -s link show eth0
ifconfig eth0

# Check ARP table (MAC addresses)
arp -a
ip neigh show

# Check for cable/link
ethtool eth0 | grep "Link detected"
```

### Layer 3: Network

```bash
# Check IP configuration
ip addr show
ifconfig

# Check routing table
ip route show
route -n
netstat -rn

# Verify gateway
ip route | grep default

# Ping tests
ping -c 4 127.0.0.1        # Loopback
ping -c 4 <gateway_ip>     # Gateway
ping -c 4 8.8.8.8          # Internet
ping -c 4 google.com       # DNS + Internet

# Trace route
traceroute google.com
traceroute -n google.com   # No DNS resolution
mtr google.com             # Continuous trace

# Path MTU discovery
ping -M do -s 1472 <destination>
```

### Layer 4: Transport

```bash
# Check listening ports
ss -tulpn
netstat -tulpn

# Check if port is open
nc -zv <host> <port>
telnet <host> <port>

# Check established connections
ss -t state established
netstat -ant

# Check for firewall rules
iptables -L -n -v
ufw status verbose
firewall-cmd --list-all
```

### DNS Troubleshooting

```bash
# Check DNS configuration
cat /etc/resolv.conf

# Test DNS resolution
nslookup google.com
dig google.com
host google.com

# Query specific DNS server
dig @8.8.8.8 google.com
nslookup google.com 8.8.8.8

# Trace DNS resolution
dig +trace google.com

# Reverse lookup
dig -x 8.8.8.8

# Check /etc/hosts
cat /etc/hosts

# Flush DNS cache
sudo systemd-resolve --flush-caches
resolvectl flush-caches
```

### Layer 7: Application

```bash
# Check if service is running
systemctl status nginx
systemctl status sshd

# Check service logs
journalctl -u nginx -f
tail -f /var/log/nginx/error.log

# Test HTTP
curl -v http://example.com
curl -I http://example.com
wget -O- http://example.com

# Test specific port
curl telnet://hostname:port
```

## Network Analysis Tools

### tcpdump

```bash
# Capture all traffic on interface
sudo tcpdump -i eth0

# Capture specific port
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 port 443

# Capture specific host
sudo tcpdump -i eth0 host 192.168.1.100

# Capture and save to file
sudo tcpdump -i eth0 -w capture.pcap

# Read capture file
tcpdump -r capture.pcap

# Show packet contents
sudo tcpdump -i eth0 -X port 80

# Common filters
sudo tcpdump -i eth0 'tcp port 80'
sudo tcpdump -i eth0 'src host 192.168.1.100'
sudo tcpdump -i eth0 'dst port 443'
sudo tcpdump -i eth0 'icmp'
sudo tcpdump -i eth0 'not port 22'   # Exclude SSH
```

### Wireshark (GUI)

```bash
# Install
sudo apt install wireshark

# Capture with tshark (CLI)
tshark -i eth0
tshark -i eth0 -f "port 80"
tshark -i eth0 -w capture.pcap

# Common display filters
http
tcp.port == 80
ip.addr == 192.168.1.100
dns
tcp.flags.syn == 1 && tcp.flags.ack == 0   # SYN packets
```

### nmap

```bash
# Scan single host
nmap 192.168.1.1

# Scan subnet
nmap 192.168.1.0/24

# Scan specific ports
nmap -p 80,443 192.168.1.1

# Scan port range
nmap -p 1-1000 192.168.1.1

# Quick scan (top 100 ports)
nmap -F 192.168.1.1

# Detect OS and services
nmap -A 192.168.1.1

# UDP scan
nmap -sU 192.168.1.1

# Check if host is up (no port scan)
nmap -sn 192.168.1.0/24
```

### netcat (nc)

```bash
# Check if port is open
nc -zv hostname 80

# Listen on port
nc -l -p 8080

# Send data
echo "test" | nc hostname 80

# Chat/file transfer
# Server: nc -l -p 1234 > received_file
# Client: nc hostname 1234 < file_to_send

# Port scanning
nc -zv hostname 1-100
```

## Common Issues and Solutions

### No Network Connectivity

| Check | Command | Fix |
|-------|---------|-----|
| Interface up? | `ip link show` | `ip link set eth0 up` |
| IP assigned? | `ip addr show` | `dhclient eth0` |
| Gateway set? | `ip route show` | `ip route add default via x.x.x.x` |
| DNS working? | `ping 8.8.8.8` vs `ping google.com` | Check `/etc/resolv.conf` |

### Cannot Connect to Service

| Check | Command | Fix |
|-------|---------|-----|
| Service running? | `systemctl status service` | `systemctl start service` |
| Listening on port? | `ss -tulpn | grep port` | Check service config |
| Firewall blocking? | `iptables -L -n` | Allow port in firewall |
| Binding to correct interface? | Check config | Bind to 0.0.0.0 |

### Slow Network

| Check | Command | Notes |
|-------|---------|-------|
| Packet loss | `ping -c 100 host` | Look for loss % |
| Latency | `ping host` | Check round-trip time |
| Route issues | `mtr host` | Identify slow hop |
| Bandwidth | `iperf3 -c host` | Test throughput |
| DNS slow | `time dig google.com` | Try different DNS |

### DNS Issues

| Issue | Solution |
|-------|----------|
| Not resolving | Check `/etc/resolv.conf` |
| Wrong IP returned | Flush cache, check hosts file |
| Slow resolution | Try different DNS server |
| Intermittent | Check multiple nameservers |

## Quick Diagnostic Script

```bash
#!/bin/bash

echo "=== Network Diagnostics ==="
echo ""

echo "1. Interface Status:"
ip -br addr
echo ""

echo "2. Default Gateway:"
ip route | grep default
echo ""

echo "3. Ping Tests:"
echo -n "   Localhost: "
ping -c 1 127.0.0.1 &>/dev/null && echo "OK" || echo "FAIL"
echo -n "   Gateway: "
ping -c 1 $(ip route | grep default | awk '{print $3}') &>/dev/null && echo "OK" || echo "FAIL"
echo -n "   Internet (8.8.8.8): "
ping -c 1 8.8.8.8 &>/dev/null && echo "OK" || echo "FAIL"
echo -n "   DNS (google.com): "
ping -c 1 google.com &>/dev/null && echo "OK" || echo "FAIL"
echo ""

echo "4. DNS Servers:"
grep nameserver /etc/resolv.conf
echo ""

echo "5. Listening Ports:"
ss -tulpn | head -20
echo ""

echo "6. Firewall Status:"
if command -v ufw &>/dev/null; then
    ufw status
elif command -v firewall-cmd &>/dev/null; then
    firewall-cmd --state
else
    iptables -L -n | head -10
fi
```

## Quick Reference

### Connectivity Tests

| Test | Command |
|------|---------|
| Ping host | `ping -c 4 hostname` |
| Check port | `nc -zv hostname port` |
| Trace route | `traceroute hostname` |
| DNS lookup | `dig hostname` |

### Status Commands

| Check | Command |
|-------|---------|
| IP address | `ip addr show` |
| Routes | `ip route show` |
| Listening ports | `ss -tulpn` |
| Connections | `ss -t state established` |
| DNS config | `cat /etc/resolv.conf` |

### Analysis Tools

| Tool | Use For |
|------|---------|
| tcpdump | Packet capture |
| wireshark | GUI packet analysis |
| nmap | Port scanning |
| mtr | Continuous trace |
| iperf3 | Bandwidth test |

---

**Previous:** [12-vpn-basics.md](12-vpn-basics.md)

---

## Congratulations!

You've completed the Networking section of the DevOps roadmap. You now understand:

- Network fundamentals and protocols
- IP addressing and subnetting
- DNS, DHCP, HTTP/HTTPS
- Firewalls and security
- Load balancing and proxies
- VPNs and troubleshooting

**Next Topic:** [03-Git](../03-Git/)
