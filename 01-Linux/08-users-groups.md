# Users & Groups

## User Management Overview

Linux is a multi-user system. Each user has:
- Unique username and UID (User ID)
- Home directory
- Default shell
- Primary group and optional secondary groups

## Important Files

| File | Purpose |
|------|---------|
| `/etc/passwd` | User account information |
| `/etc/shadow` | Encrypted passwords |
| `/etc/group` | Group information |
| `/etc/gshadow` | Secure group information |
| `/etc/skel` | Skeleton directory for new users |
| `/etc/login.defs` | Login defaults |

## Understanding /etc/passwd

```bash
cat /etc/passwd
# username:x:UID:GID:comment:home:shell
# root:x:0:0:root:/root:/bin/bash
```

| Field | Description |
|-------|-------------|
| username | Login name |
| x | Password placeholder (actual in /etc/shadow) |
| UID | User ID number |
| GID | Primary group ID |
| comment | Full name or description (GECOS) |
| home | Home directory path |
| shell | Default login shell |

### UID Ranges

| Range | Type |
|-------|------|
| 0 | Root user |
| 1-999 | System users (services) |
| 1000+ | Regular users |

## Understanding /etc/shadow

```bash
sudo cat /etc/shadow
# username:$6$hash:18000:0:99999:7:::
```

| Field | Description |
|-------|-------------|
| 1 | Username |
| 2 | Encrypted password |
| 3 | Days since password last changed (from Jan 1, 1970) |
| 4 | Minimum days between password changes |
| 5 | Maximum days password is valid |
| 6 | Days before expiry to warn |
| 7 | Days after expiry until account disabled |
| 8 | Account expiration date |

### Password Field Values

| Value | Meaning |
|-------|---------|
| `$6$...` | SHA-512 encrypted password |
| `$5$...` | SHA-256 encrypted password |
| `$1$...` | MD5 encrypted password |
| `!` or `*` | Account locked/no password |
| `!!` | Password never set |

## User Commands

### useradd - Create User

```bash
# Create user with defaults
sudo useradd username

# Create with home directory
sudo useradd -m username

# Specify home directory
sudo useradd -m -d /home/customdir username

# Specify shell
sudo useradd -m -s /bin/bash username

# Specify UID
sudo useradd -u 1500 username

# Specify primary group
sudo useradd -g groupname username

# Specify secondary groups
sudo useradd -G group1,group2 username

# Add comment
sudo useradd -c "John Doe" username

# Create system user (no home, no login)
sudo useradd -r -s /usr/sbin/nologin serviceuser

# Full example
sudo useradd -m -s /bin/bash -c "John Doe" -G sudo,docker johnd

# Set password after creation
sudo passwd username
```

### adduser - Interactive User Creation (Debian/Ubuntu)

```bash
# Interactive user creation
sudo adduser username
# Prompts for password, full name, etc.
```

### usermod - Modify User

```bash
# Change username
sudo usermod -l newname oldname

# Change home directory
sudo usermod -d /new/home -m username

# Change shell
sudo usermod -s /bin/zsh username

# Change primary group
sudo usermod -g newgroup username

# Add to secondary groups (append)
sudo usermod -aG group1,group2 username

# Add user to sudo group
sudo usermod -aG sudo username      # Debian/Ubuntu
sudo usermod -aG wheel username     # RHEL/CentOS

# Lock user account
sudo usermod -L username

# Unlock user account
sudo usermod -U username

# Set account expiry date
sudo usermod -e 2024-12-31 username

# Change UID
sudo usermod -u 1500 username

# Change comment
sudo usermod -c "Jane Doe" username
```

### userdel - Delete User

```bash
# Delete user (keep home directory)
sudo userdel username

# Delete user and home directory
sudo userdel -r username

# Force delete (even if logged in)
sudo userdel -f username
```

### passwd - Manage Passwords

```bash
# Change your own password
passwd

# Change another user's password (as root)
sudo passwd username

# Lock password
sudo passwd -l username

# Unlock password
sudo passwd -u username

# Delete password (passwordless)
sudo passwd -d username

# Expire password (force change on next login)
sudo passwd -e username

# Set password aging
sudo passwd -n 7 -x 90 -w 7 username
# -n: minimum days between changes
# -x: maximum days valid
# -w: warning days before expiry

# Check password status
sudo passwd -S username
```

## Group Commands

### Understanding /etc/group

```bash
cat /etc/group
# groupname:x:GID:member1,member2
# docker:x:999:john,jane
```

| Field | Description |
|-------|-------------|
| groupname | Group name |
| x | Password placeholder |
| GID | Group ID |
| members | Comma-separated list of users |

### groupadd - Create Group

```bash
# Create group
sudo groupadd groupname

# Create with specific GID
sudo groupadd -g 1500 groupname

# Create system group
sudo groupadd -r groupname
```

### groupmod - Modify Group

```bash
# Rename group
sudo groupmod -n newname oldname

# Change GID
sudo groupmod -g 1500 groupname
```

### groupdel - Delete Group

```bash
sudo groupdel groupname
```

### gpasswd - Group Administration

```bash
# Add user to group
sudo gpasswd -a username groupname

# Remove user from group
sudo gpasswd -d username groupname

# Set group administrators
sudo gpasswd -A admin1,admin2 groupname

# Set group password
sudo gpasswd groupname
```

## Viewing User Information

```bash
# Current user
whoami

# Current user ID and groups
id

# Specific user info
id username

# List groups for user
groups username

# Who is logged in
who
w

# Last logins
last
last username

# Failed login attempts
sudo lastb

# User information
finger username      # May need to install
getent passwd username
```

## Switching Users

```bash
# Switch to another user
su - username

# Switch to root
su -
sudo -i

# Execute command as another user
sudo -u username command

# Execute command as root
sudo command
```

## Sudo Configuration

### /etc/sudoers

```bash
# Edit sudoers (always use visudo!)
sudo visudo

# User privilege specification
# user    host=(runas)    commands
root      ALL=(ALL:ALL)   ALL
john      ALL=(ALL)       ALL
jane      ALL=(ALL)       NOPASSWD: ALL

# Group privilege (prefix with %)
%sudo     ALL=(ALL:ALL)   ALL
%admin    ALL=(ALL)       ALL

# Allow specific commands
john      ALL=(ALL)       /usr/bin/apt, /usr/bin/systemctl

# Allow with no password
john      ALL=(ALL)       NOPASSWD: /usr/bin/systemctl restart nginx
```

### Sudoers Drop-in Directory

```bash
# Create custom rules in /etc/sudoers.d/
sudo visudo -f /etc/sudoers.d/myconfig
```

## Password Aging (chage)

```bash
# View password aging info
sudo chage -l username

# Set password expiry date
sudo chage -E 2024-12-31 username

# Set maximum days
sudo chage -M 90 username

# Set minimum days
sudo chage -m 7 username

# Set warning days
sudo chage -W 14 username

# Force password change on next login
sudo chage -d 0 username

# Interactive mode
sudo chage username
```

## Skeleton Directory

Files in `/etc/skel` are copied to new user home directories:

```bash
ls -la /etc/skel
# .bash_logout
# .bashrc
# .profile

# Add custom files for new users
sudo cp myconfig /etc/skel/
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `useradd` | Create user | `useradd -m user` |
| `usermod` | Modify user | `usermod -aG sudo user` |
| `userdel` | Delete user | `userdel -r user` |
| `passwd` | Manage password | `passwd user` |
| `groupadd` | Create group | `groupadd mygroup` |
| `groupmod` | Modify group | `groupmod -n new old` |
| `groupdel` | Delete group | `groupdel mygroup` |
| `id` | Show user/group IDs | `id user` |
| `groups` | Show user's groups | `groups user` |
| `chage` | Password aging | `chage -l user` |
| `su` | Switch user | `su - user` |
| `sudo` | Run as root | `sudo command` |

---

**Previous:** [07-file-permissions.md](07-file-permissions.md) | **Next:** [09-io-redirection.md](09-io-redirection.md)
