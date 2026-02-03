# File Permissions

## Understanding Permissions

Every file and directory in Linux has three types of permissions for three categories of users.

### Permission Types

| Symbol | Permission | For Files | For Directories |
|--------|------------|-----------|-----------------|
| `r` | Read | View contents | List contents |
| `w` | Write | Modify contents | Create/delete files |
| `x` | Execute | Run as program | Enter directory |
| `-` | None | No permission | No permission |

### User Categories

| Symbol | Category | Description |
|--------|----------|-------------|
| `u` | User | Owner of the file |
| `g` | Group | Group the file belongs to |
| `o` | Others | Everyone else |
| `a` | All | All of the above |

## Reading Permissions

```bash
ls -l file.txt
# -rw-r--r-- 1 user group 1234 Jan 1 12:00 file.txt
```

```
-rw-r--r--
│├─┤├─┤├─┤
│ │  │  └── Others: r-- (read only)
│ │  └───── Group:  r-- (read only)
│ └──────── User:   rw- (read and write)
└────────── File type: - (regular file)
```

### File Type Indicators

| Symbol | Type |
|--------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `s` | Socket |
| `p` | Named pipe |

## Numeric (Octal) Permissions

Each permission has a numeric value:

| Permission | Value |
|------------|-------|
| Read (r) | 4 |
| Write (w) | 2 |
| Execute (x) | 1 |
| None (-) | 0 |

### Calculation

```
rwx = 4 + 2 + 1 = 7
rw- = 4 + 2 + 0 = 6
r-x = 4 + 0 + 1 = 5
r-- = 4 + 0 + 0 = 4
-wx = 0 + 2 + 1 = 3
-w- = 0 + 2 + 0 = 2
--x = 0 + 0 + 1 = 1
--- = 0 + 0 + 0 = 0
```

### Common Permission Combinations

| Numeric | Symbolic | Meaning |
|---------|----------|---------|
| 777 | rwxrwxrwx | Full access for all |
| 755 | rwxr-xr-x | Owner full, others read/execute |
| 750 | rwxr-x--- | Owner full, group read/execute |
| 700 | rwx------ | Owner full, others none |
| 666 | rw-rw-rw- | Read/write for all |
| 644 | rw-r--r-- | Owner read/write, others read |
| 640 | rw-r----- | Owner read/write, group read |
| 600 | rw------- | Owner read/write only |
| 555 | r-xr-xr-x | Read/execute for all |
| 444 | r--r--r-- | Read only for all |

## chmod - Change Permissions

### Symbolic Mode

```bash
# Add permission
chmod u+x file.txt      # Add execute for user
chmod g+w file.txt      # Add write for group
chmod o+r file.txt      # Add read for others
chmod a+x file.txt      # Add execute for all

# Remove permission
chmod u-w file.txt      # Remove write from user
chmod g-x file.txt      # Remove execute from group
chmod o-r file.txt      # Remove read from others

# Set exact permission
chmod u=rwx file.txt    # User: read, write, execute
chmod g=rx file.txt     # Group: read, execute
chmod o= file.txt       # Others: no permissions

# Multiple changes
chmod u+x,g-w,o-r file.txt
chmod ug+rw,o-rwx file.txt

# Copy permissions
chmod --reference=source.txt target.txt
```

### Numeric Mode

```bash
chmod 755 file.txt      # rwxr-xr-x
chmod 644 file.txt      # rw-r--r--
chmod 700 file.txt      # rwx------
chmod 600 file.txt      # rw-------

# Recursive (for directories)
chmod -R 755 directory/
```

### Common Use Cases

```bash
# Make script executable
chmod +x script.sh
chmod 755 script.sh

# Protect private key
chmod 600 ~/.ssh/id_rsa

# Web directory
chmod 755 /var/www/html

# Config file (owner only)
chmod 600 config.ini
```

## chown - Change Ownership

```bash
# Change owner
chown user file.txt

# Change owner and group
chown user:group file.txt

# Change group only
chown :group file.txt

# Recursive
chown -R user:group directory/

# Using user ID
chown 1000:1000 file.txt
```

## chgrp - Change Group

```bash
# Change group
chgrp group file.txt

# Recursive
chgrp -R group directory/
```

## Special Permissions

### SUID (Set User ID)

When set on executable, runs with owner's permissions.

```bash
# Check for SUID (s in user execute position)
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd

# Set SUID
chmod u+s file
chmod 4755 file

# Remove SUID
chmod u-s file
```

### SGID (Set Group ID)

- On files: runs with group's permissions
- On directories: new files inherit directory's group

```bash
# Check for SGID (s in group execute position)
ls -l directory/
# drwxr-sr-x 2 user group ... directory/

# Set SGID
chmod g+s directory/
chmod 2755 directory/

# Remove SGID
chmod g-s directory/
```

### Sticky Bit

On directories: only owner can delete their files (used on /tmp).

```bash
# Check for sticky bit (t in others execute position)
ls -ld /tmp
# drwxrwxrwt 10 root root ... /tmp

# Set sticky bit
chmod +t directory/
chmod 1755 directory/

# Remove sticky bit
chmod -t directory/
```

### Special Permission Values

| Value | Permission |
|-------|------------|
| 4000 | SUID |
| 2000 | SGID |
| 1000 | Sticky bit |

```bash
# SUID + standard permissions
chmod 4755 file     # rwsr-xr-x

# SGID + standard permissions
chmod 2755 directory/ # rwxr-sr-x

# Sticky + standard permissions
chmod 1755 directory/ # rwxr-xr-t

# SUID + SGID
chmod 6755 file     # rwsr-sr-x
```

## umask - Default Permissions

umask defines the default permissions for new files and directories.

```bash
# View current umask
umask

# View in symbolic form
umask -S

# Common umask values
umask 022   # Files: 644, Dirs: 755
umask 027   # Files: 640, Dirs: 750
umask 077   # Files: 600, Dirs: 700
```

### How umask Works

```
Default file permission:    666 (rw-rw-rw-)
Default directory permission: 777 (rwxrwxrwx)

Subtract umask from default:

File:      666 - 022 = 644 (rw-r--r--)
Directory: 777 - 022 = 755 (rwxr-xr-x)
```

### Set Permanent umask

```bash
# Add to ~/.bashrc or ~/.profile
echo "umask 022" >> ~/.bashrc

# System-wide: /etc/profile or /etc/login.defs
```

## Viewing Permissions

```bash
# Long listing
ls -l file.txt

# Directory itself
ls -ld directory/

# Numeric permissions
stat -c "%a %n" file.txt

# All details
stat file.txt
```

## Access Control Lists (ACL)

For more granular permissions beyond user/group/others.

```bash
# View ACL
getfacl file.txt

# Set ACL for user
setfacl -m u:username:rwx file.txt

# Set ACL for group
setfacl -m g:groupname:rx file.txt

# Remove ACL entry
setfacl -x u:username file.txt

# Remove all ACL
setfacl -b file.txt

# Recursive
setfacl -R -m u:username:rwx directory/

# Default ACL (inherited by new files)
setfacl -d -m u:username:rwx directory/
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `chmod` | Change permissions | `chmod 755 file` |
| `chown` | Change owner | `chown user:group file` |
| `chgrp` | Change group | `chgrp group file` |
| `umask` | Set default permissions | `umask 022` |
| `getfacl` | View ACL | `getfacl file` |
| `setfacl` | Set ACL | `setfacl -m u:user:rwx file` |

---

**Previous:** [06-vi-editor.md](06-vi-editor.md) | **Next:** [08-users-groups.md](08-users-groups.md)
