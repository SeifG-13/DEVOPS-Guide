# Networking Fundamentals

## Why Networking for DevOps?

As a DevOps engineer, you need to understand networking to:
- Configure servers and services
- Troubleshoot connectivity issues
- Set up secure communications
- Design cloud architectures
- Manage containers and microservices

## OSI Model (7 Layers)

The OSI (Open Systems Interconnection) model is a conceptual framework for understanding network communication.

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 7: Application    │ HTTP, HTTPS, FTP, SSH, DNS, SMTP │
├─────────────────────────────────────────────────────────────┤
│  Layer 6: Presentation   │ Encryption, Compression, SSL/TLS │
├─────────────────────────────────────────────────────────────┤
│  Layer 5: Session        │ Session management, Authentication│
├─────────────────────────────────────────────────────────────┤
│  Layer 4: Transport      │ TCP, UDP, Ports                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Network        │ IP, ICMP, Routing                 │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Data Link      │ MAC addresses, Ethernet, Switches │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Physical       │ Cables, Hubs, Physical signals    │
└─────────────────────────────────────────────────────────────┘
```

### Layer Details

| Layer | Name | Function | Examples | Data Unit |
|-------|------|----------|----------|-----------|
| 7 | Application | User interface, app protocols | HTTP, FTP, SSH | Data |
| 6 | Presentation | Data formatting, encryption | SSL, JPEG, ASCII | Data |
| 5 | Session | Session management | NetBIOS, RPC | Data |
| 4 | Transport | End-to-end delivery | TCP, UDP | Segment |
| 3 | Network | Logical addressing, routing | IP, ICMP | Packet |
| 2 | Data Link | Physical addressing | Ethernet, MAC | Frame |
| 1 | Physical | Physical transmission | Cables, Hubs | Bits |

### Mnemonic

**Top to Bottom:** All People Seem To Need Data Processing

**Bottom to Top:** Please Do Not Throw Sausage Pizza Away

## TCP/IP Model (4 Layers)

The practical model used in real-world networking.

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Application    │ HTTP, FTP, SSH, DNS, SMTP        │
│                          │ (OSI Layers 5, 6, 7)              │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Transport      │ TCP, UDP                          │
│                          │ (OSI Layer 4)                     │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Internet       │ IP, ICMP, ARP                     │
│                          │ (OSI Layer 3)                     │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Network Access │ Ethernet, Wi-Fi, MAC              │
│                          │ (OSI Layers 1, 2)                 │
└─────────────────────────────────────────────────────────────┘
```

### OSI vs TCP/IP

| OSI Model | TCP/IP Model |
|-----------|--------------|
| 7 Layers | 4 Layers |
| Theoretical | Practical |
| Protocol independent | Based on standard protocols |
| Developed by ISO | Developed by DoD |

## How Data Flows

### Encapsulation (Sending Data)

```
Application Layer:  [        DATA        ]
                              ↓
Transport Layer:    [TCP/UDP][   DATA    ]  ← Segment
                              ↓
Network Layer:      [IP][TCP][   DATA    ]  ← Packet
                              ↓
Data Link Layer:    [MAC][IP][TCP][DATA][FCS]  ← Frame
                              ↓
Physical Layer:     101010101010101010101  ← Bits
```

### Decapsulation (Receiving Data)

The reverse process - each layer strips its header.

## Key Networking Concepts

### MAC Address

- **Layer 2** identifier
- 48-bit hardware address
- Format: `AA:BB:CC:DD:EE:FF`
- Burned into network interface card (NIC)
- Used for local network communication

```bash
# View MAC address
ip link show
ifconfig
```

### IP Address

- **Layer 3** identifier
- Logical address (can be changed)
- IPv4: 32-bit (e.g., `192.168.1.1`)
- IPv6: 128-bit (e.g., `2001:db8::1`)
- Used for routing across networks

```bash
# View IP address
ip addr show
hostname -I
```

### Port Numbers

- **Layer 4** identifier
- 16-bit number (0-65535)
- Identifies specific application/service
- Well-known ports: 0-1023
- Registered ports: 1024-49151
- Dynamic/Private: 49152-65535

### Socket

A combination of IP address and port number:
```
192.168.1.100:8080
[IP Address]:[Port]
```

## Network Devices

| Device | Layer | Function |
|--------|-------|----------|
| Hub | 1 | Broadcasts to all ports |
| Switch | 2 | Forwards based on MAC |
| Router | 3 | Forwards based on IP |
| Firewall | 3-7 | Filters traffic |
| Load Balancer | 4-7 | Distributes traffic |

### How a Switch Works

```
┌─────────────────────────────────────┐
│              Switch                  │
│    MAC Table:                        │
│    Port 1 → AA:BB:CC:DD:EE:01       │
│    Port 2 → AA:BB:CC:DD:EE:02       │
│    Port 3 → AA:BB:CC:DD:EE:03       │
└─────────────────────────────────────┘
     │         │         │
   Host A    Host B    Host C
```

### How a Router Works

```
┌─────────────────────────────────────┐
│              Router                  │
│    Routing Table:                    │
│    192.168.1.0/24 → eth0            │
│    10.0.0.0/8 → eth1                │
│    0.0.0.0/0 → ISP Gateway          │
└─────────────────────────────────────┘
     │                    │
Network A             Network B
192.168.1.0/24        10.0.0.0/8
```

## Network Types

| Type | Description | Range |
|------|-------------|-------|
| LAN | Local Area Network | Building/Office |
| WAN | Wide Area Network | Cities/Countries |
| MAN | Metropolitan Area Network | City |
| WLAN | Wireless LAN | Building (Wi-Fi) |
| VPN | Virtual Private Network | Over Internet |

## Network Topologies

```
Star:                    Bus:
    ┌───┐                ════════════════
    │Hub│                │  │  │  │  │
    └─┬─┘                PC PC PC PC PC
   ╱  │  ╲
  PC  PC  PC

Ring:                    Mesh:
  ┌──PC──┐               PC───PC
  │      │                │╲ ╱│
  PC    PC                │ ╳ │
  │      │                │╱ ╲│
  └──PC──┘               PC───PC
```

## Data Transmission

### Unicast, Broadcast, Multicast

| Type | Description | Example |
|------|-------------|---------|
| Unicast | One to one | HTTP request |
| Broadcast | One to all | ARP request |
| Multicast | One to many | Video streaming |
| Anycast | One to nearest | DNS, CDN |

### Bandwidth vs Latency

| Term | Definition | Measured In |
|------|------------|-------------|
| Bandwidth | Maximum data rate | Mbps, Gbps |
| Latency | Delay/time to destination | ms |
| Throughput | Actual data rate achieved | Mbps |
| Jitter | Variation in latency | ms |

## Quick Reference

| Concept | Layer | Purpose |
|---------|-------|---------|
| MAC Address | 2 | Hardware identification |
| IP Address | 3 | Logical addressing |
| Port | 4 | Service identification |
| Protocol | 4-7 | Communication rules |
| Switch | 2 | Forward by MAC |
| Router | 3 | Forward by IP |

---

**Next:** [02-ip-addressing.md](02-ip-addressing.md)
