# File Compression & Archival

## Overview

| Tool | Extension | Algorithm | Notes |
|------|-----------|-----------|-------|
| gzip | .gz | DEFLATE | Fast, good compression |
| bzip2 | .bz2 | Burrows-Wheeler | Better compression, slower |
| xz | .xz | LZMA2 | Best compression, slowest |
| zip | .zip | DEFLATE | Cross-platform, includes archiving |
| tar | .tar | None | Archiving only, no compression |

## tar - Tape Archive

tar bundles files into a single archive.

### Basic Syntax

```bash
tar [options] archive.tar files...
```

### Common Options

| Option | Description |
|--------|-------------|
| `-c` | Create archive |
| `-x` | Extract archive |
| `-t` | List contents |
| `-v` | Verbose output |
| `-f` | Specify archive file |
| `-z` | gzip compression |
| `-j` | bzip2 compression |
| `-J` | xz compression |
| `-C` | Change directory |
| `-p` | Preserve permissions |

### Create Archives

```bash
# Create tar archive
tar -cvf archive.tar file1 file2 directory/

# Create from directory
tar -cvf archive.tar /path/to/directory/

# Exclude files
tar -cvf archive.tar --exclude="*.log" directory/
tar -cvf archive.tar --exclude-vcs directory/  # Exclude .git, .svn

# Create with gzip compression (.tar.gz or .tgz)
tar -czvf archive.tar.gz directory/

# Create with bzip2 compression (.tar.bz2)
tar -cjvf archive.tar.bz2 directory/

# Create with xz compression (.tar.xz)
tar -cJvf archive.tar.xz directory/
```

### Extract Archives

```bash
# Extract tar
tar -xvf archive.tar

# Extract to specific directory
tar -xvf archive.tar -C /destination/

# Extract gzip compressed
tar -xzvf archive.tar.gz

# Extract bzip2 compressed
tar -xjvf archive.tar.bz2

# Extract xz compressed
tar -xJvf archive.tar.xz

# Extract specific files
tar -xvf archive.tar file1.txt path/to/file2.txt
```

### List Contents

```bash
# List files in archive
tar -tvf archive.tar

# List gzip compressed
tar -tzvf archive.tar.gz

# List with less for long listings
tar -tvf archive.tar | less
```

### Practical Examples

```bash
# Backup home directory
tar -czvf home-backup-$(date +%Y%m%d).tar.gz /home/user/

# Backup with exclusions
tar -czvf backup.tar.gz \
    --exclude="*.tmp" \
    --exclude="node_modules" \
    --exclude=".git" \
    /project/

# Incremental backup
tar -czvf backup.tar.gz --newer-mtime="2024-01-01" /data/

# Append to existing archive (tar only, not compressed)
tar -rvf archive.tar newfile.txt

# Verify archive
tar -tvf archive.tar > /dev/null && echo "OK"
```

## gzip - GNU Zip

### Compress Files

```bash
# Compress file (replaces original)
gzip file.txt
# Creates: file.txt.gz

# Keep original file
gzip -k file.txt

# Compress multiple files
gzip file1.txt file2.txt file3.txt

# Compress with maximum compression
gzip -9 file.txt

# Compress with fastest speed
gzip -1 file.txt

# Recursive compress
gzip -r directory/
```

### Decompress Files

```bash
# Decompress
gzip -d file.txt.gz
gunzip file.txt.gz

# Keep compressed file
gzip -dk file.txt.gz
gunzip -k file.txt.gz

# Decompress to stdout
gzip -dc file.txt.gz
zcat file.txt.gz
```

### View Compressed Files

```bash
# View without decompressing
zcat file.txt.gz
zless file.txt.gz
zgrep "pattern" file.txt.gz
```

## bzip2 - Better Compression

```bash
# Compress
bzip2 file.txt
# Creates: file.txt.bz2

# Keep original
bzip2 -k file.txt

# Maximum compression
bzip2 -9 file.txt

# Decompress
bzip2 -d file.txt.bz2
bunzip2 file.txt.bz2

# View compressed
bzcat file.txt.bz2
bzless file.txt.bz2
bzgrep "pattern" file.txt.bz2
```

## xz - Extreme Compression

```bash
# Compress
xz file.txt
# Creates: file.txt.xz

# Keep original
xz -k file.txt

# Maximum compression
xz -9 file.txt

# Use multiple threads
xz -T0 file.txt

# Decompress
xz -d file.txt.xz
unxz file.txt.xz

# View compressed
xzcat file.txt.xz
xzless file.txt.xz
xzgrep "pattern" file.txt.xz
```

## zip - Cross-Platform Compression

### Create Zip Archives

```bash
# Create zip archive
zip archive.zip file1.txt file2.txt

# Include directory recursively
zip -r archive.zip directory/

# With password
zip -e secure.zip file.txt

# Exclude files
zip -r archive.zip directory/ -x "*.log" -x "*.tmp"

# Update existing archive
zip -u archive.zip newfile.txt

# Split large archives
zip -r -s 100m archive.zip directory/
```

### Extract Zip Archives

```bash
# Extract
unzip archive.zip

# Extract to directory
unzip archive.zip -d /destination/

# Extract specific files
unzip archive.zip file1.txt file2.txt

# List contents
unzip -l archive.zip

# Test archive integrity
unzip -t archive.zip

# Overwrite without prompting
unzip -o archive.zip

# Never overwrite
unzip -n archive.zip
```

## Compression Comparison

```bash
# Create test file
dd if=/dev/urandom of=test.bin bs=1M count=100

# Compare compression
gzip -k test.bin     # Creates test.bin.gz
bzip2 -k test.bin    # Creates test.bin.bz2
xz -k test.bin       # Creates test.bin.xz

# Check sizes
ls -lh test.bin*
```

### Typical Results

| Format | Size | Speed |
|--------|------|-------|
| Original | 100M | - |
| gzip | ~85M | Fast |
| bzip2 | ~80M | Medium |
| xz | ~75M | Slow |

## Common Archive Extensions

| Extension | Format | Command |
|-----------|--------|---------|
| `.tar` | Uncompressed tar | `tar -xvf` |
| `.tar.gz` or `.tgz` | Gzip compressed tar | `tar -xzvf` |
| `.tar.bz2` or `.tbz2` | Bzip2 compressed tar | `tar -xjvf` |
| `.tar.xz` or `.txz` | Xz compressed tar | `tar -xJvf` |
| `.gz` | Gzip compressed file | `gunzip` |
| `.bz2` | Bzip2 compressed file | `bunzip2` |
| `.xz` | Xz compressed file | `unxz` |
| `.zip` | Zip archive | `unzip` |
| `.7z` | 7-Zip archive | `7z x` |
| `.rar` | RAR archive | `unrar x` |

## 7-Zip (Optional)

```bash
# Install
sudo apt install p7zip-full

# Create archive
7z a archive.7z files/

# Extract
7z x archive.7z

# List contents
7z l archive.7z
```

## Practical Backup Scripts

### Simple Backup

```bash
#!/bin/bash
BACKUP_DIR="/backup"
SOURCE_DIR="/home/user"
DATE=$(date +%Y%m%d_%H%M%S)

tar -czvf "$BACKUP_DIR/backup_$DATE.tar.gz" "$SOURCE_DIR"
```

### Backup with Rotation

```bash
#!/bin/bash
BACKUP_DIR="/backup"
KEEP_DAYS=7

# Create backup
tar -czvf "$BACKUP_DIR/backup_$(date +%Y%m%d).tar.gz" /data/

# Delete old backups
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +$KEEP_DAYS -delete
```

## Quick Reference

| Task | Command |
|------|---------|
| Create tar | `tar -cvf archive.tar files/` |
| Create tar.gz | `tar -czvf archive.tar.gz files/` |
| Create tar.bz2 | `tar -cjvf archive.tar.bz2 files/` |
| Create tar.xz | `tar -cJvf archive.tar.xz files/` |
| Extract tar | `tar -xvf archive.tar` |
| Extract tar.gz | `tar -xzvf archive.tar.gz` |
| Extract tar.bz2 | `tar -xjvf archive.tar.bz2` |
| Extract tar.xz | `tar -xJvf archive.tar.xz` |
| List tar contents | `tar -tvf archive.tar` |
| Compress with gzip | `gzip file.txt` |
| Decompress gzip | `gunzip file.txt.gz` |
| Create zip | `zip -r archive.zip files/` |
| Extract zip | `unzip archive.zip` |

---

**Previous:** [11-search-files-patterns.md](11-search-files-patterns.md) | **Next:** [13-process-management.md](13-process-management.md)
