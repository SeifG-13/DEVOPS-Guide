# Networking Commands

## Network Configuration

### ip Command (Modern)

```bash
# Show all interfaces
ip addr
ip a

# Show specific interface
ip addr show eth0

# Show only IPv4
ip -4 addr

# Show only IPv6
ip -6 addr

# Add IP address
sudo ip addr add 192.168.1.100/24 dev eth0

# Remove IP address
sudo ip addr del 192.168.1.100/24 dev eth0

# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Show link status
ip link
ip link show eth0
```

### ifconfig Command (Legacy)

```bash
# Show all interfaces
ifconfig
ifconfig -a

# Show specific interface
ifconfig eth0

# Set IP address
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0

# Bring up/down
sudo ifconfig eth0 up
sudo ifconfig eth0 down
```

### Routing

```bash
# Show routing table
ip route
route -n
netstat -rn

# Add route
sudo ip route add 10.0.0.0/8 via 192.168.1.1
sudo ip route add 10.0.0.0/8 via 192.168.1.1 dev eth0

# Delete route
sudo ip route del 10.0.0.0/8

# Add default gateway
sudo ip route add default via 192.168.1.1

# Delete default gateway
sudo ip route del default
```

## Connection Testing

### ping - Test Connectivity

```bash
# Basic ping
ping google.com

# Specify count
ping -c 4 google.com

# Continuous ping with interval
ping -i 0.5 google.com

# Quiet mode (summary only)
ping -q -c 10 google.com

# Set packet size
ping -s 1000 google.com

# Flood ping (careful!)
sudo ping -f google.com
```

### traceroute - Trace Path

```bash
# Trace route to host
traceroute google.com

# Use ICMP
traceroute -I google.com

# Use TCP
traceroute -T google.com

# Set max hops
traceroute -m 20 google.com

# Don't resolve hostnames
traceroute -n google.com

# mtr - combined ping and traceroute
mtr google.com
mtr -r -c 10 google.com   # Report mode
```

## Port and Connection Status

### netstat (Legacy)

```bash
# All connections
netstat -a

# Listening ports
netstat -l

# TCP connections
netstat -t

# UDP connections
netstat -u

# Show PIDs
netstat -p

# Numeric output (no DNS)
netstat -n

# Combined common options
netstat -tulpn

# Routing table
netstat -r

# Interface statistics
netstat -i
```

### ss (Modern Replacement)

```bash
# All sockets
ss -a

# Listening sockets
ss -l

# TCP sockets
ss -t

# UDP sockets
ss -u

# Show processes
ss -p

# Numeric output
ss -n

# Combined common options
ss -tulpn

# Filter by state
ss state established
ss state listening

# Filter by port
ss -t dst :443
ss -t src :22

# Filter by address
ss -t dst 192.168.1.1
```

### lsof - List Open Files

```bash
# Network connections
lsof -i

# Specific port
lsof -i :80
lsof -i :22

# TCP only
lsof -i TCP

# UDP only
lsof -i UDP

# Specific address
lsof -i @192.168.1.1

# By user
lsof -u username -i

# By process
lsof -p 1234 -i
```

## DNS Tools

### nslookup

```bash
# Basic lookup
nslookup google.com

# Specific DNS server
nslookup google.com 8.8.8.8

# Query type
nslookup -type=mx google.com
nslookup -type=ns google.com
nslookup -type=a google.com

# Reverse lookup
nslookup 8.8.8.8
```

### dig (More Powerful)

```bash
# Basic lookup
dig google.com

# Short answer
dig +short google.com

# Specific record type
dig google.com MX
dig google.com NS
dig google.com A
dig google.com AAAA
dig google.com TXT

# Use specific DNS server
dig @8.8.8.8 google.com

# Reverse lookup
dig -x 8.8.8.8

# Trace delegation
dig +trace google.com

# All records
dig google.com ANY
```

### host

```bash
# Basic lookup
host google.com

# Reverse lookup
host 8.8.8.8

# Specific record
host -t MX google.com
host -t NS google.com
```

### /etc/hosts

```bash
# View hosts file
cat /etc/hosts

# Edit hosts file
sudo vim /etc/hosts

# Example entries
# 127.0.0.1    localhost
# 192.168.1.100    myserver.local myserver
```

### /etc/resolv.conf

```bash
# View DNS configuration
cat /etc/resolv.conf

# Example content
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# search example.com
```

## Data Transfer

### curl - Transfer Data

```bash
# GET request
curl https://api.example.com

# Save to file
curl -o file.html https://example.com
curl -O https://example.com/file.zip

# Follow redirects
curl -L https://example.com

# Show headers only
curl -I https://example.com

# Include headers in output
curl -i https://example.com

# POST request
curl -X POST -d "data=value" https://api.example.com

# POST JSON
curl -X POST -H "Content-Type: application/json" \
     -d '{"key":"value"}' https://api.example.com

# Custom headers
curl -H "Authorization: Bearer token" https://api.example.com

# Basic auth
curl -u username:password https://api.example.com

# Verbose output
curl -v https://example.com

# Silent mode
curl -s https://example.com

# Timeout
curl --connect-timeout 5 -m 10 https://example.com

# Upload file
curl -F "file=@/path/to/file" https://api.example.com/upload
```

### wget - Download Files

```bash
# Download file
wget https://example.com/file.zip

# Save with different name
wget -O output.zip https://example.com/file.zip

# Download in background
wget -b https://example.com/file.zip

# Resume download
wget -c https://example.com/file.zip

# Limit speed
wget --limit-rate=1m https://example.com/file.zip

# Mirror website
wget -m https://example.com

# Recursive download
wget -r -l 2 https://example.com

# Quiet mode
wget -q https://example.com/file.zip

# Download multiple files
wget -i urls.txt
```

## Network Configuration Files

### Debian/Ubuntu (Netplan - Modern)

```yaml
# /etc/netplan/01-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
# Apply netplan
sudo netplan apply
```

### RHEL/CentOS (NetworkManager)

```bash
# /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
BOOTPROTO=static
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
ONBOOT=yes

# Restart network
sudo systemctl restart NetworkManager
```

### nmcli (NetworkManager CLI)

```bash
# Show connections
nmcli con show

# Show devices
nmcli dev status

# Connect/disconnect
nmcli con up "Connection Name"
nmcli con down "Connection Name"

# Add static IP
nmcli con add type ethernet con-name static-eth0 ifname eth0 \
      ip4 192.168.1.100/24 gw4 192.168.1.1

# Set DNS
nmcli con mod static-eth0 ipv4.dns "8.8.8.8 8.8.4.4"

# Set DHCP
nmcli con mod eth0 ipv4.method auto
```

## Network Troubleshooting

### Quick Checks

```bash
# Check interface status
ip link show

# Check IP addresses
ip addr show

# Check routing
ip route show

# Check DNS
cat /etc/resolv.conf

# Test connectivity
ping -c 4 8.8.8.8

# Test DNS resolution
ping -c 4 google.com

# Check listening ports
ss -tulpn
```

### Common Issues

| Issue | Check |
|-------|-------|
| No IP address | `ip addr`, `dhclient eth0` |
| Can't reach gateway | `ping <gateway>`, check cables |
| DNS not working | `ping 8.8.8.8`, check `/etc/resolv.conf` |
| Port not accessible | `ss -tulpn`, check firewall |
| Slow network | `mtr <host>`, check for packet loss |

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `ip addr` | Show IP addresses | `ip addr show eth0` |
| `ip route` | Show/modify routes | `ip route` |
| `ping` | Test connectivity | `ping -c 4 google.com` |
| `traceroute` | Trace network path | `traceroute google.com` |
| `ss` | Socket statistics | `ss -tulpn` |
| `netstat` | Network statistics | `netstat -tulpn` |
| `dig` | DNS lookup | `dig google.com` |
| `nslookup` | DNS lookup | `nslookup google.com` |
| `curl` | Transfer data | `curl -O url` |
| `wget` | Download files | `wget url` |
| `lsof -i` | Network connections | `lsof -i :80` |
| `nmcli` | Network Manager | `nmcli con show` |

---

**Previous:** [15-package-management.md](15-package-management.md) | **Next:** [17-ssh-scp.md](17-ssh-scp.md)
