# Search for Files & Patterns

## find - Search for Files

### Basic Syntax

```bash
find [path] [options] [expression]
```

### Search by Name

```bash
# Find by exact name
find /home -name "file.txt"

# Case insensitive
find /home -iname "file.txt"

# Wildcard patterns
find . -name "*.txt"
find . -name "file*"
find . -name "*config*"

# Multiple patterns
find . -name "*.txt" -o -name "*.md"
find . \( -name "*.txt" -o -name "*.md" \)
```

### Search by Type

| Type | Description |
|------|-------------|
| `f` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `s` | Socket |
| `p` | Named pipe |

```bash
# Find only files
find . -type f

# Find only directories
find . -type d

# Find symbolic links
find . -type l

# Combined with name
find . -type f -name "*.log"
find /etc -type d -name "conf*"
```

### Search by Size

```bash
# Exact size
find . -size 100M

# Greater than
find . -size +100M

# Less than
find . -size -100M

# Size units: c (bytes), k (KB), M (MB), G (GB)
find . -size +10k -size -1M     # Between 10KB and 1MB
find . -type f -size 0          # Empty files
```

### Search by Time

```bash
# Modified time (days)
find . -mtime -7      # Modified in last 7 days
find . -mtime +30     # Modified more than 30 days ago
find . -mtime 1       # Modified exactly 1 day ago

# Access time
find . -atime -7      # Accessed in last 7 days

# Change time (metadata)
find . -ctime -7      # Changed in last 7 days

# Minutes instead of days
find . -mmin -60      # Modified in last 60 minutes
find . -amin -30      # Accessed in last 30 minutes

# Newer than reference file
find . -newer reference.txt
```

### Search by Permissions

```bash
# Exact permissions
find . -perm 644
find . -perm 755

# At least these permissions
find . -perm -644

# Any of these permissions
find . -perm /644

# Find SUID files
find / -perm -4000 2>/dev/null

# Find SGID files
find / -perm -2000 2>/dev/null

# Find world-writable files
find / -perm -002 -type f 2>/dev/null
```

### Search by Owner/Group

```bash
# By user
find /home -user john
find . -user root

# By group
find . -group developers

# By UID/GID
find . -uid 1000
find . -gid 1000

# Files with no owner
find . -nouser
find . -nogroup
```

### Search Depth

```bash
# Maximum depth
find . -maxdepth 2 -name "*.txt"

# Minimum depth
find . -mindepth 2 -name "*.txt"

# Only current directory
find . -maxdepth 1 -type f
```

### Actions

```bash
# Delete found files
find . -name "*.tmp" -delete

# Execute command on each file
find . -name "*.txt" -exec cat {} \;

# Execute command with confirmation
find . -name "*.log" -ok rm {} \;

# Execute with multiple files at once
find . -name "*.txt" -exec grep "pattern" {} +

# Print with null separator (for xargs)
find . -name "*.txt" -print0 | xargs -0 grep "pattern"

# Print detailed info
find . -name "*.txt" -ls

# Custom format
find . -name "*.txt" -printf "%p %s %T+\n"
```

### Practical Examples

```bash
# Find and delete old log files
find /var/log -name "*.log" -mtime +30 -delete

# Find large files
find / -type f -size +100M 2>/dev/null

# Find and change permissions
find . -type f -exec chmod 644 {} \;
find . -type d -exec chmod 755 {} \;

# Find and compress
find . -name "*.log" -exec gzip {} \;

# Find empty files and directories
find . -empty

# Find broken symlinks
find . -xtype l

# Find files modified today
find . -daystart -mtime 0

# Exclude directory
find . -path "./node_modules" -prune -o -name "*.js" -print
```

## locate - Fast File Search

Uses a database for faster searches (requires `updatedb`).

```bash
# Basic search
locate filename

# Case insensitive
locate -i filename

# Count matches
locate -c filename

# Limit results
locate -l 10 filename

# Show only existing files
locate -e filename

# Update database (run as root)
sudo updatedb

# Regex search
locate -r ".*\.txt$"
```

## which - Find Command Location

```bash
# Find command path
which ls
# /usr/bin/ls

which python
# /usr/bin/python

# Find all matches
which -a python
```

## whereis - Find Binary, Source, Manual

```bash
whereis ls
# ls: /usr/bin/ls /usr/share/man/man1/ls.1.gz

# Binary only
whereis -b ls

# Manual only
whereis -m ls

# Source only
whereis -s ls
```

## type - Command Type

```bash
type ls
# ls is aliased to 'ls --color=auto'

type cd
# cd is a shell builtin

type grep
# grep is /usr/bin/grep

# Show all matches
type -a echo
```

## Searching File Contents

### grep Recursive Search

```bash
# Search in all files recursively
grep -r "pattern" /path/

# Include specific files
grep -r --include="*.py" "import" /project/

# Exclude directories
grep -r --exclude-dir=".git" "pattern" .

# Exclude files
grep -r --exclude="*.log" "pattern" .

# Binary files as text
grep -a "pattern" binaryfile
```

### Find + Grep Combination

```bash
# Find files and search in them
find . -name "*.py" -exec grep -l "import os" {} \;

# More efficient with xargs
find . -name "*.py" -print0 | xargs -0 grep -l "import os"

# Find and show matching content
find . -name "*.log" -exec grep -H "ERROR" {} \;
```

### ripgrep (rg) - Modern Alternative

```bash
# Fast recursive search
rg "pattern"

# Specific file type
rg "pattern" -t py

# Case insensitive
rg -i "pattern"

# With context
rg -C 3 "pattern"

# Files only
rg -l "pattern"

# Count matches
rg -c "pattern"
```

### ag (The Silver Searcher)

```bash
# Fast recursive search
ag "pattern"

# Specific file type
ag "pattern" --python

# List files only
ag -l "pattern"
```

## Regular Expression Basics

| Pattern | Meaning |
|---------|---------|
| `.` | Any single character |
| `*` | Zero or more of previous |
| `+` | One or more of previous |
| `?` | Zero or one of previous |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Character class |
| `[^abc]` | Negated class |
| `[a-z]` | Range |
| `\d` | Digit (some tools) |
| `\w` | Word character |
| `\s` | Whitespace |
| `\|` | Alternation (OR) |
| `()` | Grouping |
| `{n}` | Exactly n times |
| `{n,m}` | n to m times |

### Examples

```bash
# Lines starting with #
grep "^#" file.txt

# Lines ending with semicolon
grep ";$" file.txt

# Email pattern
grep -E "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt

# IP address
grep -E "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" file.txt

# Phone number
grep -E "\b[0-9]{3}-[0-9]{3}-[0-9]{4}\b" file.txt
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `find` | Search files | `find . -name "*.txt"` |
| `locate` | Fast search (DB) | `locate file.txt` |
| `which` | Command path | `which python` |
| `whereis` | Binary/man/source | `whereis ls` |
| `type` | Command type | `type cd` |
| `grep -r` | Search in files | `grep -r "pattern" .` |
| `rg` | Fast search (ripgrep) | `rg "pattern"` |

---

**Previous:** [10-text-processing.md](10-text-processing.md) | **Next:** [12-file-compression-archival.md](12-file-compression-archival.md)
