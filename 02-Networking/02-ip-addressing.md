# IP Addressing

## IPv4 Addressing

### Structure

IPv4 addresses are 32-bit numbers, written in dotted decimal notation.

```
192.168.1.100
│   │   │  │
│   │   │  └── Octet 4 (8 bits)
│   │   └───── Octet 3 (8 bits)
│   └───────── Octet 2 (8 bits)
└───────────── Octet 1 (8 bits)

Binary: 11000000.10101000.00000001.01100100
Decimal: 192.168.1.100
```

### Octet Range

Each octet can be 0-255 (2^8 = 256 values)

```
Minimum: 0.0.0.0
Maximum: 255.255.255.255
Total addresses: 2^32 = 4,294,967,296
```

### Binary Conversion

```
128  64  32  16   8   4   2   1
 │    │   │   │   │   │   │   │
 2^7 2^6 2^5 2^4 2^3 2^2 2^1 2^0

Example: 192
128 + 64 = 192
Binary: 11000000

Example: 168
128 + 32 + 8 = 168
Binary: 10101000
```

## IP Address Classes (Classful - Legacy)

| Class | First Octet | Range | Default Mask | Networks | Hosts/Network |
|-------|-------------|-------|--------------|----------|---------------|
| A | 1-126 | 1.0.0.0 - 126.255.255.255 | 255.0.0.0 (/8) | 126 | 16,777,214 |
| B | 128-191 | 128.0.0.0 - 191.255.255.255 | 255.255.0.0 (/16) | 16,384 | 65,534 |
| C | 192-223 | 192.0.0.0 - 223.255.255.255 | 255.255.255.0 (/24) | 2,097,152 | 254 |
| D | 224-239 | 224.0.0.0 - 239.255.255.255 | Multicast | - | - |
| E | 240-255 | 240.0.0.0 - 255.255.255.255 | Reserved | - | - |

**Note:** 127.x.x.x is reserved for loopback (localhost)

## Private vs Public IP Addresses

### Private IP Ranges (RFC 1918)

| Class | Range | CIDR | # of Addresses |
|-------|-------|------|----------------|
| A | 10.0.0.0 - 10.255.255.255 | 10.0.0.0/8 | 16,777,216 |
| B | 172.16.0.0 - 172.31.255.255 | 172.16.0.0/12 | 1,048,576 |
| C | 192.168.0.0 - 192.168.255.255 | 192.168.0.0/16 | 65,536 |

### Characteristics

| Private IP | Public IP |
|------------|-----------|
| Not routable on internet | Routable on internet |
| Free to use | Assigned by ISP/RIR |
| Used in internal networks | Used for internet access |
| Requires NAT for internet | Directly accessible |
| Can be reused | Globally unique |

### Special Addresses

| Address | Purpose |
|---------|---------|
| 0.0.0.0 | Default route / All interfaces |
| 127.0.0.1 | Localhost (loopback) |
| 127.0.0.0/8 | Loopback range |
| 169.254.0.0/16 | Link-local (APIPA) |
| 255.255.255.255 | Broadcast |

## CIDR Notation

CIDR (Classless Inter-Domain Routing) replaced classful addressing.

### Format

```
IP Address / Prefix Length
192.168.1.0/24
```

### Common CIDR Blocks

| CIDR | Subnet Mask | # Hosts | Wildcard |
|------|-------------|---------|----------|
| /8 | 255.0.0.0 | 16,777,214 | 0.255.255.255 |
| /16 | 255.255.0.0 | 65,534 | 0.0.255.255 |
| /24 | 255.255.255.0 | 254 | 0.0.0.255 |
| /25 | 255.255.255.128 | 126 | 0.0.0.127 |
| /26 | 255.255.255.192 | 62 | 0.0.0.63 |
| /27 | 255.255.255.224 | 30 | 0.0.0.31 |
| /28 | 255.255.255.240 | 14 | 0.0.0.15 |
| /29 | 255.255.255.248 | 6 | 0.0.0.7 |
| /30 | 255.255.255.252 | 2 | 0.0.0.3 |
| /31 | 255.255.255.254 | 2 | 0.0.0.1 |
| /32 | 255.255.255.255 | 1 | 0.0.0.0 |

### CIDR Calculation

```
/24 = 24 network bits, 8 host bits
Hosts = 2^8 - 2 = 254 (subtract network and broadcast)

/26 = 26 network bits, 6 host bits
Hosts = 2^6 - 2 = 62
```

## Subnet Mask

### Understanding Subnet Masks

```
IP Address:    192.168.1.100
Subnet Mask:   255.255.255.0

Binary:
IP:    11000000.10101000.00000001.01100100
Mask:  11111111.11111111.11111111.00000000
       └────── Network ────────┘└─ Host ─┘

Network Address: 192.168.1.0
Host Portion:    .100
```

### Network and Broadcast Address

```
Network: 192.168.1.0/24

Network Address:   192.168.1.0     (first, all host bits = 0)
First Host:        192.168.1.1
Last Host:         192.168.1.254
Broadcast Address: 192.168.1.255   (last, all host bits = 1)
```

## IPv6 Addressing

### Structure

128-bit addresses written in hexadecimal, separated by colons.

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
│    │    │    │    │    │    │    │
└────┴────┴────┴────┴────┴────┴────┴── 8 groups of 16 bits
```

### Simplification Rules

1. **Leading zeros** can be omitted
2. **Consecutive zero groups** can be replaced with :: (once)

```
Full:        2001:0db8:0000:0000:0000:0000:0000:0001
Simplified:  2001:db8::1
```

### IPv6 Address Types

| Type | Prefix | Description |
|------|--------|-------------|
| Global Unicast | 2000::/3 | Public addresses |
| Link-Local | fe80::/10 | Auto-configured, non-routable |
| Unique Local | fc00::/7 | Private addresses |
| Multicast | ff00::/8 | One to many |
| Loopback | ::1 | Localhost |
| Unspecified | :: | No address |

### IPv4 vs IPv6

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address Size | 32 bits | 128 bits |
| Address Format | Decimal | Hexadecimal |
| Example | 192.168.1.1 | 2001:db8::1 |
| Total Addresses | 4.3 billion | 340 undecillion |
| NAT | Required | Not needed |
| DHCP | Optional | DHCPv6 or SLAAC |
| IPSec | Optional | Built-in |

## IP Configuration in Linux

### View IP Address

```bash
# Modern command
ip addr show
ip a

# Specific interface
ip addr show eth0

# Legacy command
ifconfig
ifconfig eth0
```

### Temporary Configuration

```bash
# Add IP address
sudo ip addr add 192.168.1.100/24 dev eth0

# Remove IP address
sudo ip addr del 192.168.1.100/24 dev eth0

# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down
```

### Permanent Configuration (Ubuntu/Netplan)

```yaml
# /etc/netplan/01-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
sudo netplan apply
```

### Permanent Configuration (RHEL/CentOS)

```bash
# /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE=Ethernet
BOOTPROTO=static
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
ONBOOT=yes
```

```bash
sudo systemctl restart NetworkManager
```

## Quick Reference

| Term | Description |
|------|-------------|
| Octet | 8-bit section of IPv4 (0-255) |
| Subnet Mask | Divides IP into network/host |
| CIDR | Notation: IP/prefix length |
| Network Address | First address in subnet |
| Broadcast Address | Last address in subnet |
| Private IP | Not routable on internet |
| Public IP | Routable on internet |
| NAT | Translates private to public |

---

**Previous:** [01-networking-fundamentals.md](01-networking-fundamentals.md) | **Next:** [03-subnetting-deep-dive.md](03-subnetting-deep-dive.md)
