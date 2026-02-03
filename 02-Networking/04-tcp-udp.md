# TCP & UDP

## Transport Layer Overview

The transport layer (Layer 4) provides end-to-end communication between applications.

```
┌─────────────────────────────────────────────────────────────┐
│                    Transport Layer                          │
│                                                              │
│    ┌─────────────────┐        ┌─────────────────┐           │
│    │       TCP       │        │       UDP       │           │
│    │  (Reliable)     │        │  (Fast)         │           │
│    │  Connection-    │        │  Connectionless │           │
│    │  oriented       │        │                 │           │
│    └─────────────────┘        └─────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

## TCP (Transmission Control Protocol)

### Characteristics

| Feature | Description |
|---------|-------------|
| Connection-oriented | Establishes connection before data transfer |
| Reliable | Guarantees delivery and order |
| Error checking | Checksums and acknowledgments |
| Flow control | Prevents sender from overwhelming receiver |
| Congestion control | Adjusts to network conditions |
| Ordered delivery | Data arrives in correct sequence |

### TCP Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────────────┼───────────────────────────────┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┴───────────────────────────────┤
│                        Sequence Number                        │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                      │
├───────┬───────┬───────────────┼───────────────────────────────┤
│ Data  │       │C│E│U│A│P│R│S│F│                               │
│Offset │ Res.  │W│C│R│C│S│S│Y│I│           Window              │
│       │       │R│E│G│K│H│T│N│N│                               │
├───────┴───────┴───────────────┼───────────────────────────────┤
│          Checksum             │         Urgent Pointer        │
├───────────────────────────────┴───────────────────────────────┤
│                    Options (if any)                           │
└───────────────────────────────────────────────────────────────┘
```

### TCP Flags

| Flag | Name | Purpose |
|------|------|---------|
| SYN | Synchronize | Initiate connection |
| ACK | Acknowledge | Confirm receipt |
| FIN | Finish | Terminate connection |
| RST | Reset | Abort connection |
| PSH | Push | Send data immediately |
| URG | Urgent | Priority data |

### TCP Three-Way Handshake

```
    Client                              Server
       │                                   │
       │──────── SYN (seq=100) ───────────►│
       │                                   │
       │◄─── SYN-ACK (seq=300,ack=101) ────│
       │                                   │
       │──────── ACK (ack=301) ───────────►│
       │                                   │
       │     Connection Established        │
       │                                   │
```

**Step by step:**
1. **SYN**: Client sends SYN with initial sequence number
2. **SYN-ACK**: Server responds with SYN-ACK, acknowledges client's seq+1
3. **ACK**: Client acknowledges server's seq+1

### TCP Connection Termination (Four-Way Handshake)

```
    Client                              Server
       │                                   │
       │──────── FIN (seq=100) ───────────►│
       │                                   │
       │◄─────── ACK (ack=101) ────────────│
       │                                   │
       │◄─────── FIN (seq=300) ────────────│
       │                                   │
       │──────── ACK (ack=301) ───────────►│
       │                                   │
       │     Connection Closed             │
```

### TCP States

```
                    ┌──────────────┐
                    │    CLOSED    │
                    └──────┬───────┘
                           │ (passive open)
                    ┌──────▼───────┐
                    │    LISTEN    │
                    └──────┬───────┘
                           │ (receive SYN)
                    ┌──────▼───────┐
                    │   SYN_RCVD   │
                    └──────┬───────┘
                           │ (receive ACK)
                    ┌──────▼───────┐
                    │ ESTABLISHED  │
                    └──────┬───────┘
                           │ (close)
                    ┌──────▼───────┐
                    │   FIN_WAIT   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  TIME_WAIT   │
                    └──────┬───────┘
                           │ (timeout)
                    ┌──────▼───────┐
                    │    CLOSED    │
                    └──────────────┘
```

### View TCP Connections

```bash
# All TCP connections
ss -t
netstat -t

# TCP listening ports
ss -tl
netstat -tl

# TCP with process info
ss -tlp
netstat -tlp

# TCP connection states
ss -t state established
ss -t state time-wait
ss -t state close-wait
```

## UDP (User Datagram Protocol)

### Characteristics

| Feature | Description |
|---------|-------------|
| Connectionless | No connection setup |
| Unreliable | No guarantee of delivery |
| No ordering | Packets may arrive out of order |
| No flow control | Sender can overwhelm receiver |
| Fast | Low overhead |
| Simple | Minimal header |

### UDP Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────────────┼───────────────────────────────┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┼───────────────────────────────┤
│            Length             │           Checksum            │
├───────────────────────────────┴───────────────────────────────┤
│                             Data                              │
└───────────────────────────────────────────────────────────────┘
```

**UDP Header Size: 8 bytes** (vs TCP's 20+ bytes)

### View UDP Connections

```bash
# All UDP sockets
ss -u
netstat -u

# UDP listening ports
ss -ul
netstat -ul

# UDP with process info
ss -ulp
```

## TCP vs UDP Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Ordered | Unordered |
| Speed | Slower | Faster |
| Overhead | Higher (20+ bytes) | Lower (8 bytes) |
| Flow Control | Yes | No |
| Error Checking | Extensive | Basic checksum |
| Use Case | Accuracy critical | Speed critical |

### When to Use TCP

- Web browsing (HTTP/HTTPS)
- Email (SMTP, IMAP, POP3)
- File transfer (FTP, SFTP)
- Remote access (SSH, Telnet)
- Database connections

### When to Use UDP

- Live video/audio streaming
- Online gaming
- VoIP
- DNS queries
- DHCP
- SNMP

## Port Numbers

### Port Ranges

| Range | Name | Description |
|-------|------|-------------|
| 0-1023 | Well-known | System/standard services |
| 1024-49151 | Registered | User applications |
| 49152-65535 | Dynamic/Private | Temporary/ephemeral |

### Common TCP Ports

| Port | Service | Description |
|------|---------|-------------|
| 20 | FTP-Data | File transfer data |
| 21 | FTP | File transfer control |
| 22 | SSH | Secure shell |
| 23 | Telnet | Remote login (insecure) |
| 25 | SMTP | Email sending |
| 80 | HTTP | Web traffic |
| 110 | POP3 | Email retrieval |
| 143 | IMAP | Email retrieval |
| 443 | HTTPS | Secure web traffic |
| 465 | SMTPS | Secure SMTP |
| 587 | Submission | Email submission |
| 993 | IMAPS | Secure IMAP |
| 995 | POP3S | Secure POP3 |
| 3306 | MySQL | Database |
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache/database |
| 8080 | HTTP-Alt | Alternative HTTP |

### Common UDP Ports

| Port | Service | Description |
|------|---------|-------------|
| 53 | DNS | Domain name resolution |
| 67 | DHCP Server | IP assignment |
| 68 | DHCP Client | IP request |
| 69 | TFTP | Trivial file transfer |
| 123 | NTP | Time synchronization |
| 161 | SNMP | Network management |
| 162 | SNMP Trap | SNMP notifications |
| 500 | IKE | VPN key exchange |
| 514 | Syslog | System logging |

### Check Ports

```bash
# Check if port is open
nc -zv hostname 80
telnet hostname 80

# Check listening ports
ss -tulpn

# Check specific port
ss -tulpn | grep :80
lsof -i :80

# Scan ports (nmap)
nmap hostname
nmap -p 80,443 hostname
```

## Socket Programming Concept

```
Server Socket: IP + Port (listening)
Example: 192.168.1.100:80

Client Socket: IP + Port (connecting)
Example: 192.168.1.50:52000

Connection Tuple:
(Source IP, Source Port, Dest IP, Dest Port, Protocol)
(192.168.1.50, 52000, 192.168.1.100, 80, TCP)
```

## Quick Reference

| Protocol | Connection | Reliable | Ordered | Use Case |
|----------|------------|----------|---------|----------|
| TCP | Yes | Yes | Yes | Web, Email, SSH |
| UDP | No | No | No | DNS, Streaming |

| Port Type | Range | Example |
|-----------|-------|---------|
| Well-known | 0-1023 | 80, 443, 22 |
| Registered | 1024-49151 | 3306, 8080 |
| Dynamic | 49152-65535 | Client ports |

---

**Previous:** [03-subnetting-deep-dive.md](03-subnetting-deep-dive.md) | **Next:** [05-dns.md](05-dns.md)
