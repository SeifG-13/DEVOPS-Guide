# File Operations in Shell Scripts

## Checking Files and Directories

### File Test Operators

```bash
#!/bin/bash

file="/path/to/file"

# Existence checks
[ -e "$file" ]    # File exists (any type)
[ -f "$file" ]    # Regular file
[ -d "$file" ]    # Directory
[ -L "$file" ]    # Symbolic link
[ -b "$file" ]    # Block device
[ -c "$file" ]    # Character device
[ -p "$file" ]    # Named pipe
[ -S "$file" ]    # Socket

# Permission checks
[ -r "$file" ]    # Readable
[ -w "$file" ]    # Writable
[ -x "$file" ]    # Executable

# Size checks
[ -s "$file" ]    # Size > 0
[ ! -s "$file" ]  # Empty file

# Ownership checks
[ -O "$file" ]    # Owned by current user
[ -G "$file" ]    # Owned by current group
```

### Example: File Checker

```bash
#!/bin/bash

check_file() {
    local file="$1"

    echo "Checking: $file"
    echo "========================"

    if [ ! -e "$file" ]; then
        echo "File does not exist!"
        return 1
    fi

    # Type
    if [ -f "$file" ]; then
        echo "Type: Regular file"
    elif [ -d "$file" ]; then
        echo "Type: Directory"
    elif [ -L "$file" ]; then
        echo "Type: Symbolic link -> $(readlink "$file")"
    fi

    # Permissions
    echo -n "Permissions: "
    [ -r "$file" ] && echo -n "r" || echo -n "-"
    [ -w "$file" ] && echo -n "w" || echo -n "-"
    [ -x "$file" ] && echo -n "x" || echo -n "-"
    echo

    # Size
    if [ -f "$file" ]; then
        size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file" 2>/dev/null)
        echo "Size: $size bytes"
    fi
}

check_file "$1"
```

## Reading Files

### Read Entire File

```bash
#!/bin/bash

# Method 1: cat command
content=$(cat file.txt)

# Method 2: Read into variable
content=$(<file.txt)

# Method 3: Using read with while loop (preferred for large files)
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

### Read Line by Line

```bash
#!/bin/bash

# Standard way
while IFS= read -r line; do
    echo "Line: $line"
done < "input.txt"

# Handle file without newline at end
while IFS= read -r line || [ -n "$line" ]; do
    echo "Line: $line"
done < "input.txt"

# Read with line numbers
linenum=0
while IFS= read -r line; do
    ((linenum++))
    echo "$linenum: $line"
done < "input.txt"
```

### Read Specific Fields

```bash
#!/bin/bash

# Read CSV fields
while IFS=',' read -r name age city; do
    echo "Name: $name, Age: $age, City: $city"
done < "data.csv"

# Read /etc/passwd
while IFS=':' read -r user pass uid gid desc home shell; do
    echo "User: $user, Home: $home"
done < /etc/passwd

# Read with specific columns
while read -r col1 col2 rest; do
    echo "Col1: $col1, Col2: $col2"
done < "data.txt"
```

### Read from Command Output

```bash
#!/bin/bash

# Using pipe (runs in subshell - variables not preserved)
ls -la | while read -r line; do
    echo "$line"
done

# Using process substitution (preserves variables)
while read -r line; do
    echo "$line"
done < <(ls -la)

# Read from here-string
while read -r word; do
    echo "Word: $word"
done <<< "one two three"
```

## Writing Files

### Overwrite File

```bash
#!/bin/bash

# Using echo
echo "Hello, World!" > output.txt

# Using printf
printf "Line 1\nLine 2\n" > output.txt

# Multiple lines with echo
echo "Line 1" > output.txt
echo "Line 2" >> output.txt  # Append

# Using cat with heredoc
cat > output.txt << EOF
Line 1
Line 2
Line 3
EOF

# Prevent variable expansion in heredoc
cat > output.txt << 'EOF'
$HOME will not be expanded
EOF
```

### Append to File

```bash
#!/bin/bash

# Append single line
echo "New line" >> output.txt

# Append multiple lines
cat >> output.txt << EOF
Appended line 1
Appended line 2
EOF

# Append with date
echo "$(date): Log entry" >> log.txt
```

### Write with Permissions Check

```bash
#!/bin/bash

write_safe() {
    local file="$1"
    local content="$2"

    # Check if directory exists
    local dir=$(dirname "$file")
    if [ ! -d "$dir" ]; then
        echo "Error: Directory $dir does not exist"
        return 1
    fi

    # Check write permission
    if [ -e "$file" ] && [ ! -w "$file" ]; then
        echo "Error: No write permission for $file"
        return 1
    fi

    echo "$content" > "$file"
    return 0
}

write_safe "/tmp/test.txt" "Hello, World!"
```

## File Descriptors

### Standard Descriptors

| FD | Name | Default |
|----|------|---------|
| 0 | stdin | Keyboard |
| 1 | stdout | Screen |
| 2 | stderr | Screen |

### Custom File Descriptors

```bash
#!/bin/bash

# Open file for reading (FD 3)
exec 3< input.txt
while read -r line <&3; do
    echo "$line"
done
exec 3<&-    # Close FD 3

# Open file for writing (FD 4)
exec 4> output.txt
echo "Line 1" >&4
echo "Line 2" >&4
exec 4>&-    # Close FD 4

# Open file for read/write (FD 5)
exec 5<> file.txt
read -r line <&5
echo "New content" >&5
exec 5>&-    # Close FD 5
```

### Redirecting Standard Streams

```bash
#!/bin/bash

# Redirect stdout to file
ls > output.txt

# Redirect stderr to file
ls /nonexistent 2> error.txt

# Redirect both stdout and stderr
ls /tmp /nonexistent > output.txt 2>&1
ls /tmp /nonexistent &> all.txt        # Bash shorthand

# Suppress output
ls /nonexistent 2>/dev/null

# Append stdout and stderr
ls /tmp /nonexistent >> output.txt 2>&1
```

### Swap stdout and stderr

```bash
#!/bin/bash

# Swap stdout and stderr
some_command 3>&1 1>&2 2>&3
```

## Here Documents (Heredoc)

### Basic Heredoc

```bash
#!/bin/bash

# Basic heredoc
cat << EOF
This is line 1
This is line 2
Variable expansion: $HOME
EOF

# Suppress expansion (single quotes around EOF)
cat << 'EOF'
This $HOME will NOT be expanded
EOF

# Heredoc to file
cat > output.txt << EOF
File content here
With multiple lines
EOF
```

### Indented Heredoc

```bash
#!/bin/bash

# Tab-indented heredoc (use <<-)
function example() {
    cat <<- EOF
		This line starts with tab
		So does this one
		Tabs will be removed
	EOF
}

example
```

### Heredoc with Commands

```bash
#!/bin/bash

# SSH with heredoc
ssh user@host << 'EOF'
    cd /var/log
    ls -la
    tail -n 10 syslog
EOF

# MySQL with heredoc
mysql -u user -p database << EOF
SELECT * FROM users;
UPDATE settings SET value='new' WHERE key='config';
EOF
```

## Here Strings

```bash
#!/bin/bash

# Here string (single line input)
read -r first second third <<< "one two three"
echo "First: $first"

# With variable
data="apple banana cherry"
read -r a b c <<< "$data"

# Process here string
while read -r word; do
    echo "Word: $word"
done <<< "word1 word2 word3"
```

## File Locking

### Using flock

```bash
#!/bin/bash

LOCKFILE="/tmp/myscript.lock"

# Method 1: Using flock with FD
exec 200>"$LOCKFILE"
flock -n 200 || { echo "Script already running"; exit 1; }

# Script continues...
echo "Running..."
sleep 10

# Lock released when script exits

# Method 2: Subshell with flock
(
    flock -n 9 || { echo "Already running"; exit 1; }

    # Script code here
    echo "Running..."
    sleep 10

) 9>"$LOCKFILE"
```

### Using mkdir for Locking

```bash
#!/bin/bash

LOCKDIR="/tmp/myscript.lock"

# Acquire lock
acquire_lock() {
    if mkdir "$LOCKDIR" 2>/dev/null; then
        trap 'rm -rf "$LOCKDIR"' EXIT
        return 0
    else
        return 1
    fi
}

# Try to acquire lock
if ! acquire_lock; then
    echo "Script already running"
    exit 1
fi

# Script continues...
echo "Running..."
```

## Temporary Files

### Creating Temp Files

```bash
#!/bin/bash

# Using mktemp
tmpfile=$(mktemp)
echo "Temp file: $tmpfile"

# With template
tmpfile=$(mktemp /tmp/myapp.XXXXXX)

# Create temp directory
tmpdir=$(mktemp -d)
echo "Temp dir: $tmpdir"

# Cleanup on exit
trap 'rm -rf "$tmpfile" "$tmpdir"' EXIT
```

### Safe Temp File Pattern

```bash
#!/bin/bash

# Create temp file safely
TMPFILE=""

cleanup() {
    [ -n "$TMPFILE" ] && rm -f "$TMPFILE"
}

trap cleanup EXIT

TMPFILE=$(mktemp) || { echo "Failed to create temp file"; exit 1; }

# Use the temp file
echo "Working with $TMPFILE"
echo "Some data" > "$TMPFILE"
```

## Practical Examples

### File Backup Script

```bash
#!/bin/bash

backup_file() {
    local file="$1"
    local backup_dir="${2:-./backups}"

    if [ ! -f "$file" ]; then
        echo "Error: File not found: $file"
        return 1
    fi

    # Create backup directory
    mkdir -p "$backup_dir"

    # Create backup with timestamp
    local filename=$(basename "$file")
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_name="${filename}.${timestamp}.bak"

    cp "$file" "$backup_dir/$backup_name"
    echo "Backup created: $backup_dir/$backup_name"
}

backup_file "$1" "$2"
```

### Configuration File Parser

```bash
#!/bin/bash

parse_config() {
    local config_file="$1"

    if [ ! -f "$config_file" ]; then
        echo "Config file not found: $config_file"
        return 1
    fi

    while IFS='=' read -r key value; do
        # Skip comments and empty lines
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [[ -z "$key" ]] && continue

        # Trim whitespace
        key=$(echo "$key" | xargs)
        value=$(echo "$value" | xargs)

        # Remove quotes
        value="${value%\"}"
        value="${value#\"}"

        # Export as variable
        export "$key=$value"
        echo "Loaded: $key=$value"
    done < "$config_file"
}

parse_config "app.conf"
```

### Log File Writer

```bash
#!/bin/bash

LOGFILE="/var/log/myapp.log"

log() {
    local level="$1"
    shift
    local message="$*"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    echo "[$timestamp] [$level] $message" >> "$LOGFILE"

    # Also print to stdout for INFO and above
    case "$level" in
        INFO|WARN|ERROR)
            echo "[$level] $message"
            ;;
    esac
}

log_info()  { log "INFO" "$@"; }
log_warn()  { log "WARN" "$@"; }
log_error() { log "ERROR" "$@"; }
log_debug() { log "DEBUG" "$@"; }

# Usage
log_info "Application started"
log_warn "Low disk space"
log_error "Connection failed"
```

### File Watcher

```bash
#!/bin/bash

watch_file() {
    local file="$1"
    local last_mod=""

    while true; do
        if [ -f "$file" ]; then
            current_mod=$(stat -c%Y "$file" 2>/dev/null || stat -f%m "$file" 2>/dev/null)

            if [ "$current_mod" != "$last_mod" ]; then
                echo "$(date): File changed: $file"
                last_mod="$current_mod"
                # Add your action here
            fi
        fi

        sleep 1
    done
}

watch_file "$1"
```

## Quick Reference

| Operation | Command |
|-----------|---------|
| Read file | `content=$(<file)` |
| Read line by line | `while IFS= read -r line; do ... done < file` |
| Write to file | `echo "text" > file` |
| Append to file | `echo "text" >> file` |
| Heredoc | `cat << EOF ... EOF` |
| Here string | `command <<< "string"` |
| Temp file | `tmpfile=$(mktemp)` |
| File lock | `flock -n 200` |
| Check exists | `[ -e "$file" ]` |
| Check readable | `[ -r "$file" ]` |

---

**Previous:** [09-string-manipulation.md](09-string-manipulation.md) | **Next:** [11-text-processing.md](11-text-processing.md)
