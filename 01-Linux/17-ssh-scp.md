# SSH & SCP

## SSH - Secure Shell

SSH provides secure encrypted connection to remote systems.

### Basic Connection

```bash
# Connect to remote host
ssh user@hostname
ssh user@192.168.1.100

# Connect on different port
ssh -p 2222 user@hostname

# Connect with specific identity
ssh -i ~/.ssh/id_rsa user@hostname

# Verbose mode (debugging)
ssh -v user@hostname
ssh -vvv user@hostname   # More verbose
```

### SSH Configuration

#### Client Config (~/.ssh/config)

```bash
# ~/.ssh/config

# Default settings for all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Specific host configuration
Host myserver
    HostName 192.168.1.100
    User admin
    Port 2222
    IdentityFile ~/.ssh/myserver_key

Host production
    HostName prod.example.com
    User deploy
    IdentityFile ~/.ssh/deploy_key
    ForwardAgent yes

# Wildcard patterns
Host *.example.com
    User admin
    IdentityFile ~/.ssh/example_key
```

Then connect with:
```bash
ssh myserver
ssh production
```

#### Server Config (/etc/ssh/sshd_config)

```bash
# Common security settings
Port 22                          # Change to non-standard port
PermitRootLogin no               # Disable root login
PasswordAuthentication no        # Disable password auth
PubkeyAuthentication yes         # Enable key auth
MaxAuthTries 3                   # Limit auth attempts
AllowUsers user1 user2           # Whitelist users

# Restart after changes
sudo systemctl restart sshd
```

## SSH Key Authentication

### Generate SSH Keys

```bash
# Generate RSA key (default)
ssh-keygen

# Generate with specific type and bits
ssh-keygen -t rsa -b 4096
ssh-keygen -t ed25519          # Recommended, more secure

# Generate with comment
ssh-keygen -t ed25519 -C "user@hostname"

# Specify output file
ssh-keygen -t ed25519 -f ~/.ssh/mykey

# Generate without passphrase (not recommended)
ssh-keygen -t ed25519 -N ""
```

### SSH Key Files

| File | Purpose |
|------|---------|
| `~/.ssh/id_rsa` | Private RSA key |
| `~/.ssh/id_rsa.pub` | Public RSA key |
| `~/.ssh/id_ed25519` | Private Ed25519 key |
| `~/.ssh/id_ed25519.pub` | Public Ed25519 key |
| `~/.ssh/authorized_keys` | Authorized public keys |
| `~/.ssh/known_hosts` | Known host fingerprints |
| `~/.ssh/config` | Client configuration |

### Copy Public Key to Server

```bash
# Using ssh-copy-id (recommended)
ssh-copy-id user@hostname
ssh-copy-id -i ~/.ssh/mykey.pub user@hostname

# Manual method
cat ~/.ssh/id_ed25519.pub | ssh user@hostname "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Or copy and paste manually
cat ~/.ssh/id_ed25519.pub
# Then add to remote ~/.ssh/authorized_keys
```

### Key Permissions

```bash
# Fix permissions (required for SSH to work)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_rsa.pub
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config
```

### SSH Agent

```bash
# Start agent
eval $(ssh-agent)

# Add key to agent
ssh-add
ssh-add ~/.ssh/mykey

# List keys in agent
ssh-add -l

# Remove key from agent
ssh-add -d ~/.ssh/mykey

# Remove all keys
ssh-add -D

# Add key with timeout (seconds)
ssh-add -t 3600 ~/.ssh/mykey
```

## SSH Tunneling

### Local Port Forwarding

Forward local port to remote destination.

```bash
# Forward local:8080 to remote:80
ssh -L 8080:localhost:80 user@remote

# Forward to third host through remote
ssh -L 8080:database.internal:3306 user@jumphost

# Background tunnel
ssh -fN -L 8080:localhost:80 user@remote
```

### Remote Port Forwarding

Make local port accessible on remote.

```bash
# Make local:3000 accessible on remote:8080
ssh -R 8080:localhost:3000 user@remote

# Background tunnel
ssh -fN -R 8080:localhost:3000 user@remote
```

### Dynamic Port Forwarding (SOCKS Proxy)

```bash
# Create SOCKS proxy on local:1080
ssh -D 1080 user@remote

# Background proxy
ssh -fN -D 1080 user@remote

# Use with curl
curl --socks5 localhost:1080 http://example.com
```

### Jump Host (ProxyJump)

```bash
# Connect through jump host
ssh -J jumphost user@destination

# Multiple jumps
ssh -J jump1,jump2 user@destination

# In config file
Host destination
    HostName dest.internal
    User admin
    ProxyJump jumphost
```

## SCP - Secure Copy

### Basic Usage

```bash
# Copy local to remote
scp file.txt user@remote:/path/to/destination/
scp file.txt user@remote:~

# Copy remote to local
scp user@remote:/path/to/file.txt /local/path/
scp user@remote:~/file.txt .

# Copy directory recursively
scp -r directory/ user@remote:/path/

# Preserve permissions and timestamps
scp -p file.txt user@remote:/path/

# Use specific port
scp -P 2222 file.txt user@remote:/path/

# Use specific key
scp -i ~/.ssh/mykey file.txt user@remote:/path/

# Copy between two remote hosts
scp user1@host1:/file.txt user2@host2:/destination/
```

### SCP Options

| Option | Description |
|--------|-------------|
| `-r` | Recursive (directories) |
| `-p` | Preserve times and modes |
| `-P` | Port number |
| `-i` | Identity file |
| `-q` | Quiet mode |
| `-v` | Verbose mode |
| `-C` | Enable compression |
| `-l` | Limit bandwidth (Kbit/s) |

## rsync - Advanced Copy

rsync is more powerful than scp for synchronization.

### Basic Usage

```bash
# Sync local to remote
rsync -av source/ user@remote:/destination/

# Sync remote to local
rsync -av user@remote:/source/ /local/destination/

# Dry run (show what would happen)
rsync -avn source/ user@remote:/destination/

# Delete files on destination not in source
rsync -av --delete source/ user@remote:/destination/

# Exclude files
rsync -av --exclude="*.log" source/ destination/
rsync -av --exclude-from=exclude.txt source/ destination/

# Show progress
rsync -av --progress source/ destination/

# Use specific SSH port
rsync -av -e "ssh -p 2222" source/ user@remote:/dest/

# Compress during transfer
rsync -avz source/ user@remote:/destination/
```

### rsync Options

| Option | Description |
|--------|-------------|
| `-a` | Archive mode (recursive, preserves everything) |
| `-v` | Verbose |
| `-z` | Compress during transfer |
| `-P` | Progress + partial (resume) |
| `--delete` | Delete extraneous files from destination |
| `--exclude` | Exclude pattern |
| `--dry-run` or `-n` | Show what would be transferred |
| `-e` | Specify remote shell |

### Common rsync Patterns

```bash
# Backup with date
rsync -av /source/ /backup/$(date +%Y%m%d)/

# Mirror with delete
rsync -av --delete /source/ /mirror/

# Sync only newer files
rsync -av --update /source/ /destination/

# Bandwidth limit (KB/s)
rsync -av --bwlimit=1000 /source/ /destination/
```

## SFTP - Secure FTP

```bash
# Connect
sftp user@remote

# SFTP commands
sftp> ls                  # List remote
sftp> lls                 # List local
sftp> cd /path            # Change remote dir
sftp> lcd /path           # Change local dir
sftp> get file.txt        # Download
sftp> put file.txt        # Upload
sftp> get -r directory/   # Download directory
sftp> put -r directory/   # Upload directory
sftp> rm file.txt         # Delete remote
sftp> mkdir newdir        # Create directory
sftp> pwd                 # Print remote directory
sftp> lpwd                # Print local directory
sftp> exit                # Quit
```

## SSH Security Best Practices

1. **Use SSH keys** instead of passwords
2. **Use Ed25519** or RSA 4096-bit keys
3. **Protect private keys** with passphrase
4. **Disable root login** (`PermitRootLogin no`)
5. **Disable password auth** (`PasswordAuthentication no`)
6. **Use non-standard port** (reduces automated attacks)
7. **Limit users** (`AllowUsers`, `AllowGroups`)
8. **Use fail2ban** to block brute force
9. **Keep SSH updated**
10. **Use SSH agent** for key management

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `ssh` | Connect to remote | `ssh user@host` |
| `ssh-keygen` | Generate keys | `ssh-keygen -t ed25519` |
| `ssh-copy-id` | Copy public key | `ssh-copy-id user@host` |
| `ssh-add` | Add key to agent | `ssh-add ~/.ssh/key` |
| `scp` | Secure copy | `scp file user@host:/path` |
| `rsync` | Sync files | `rsync -av src/ dest/` |
| `sftp` | Secure FTP | `sftp user@host` |
| `ssh -L` | Local tunnel | `ssh -L 8080:localhost:80 host` |
| `ssh -R` | Remote tunnel | `ssh -R 8080:localhost:80 host` |
| `ssh -D` | SOCKS proxy | `ssh -D 1080 host` |
| `ssh -J` | Jump host | `ssh -J jump user@dest` |

---

**Previous:** [16-networking-commands.md](16-networking-commands.md) | **Next:** [18-storage.md](18-storage.md)
