# File Types & Hierarchy

## Linux File Types

Linux has 7 types of files, identified by the first character in `ls -l` output.

| Symbol | Type | Description | Example |
|--------|------|-------------|---------|
| `-` | Regular file | Text, binary, scripts | `/etc/passwd` |
| `d` | Directory | Container for files | `/home` |
| `l` | Symbolic link | Shortcut/pointer to another file | `/lib` → `/usr/lib` |
| `c` | Character device | Stream devices (keyboard, terminal) | `/dev/tty` |
| `b` | Block device | Block storage devices (disk, USB) | `/dev/sda` |
| `s` | Socket | Inter-process communication | `/var/run/docker.sock` |
| `p` | Named pipe (FIFO) | Inter-process communication | Created with `mkfifo` |

### Identify File Types

```bash
# Using ls -l (first character)
ls -l /etc/passwd
# -rw-r--r-- 1 root root 2847 Jan 1 12:00 /etc/passwd
# ^ regular file

ls -l /dev/sda
# brw-rw---- 1 root disk 8, 0 Jan 1 12:00 /dev/sda
# ^ block device

# Using file command
file /etc/passwd
# /etc/passwd: ASCII text

file /bin/ls
# /bin/ls: ELF 64-bit LSB shared object, x86-64

file /dev/sda
# /dev/sda: block special

# Using stat
stat /etc/passwd
```

### Creating Different File Types

```bash
# Regular file
touch myfile.txt

# Directory
mkdir mydir

# Symbolic link
ln -s /path/to/original linkname

# Hard link
ln /path/to/original linkname

# Named pipe
mkfifo mypipe

# Socket (created by applications)
```

## Symbolic Links vs Hard Links

| Feature | Symbolic Link | Hard Link |
|---------|---------------|-----------|
| Points to | Filename/path | Inode |
| Cross filesystem | Yes | No |
| Link to directory | Yes | No (usually) |
| Original deleted | Broken link | Still works |
| Size | Small (path length) | Same as original |

```bash
# Create symbolic link
ln -s /original/file /path/to/symlink

# Create hard link
ln /original/file /path/to/hardlink

# Find all hard links to a file
find / -samefile /path/to/file

# Check inode
ls -li file
```

## Filesystem Hierarchy Standard (FHS)

```
/                   # Root directory
├── bin/            # Essential user binaries (ls, cp, mv)
├── boot/           # Boot loader files, kernel
├── dev/            # Device files
├── etc/            # System configuration files
├── home/           # User home directories
│   └── username/   # Individual user directory
├── lib/            # Essential shared libraries
├── lib64/          # 64-bit libraries
├── media/          # Mount point for removable media
├── mnt/            # Temporary mount points
├── opt/            # Optional/third-party software
├── proc/           # Virtual filesystem (process info)
├── root/           # Root user's home directory
├── run/            # Runtime data (PIDs, sockets)
├── sbin/           # System binaries (admin commands)
├── srv/            # Service data (web, FTP)
├── sys/            # Virtual filesystem (kernel/hardware)
├── tmp/            # Temporary files (cleared on reboot)
├── usr/            # User programs and data
│   ├── bin/        # User binaries
│   ├── lib/        # Libraries
│   ├── local/      # Locally installed software
│   ├── sbin/       # System binaries
│   └── share/      # Shared data (docs, icons)
└── var/            # Variable data
    ├── log/        # Log files
    ├── cache/      # Application cache
    ├── spool/      # Print/mail queues
    └── tmp/        # Temporary files (persistent)
```

## Important Directories Explained

### /etc - Configuration Files

```bash
/etc/
├── passwd          # User accounts
├── shadow          # Encrypted passwords
├── group           # Group definitions
├── hosts           # Static hostname resolution
├── hostname        # System hostname
├── fstab           # Filesystem mount table
├── sudoers         # Sudo configuration
├── ssh/            # SSH configuration
├── crontab         # System cron jobs
├── systemd/        # systemd configuration
└── apt/ or yum.repos.d/  # Package manager config
```

### /var - Variable Data

```bash
/var/
├── log/            # System and application logs
│   ├── syslog      # General system log
│   ├── auth.log    # Authentication log
│   └── messages    # General messages
├── cache/          # Application cache
├── lib/            # State information (databases)
├── spool/          # Queue directories
│   ├── mail/       # User mailboxes
│   └── cron/       # Cron job spools
└── www/            # Web server files
```

### /proc - Virtual Filesystem

```bash
/proc/
├── cpuinfo         # CPU information
├── meminfo         # Memory information
├── version         # Kernel version
├── uptime          # System uptime
├── loadavg         # Load average
├── mounts          # Mounted filesystems
├── [PID]/          # Process directories
│   ├── cmdline     # Command line
│   ├── status      # Process status
│   ├── fd/         # File descriptors
│   └── environ     # Environment variables
└── sys/            # Kernel parameters
```

### /dev - Device Files

```bash
/dev/
├── sda             # First SCSI/SATA disk
│   ├── sda1        # First partition
│   └── sda2        # Second partition
├── nvme0n1         # NVMe disk
├── tty*            # Terminal devices
├── null            # Null device (discard output)
├── zero            # Zero device (produces zeros)
├── random          # Random number generator
├── urandom         # Non-blocking random
└── loop*           # Loop devices
```

## Special Devices

```bash
# /dev/null - Discard output
command > /dev/null 2>&1

# /dev/zero - Create empty file of specific size
dd if=/dev/zero of=file.img bs=1M count=100

# /dev/random - Generate random data
head -c 32 /dev/urandom | base64
```

## Filesystem Navigation

```bash
# Current directory
pwd

# Home directory
cd ~
cd $HOME
cd

# Previous directory
cd -

# Parent directory
cd ..

# Absolute path (starts with /)
cd /var/log

# Relative path
cd ../etc
```

## Quick Reference

| Path | Purpose |
|------|---------|
| `/` | Root of everything |
| `/home` | User data |
| `/etc` | Configuration |
| `/var/log` | Logs |
| `/tmp` | Temporary files |
| `/opt` | Third-party software |
| `/usr/local` | Locally installed |
| `/boot` | Kernel and bootloader |

---

**Previous:** [03-boot-sequence-runlevels.md](03-boot-sequence-runlevels.md) | **Next:** [05-basic-commands.md](05-basic-commands.md)
