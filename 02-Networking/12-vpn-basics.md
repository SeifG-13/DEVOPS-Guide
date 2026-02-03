# VPN Basics

## What is a VPN?

VPN (Virtual Private Network) creates a secure, encrypted tunnel over a public network (like the internet).

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│   Your Device              Internet              VPN Server       │
│                                                                   │
│   ┌────────┐          ╔════════════════╗        ┌──────────┐     │
│   │        │          ║   Encrypted    ║        │          │     │
│   │ Client │══════════║    Tunnel      ║════════│  Server  │────►│
│   │        │          ║                ║        │          │     │
│   └────────┘          ╚════════════════╝        └──────────┘     │
│                                                                   │
│   Your IP: Hidden                              Server IP: Visible │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## Why Use VPN?

| Use Case | Description |
|----------|-------------|
| Privacy | Hide your IP and location |
| Security | Encrypt traffic on public WiFi |
| Remote Access | Connect to corporate network |
| Site-to-Site | Connect office networks |
| Bypass Geo-blocks | Access region-restricted content |
| Avoid Censorship | Access blocked websites |

## VPN Types

### By Use Case

| Type | Description | Example |
|------|-------------|---------|
| Remote Access | Individual connects to network | Employee to corporate |
| Site-to-Site | Connect two networks | Office A to Office B |
| Client-to-Client | Direct connection between users | Mesh VPN |

### By Architecture

```
Remote Access VPN:
┌────────┐                              ┌─────────────────────┐
│ Remote │══════════ Internet ══════════│ Corporate Network   │
│  User  │         (Encrypted)          │    VPN Gateway      │
└────────┘                              └─────────────────────┘

Site-to-Site VPN:
┌─────────────────┐                     ┌─────────────────┐
│   Office A      │                     │    Office B     │
│  ┌───────────┐  │                     │  ┌───────────┐  │
│  │VPN Gateway│══════ Internet ════════│  │VPN Gateway│  │
│  └───────────┘  │   (Encrypted)       │  └───────────┘  │
└─────────────────┘                     └─────────────────┘
```

## VPN Protocols

### Common Protocols

| Protocol | Security | Speed | Port | Notes |
|----------|----------|-------|------|-------|
| OpenVPN | High | Medium | 1194 UDP/TCP | Open source, flexible |
| WireGuard | High | Fast | 51820 UDP | Modern, simple |
| IPSec/IKEv2 | High | Fast | 500, 4500 UDP | Native on many OS |
| L2TP/IPSec | Medium | Medium | 1701, 500, 4500 | Often blocked |
| PPTP | Low | Fast | 1723 | Deprecated, insecure |
| SSTP | High | Medium | 443 TCP | Microsoft, firewall-friendly |

### Protocol Comparison

```
Security:     WireGuard ≥ OpenVPN > IKEv2 > L2TP > PPTP
Speed:        WireGuard > IKEv2 > OpenVPN > L2TP > PPTP
Simplicity:   WireGuard > IKEv2 > L2TP > OpenVPN > PPTP
Compatibility: OpenVPN > IKEv2 > L2TP > PPTP > WireGuard
```

## OpenVPN

### Server Setup (Ubuntu)

```bash
# Install OpenVPN
sudo apt install openvpn easy-rsa

# Set up PKI
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret ta.key

# Generate client certificate
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

### Server Configuration

```bash
# /etc/openvpn/server.conf

port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0

server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"

keepalive 10 120
cipher AES-256-GCM
user nobody
group nogroup
persist-key
persist-tun
status /var/log/openvpn-status.log
verb 3
```

### Client Configuration

```bash
# client.ovpn

client
dev tun
proto udp
remote vpn.example.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
verb 3

<ca>
# CA certificate content
</ca>

<cert>
# Client certificate content
</cert>

<key>
# Client private key content
</key>

<tls-auth>
# TLS auth key content
</tls-auth>
key-direction 1
```

### Start OpenVPN

```bash
# Server
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server

# Client
sudo openvpn --config client.ovpn
```

## WireGuard

### Server Setup

```bash
# Install WireGuard
sudo apt install wireguard

# Generate keys
wg genkey | tee privatekey | wg pubkey > publickey

# Server configuration
sudo vim /etc/wireguard/wg0.conf
```

```ini
# /etc/wireguard/wg0.conf (Server)

[Interface]
PrivateKey = <server_private_key>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

### Client Setup

```ini
# /etc/wireguard/wg0.conf (Client)

[Interface]
PrivateKey = <client_private_key>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <server_public_key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### Start WireGuard

```bash
# Start
sudo wg-quick up wg0

# Enable at boot
sudo systemctl enable wg-quick@wg0

# Check status
sudo wg show

# Stop
sudo wg-quick down wg0
```

## IPSec/IKEv2 (StrongSwan)

```bash
# Install
sudo apt install strongswan strongswan-pki

# Generate CA
ipsec pki --gen --outform pem > ca-key.pem
ipsec pki --self --ca --lifetime 3650 --in ca-key.pem --outform pem > ca-cert.pem

# Generate server cert
ipsec pki --gen --outform pem > server-key.pem
ipsec pki --pub --in server-key.pem | ipsec pki --issue --lifetime 365 \
    --cacert ca-cert.pem --cakey ca-key.pem --dn "CN=vpn.example.com" \
    --san vpn.example.com --flag serverAuth --flag ikeIntermediate \
    --outform pem > server-cert.pem
```

## VPN Troubleshooting

### Common Issues

| Issue | Check |
|-------|-------|
| Can't connect | Firewall blocking VPN ports? |
| Connected but no internet | DNS settings, routing |
| Slow speeds | Try different protocol/server |
| Disconnects | Check keepalive settings |

### Debugging Commands

```bash
# OpenVPN
sudo openvpn --config client.ovpn --verb 5
journalctl -u openvpn@server -f

# WireGuard
sudo wg show
ping 10.0.0.1  # Ping VPN gateway

# IPSec
ipsec statusall
ipsec status

# General
ip route show
ip addr show
```

## Split Tunneling

Only route specific traffic through VPN.

```
Full Tunnel:        All traffic → VPN
Split Tunnel:       Selected traffic → VPN
                    Other traffic → Direct
```

### WireGuard Split Tunnel

```ini
[Peer]
# Only route specific subnets through VPN
AllowedIPs = 10.0.0.0/8, 192.168.1.0/24

# Full tunnel (all traffic)
# AllowedIPs = 0.0.0.0/0
```

### OpenVPN Split Tunnel

```bash
# Server: Don't push default gateway
# Remove: push "redirect-gateway def1"

# Push only specific routes
push "route 10.0.0.0 255.0.0.0"
push "route 192.168.1.0 255.255.255.0"
```

## Quick Reference

| Protocol | Best For | Port |
|----------|----------|------|
| WireGuard | Speed + simplicity | 51820/UDP |
| OpenVPN | Compatibility | 1194/UDP or 443/TCP |
| IKEv2 | Mobile devices | 500, 4500/UDP |

| Command | Purpose |
|---------|---------|
| `wg show` | WireGuard status |
| `wg-quick up wg0` | Start WireGuard |
| `openvpn --config file.ovpn` | Start OpenVPN |
| `ipsec statusall` | IPSec status |

---

**Previous:** [11-proxy-reverse-proxy.md](11-proxy-reverse-proxy.md) | **Next:** [13-network-troubleshooting.md](13-network-troubleshooting.md)
