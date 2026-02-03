# Subnetting Deep Dive

## Why Subnetting?

Subnetting divides a large network into smaller, manageable segments.

### Benefits

- **Efficient IP usage** - Allocate only what's needed
- **Reduced broadcast traffic** - Smaller broadcast domains
- **Improved security** - Isolate network segments
- **Better organization** - Logical network structure
- **Simplified management** - Easier troubleshooting

## Subnetting Basics

### Key Concepts

```
Given: 192.168.1.0/24

Network Bits:    24 (fixed, identifies network)
Host Bits:       8  (variable, identifies hosts)
Subnet Mask:     255.255.255.0

Binary:
11111111.11111111.11111111.00000000
└────── Network (24 bits) ─┘└Host(8)┘
```

### Subnet Mask Reference Table

| CIDR | Subnet Mask | Binary | Hosts | Subnets from /24 |
|------|-------------|--------|-------|------------------|
| /24 | 255.255.255.0 | 11111111.11111111.11111111.00000000 | 254 | 1 |
| /25 | 255.255.255.128 | 11111111.11111111.11111111.10000000 | 126 | 2 |
| /26 | 255.255.255.192 | 11111111.11111111.11111111.11000000 | 62 | 4 |
| /27 | 255.255.255.224 | 11111111.11111111.11111111.11100000 | 30 | 8 |
| /28 | 255.255.255.240 | 11111111.11111111.11111111.11110000 | 14 | 16 |
| /29 | 255.255.255.248 | 11111111.11111111.11111111.11111000 | 6 | 32 |
| /30 | 255.255.255.252 | 11111111.11111111.11111111.11111100 | 2 | 64 |
| /31 | 255.255.255.254 | 11111111.11111111.11111111.11111110 | 2* | 128 |
| /32 | 255.255.255.255 | 11111111.11111111.11111111.11111111 | 1 | 256 |

**/31 is used for point-to-point links (no network/broadcast needed)*

## Subnetting Formulas

### Calculate Number of Subnets

```
Subnets = 2^(borrowed bits)

Example: Subnetting /24 to /26
Borrowed bits = 26 - 24 = 2
Subnets = 2^2 = 4
```

### Calculate Hosts per Subnet

```
Hosts = 2^(host bits) - 2

Example: /26 network
Host bits = 32 - 26 = 6
Hosts = 2^6 - 2 = 62

(Subtract 2 for network and broadcast addresses)
```

### Block Size / Increment

```
Block Size = 256 - (last octet of subnet mask)

Example: /26 (255.255.255.192)
Block Size = 256 - 192 = 64

Subnets start at: 0, 64, 128, 192
```

## Step-by-Step Subnetting

### Example 1: Subnet 192.168.1.0/24 into 4 equal subnets

**Step 1:** Determine bits needed
```
4 subnets = 2^n where n = 2
Borrow 2 bits from host portion
New prefix: /24 + 2 = /26
```

**Step 2:** Calculate block size
```
/26 = 255.255.255.192
Block size = 256 - 192 = 64
```

**Step 3:** List subnets

| Subnet | Network | First Host | Last Host | Broadcast |
|--------|---------|------------|-----------|-----------|
| 1 | 192.168.1.0 | 192.168.1.1 | 192.168.1.62 | 192.168.1.63 |
| 2 | 192.168.1.64 | 192.168.1.65 | 192.168.1.126 | 192.168.1.127 |
| 3 | 192.168.1.128 | 192.168.1.129 | 192.168.1.190 | 192.168.1.191 |
| 4 | 192.168.1.192 | 192.168.1.193 | 192.168.1.254 | 192.168.1.255 |

### Example 2: Find subnet for IP 10.50.100.200/20

**Step 1:** Find block size
```
/20 = 255.255.240.0
Block size in 3rd octet = 256 - 240 = 16
```

**Step 2:** Find network address
```
Third octet: 100
100 ÷ 16 = 6.25 → 6 × 16 = 96

Network: 10.50.96.0/20
```

**Step 3:** Calculate range
```
Network:   10.50.96.0
Broadcast: 10.50.111.255  (96 + 16 - 1 = 111)
First host: 10.50.96.1
Last host:  10.50.111.254
```

### Example 3: Need 50 hosts per subnet

**Step 1:** Find host bits needed
```
2^n - 2 ≥ 50
2^6 - 2 = 62 ≥ 50 ✓

Need 6 host bits
Prefix = 32 - 6 = /26
```

**Step 2:** Subnet mask
```
/26 = 255.255.255.192
```

## VLSM (Variable Length Subnet Masking)

VLSM allows different subnet sizes within the same network.

### Example: Allocate from 192.168.1.0/24

Requirements:
- Network A: 100 hosts
- Network B: 50 hosts
- Network C: 25 hosts
- Network D: 2 hosts (point-to-point)

**Step 1:** Sort by size (largest first)
```
A: 100 hosts → /25 (126 hosts)
B: 50 hosts  → /26 (62 hosts)
C: 25 hosts  → /27 (30 hosts)
D: 2 hosts   → /30 (2 hosts)
```

**Step 2:** Allocate sequentially

| Network | Required | Allocated | Range | Mask |
|---------|----------|-----------|-------|------|
| A | 100 | /25 (126) | 192.168.1.0 - 192.168.1.127 | 255.255.255.128 |
| B | 50 | /26 (62) | 192.168.1.128 - 192.168.1.191 | 255.255.255.192 |
| C | 25 | /27 (30) | 192.168.1.192 - 192.168.1.223 | 255.255.255.224 |
| D | 2 | /30 (2) | 192.168.1.224 - 192.168.1.227 | 255.255.255.252 |

Remaining: 192.168.1.228 - 192.168.1.255 (28 addresses)

## Supernetting (Route Aggregation)

Combining multiple networks into one larger network.

### Example: Summarize these networks

```
192.168.0.0/24
192.168.1.0/24
192.168.2.0/24
192.168.3.0/24
```

**Step 1:** Convert to binary (3rd octet)
```
0 = 00000000
1 = 00000001
2 = 00000010
3 = 00000011
      ^^^^^^
      Common: 000000
```

**Step 2:** Find common bits
```
6 common bits in 3rd octet
Original network bits: 24
New prefix: 16 + 6 = 22
```

**Result:** 192.168.0.0/22

## Quick Subnetting Methods

### The "Magic Number" Method

1. Find the "interesting octet" (where mask is not 0 or 255)
2. Calculate: 256 - subnet mask value = magic number
3. Subnets increment by magic number

```
Example: 255.255.255.192
Magic number = 256 - 192 = 64
Subnets: .0, .64, .128, .192
```

### Powers of 2 Reference

| Power | Value | CIDR Hosts |
|-------|-------|------------|
| 2^1 | 2 | /31, /30 |
| 2^2 | 4 | /30 |
| 2^3 | 8 | /29 (6) |
| 2^4 | 16 | /28 (14) |
| 2^5 | 32 | /27 (30) |
| 2^6 | 64 | /26 (62) |
| 2^7 | 128 | /25 (126) |
| 2^8 | 256 | /24 (254) |

## Practice Problems

### Problem 1
```
Given: 172.16.0.0/16
Required: 100 subnets
Find: New subnet mask and hosts per subnet

Solution:
2^n ≥ 100 → n = 7 (128 subnets)
New prefix: /16 + 7 = /23
Hosts: 2^9 - 2 = 510
Mask: 255.255.254.0
```

### Problem 2
```
Given: 10.0.0.0/8
Required: 1000 hosts per subnet
Find: Subnet mask

Solution:
2^n - 2 ≥ 1000 → n = 10 (1022 hosts)
Host bits: 10
Network bits: 32 - 10 = 22
Mask: /22 = 255.255.252.0
```

### Problem 3
```
IP: 172.20.45.100/21
Find: Network, Broadcast, Range

Solution:
/21 = 255.255.248.0
Block size: 256 - 248 = 8 (3rd octet)
45 ÷ 8 = 5.x → 5 × 8 = 40

Network: 172.20.40.0
Broadcast: 172.20.47.255
Range: 172.20.40.1 - 172.20.47.254
Hosts: 2^11 - 2 = 2046
```

## Subnetting Cheat Sheet

```
/30 = 4 IPs, 2 hosts     (point-to-point)
/29 = 8 IPs, 6 hosts     (small office)
/28 = 16 IPs, 14 hosts   (small office)
/27 = 32 IPs, 30 hosts   (small office)
/26 = 64 IPs, 62 hosts   (medium office)
/25 = 128 IPs, 126 hosts (large office)
/24 = 256 IPs, 254 hosts (standard LAN)
/23 = 512 IPs, 510 hosts
/22 = 1024 IPs, 1022 hosts
/21 = 2048 IPs, 2046 hosts
/20 = 4096 IPs, 4094 hosts
/16 = 65536 IPs, 65534 hosts
```

---

**Previous:** [02-ip-addressing.md](02-ip-addressing.md) | **Next:** [04-tcp-udp.md](04-tcp-udp.md)
