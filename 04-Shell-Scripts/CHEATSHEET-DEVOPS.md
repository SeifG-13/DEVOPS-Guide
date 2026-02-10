# Shell Scripting Cheat Sheet for DevOps Engineers

## Quick Reference - Bash Basics

### Script Header
```bash
#!/bin/bash
# Description: Script description here
# Author: Your Name
# Date: YYYY-MM-DD

set -euo pipefail  # Exit on error, undefined vars, pipe failures
IFS=$'\n\t'        # Safer word splitting
```

### Variables
```bash
NAME="value"                    # No spaces around =
readonly CONST="immutable"      # Constant
export PATH_VAR="/usr/bin"      # Environment variable
unset NAME                      # Delete variable

# Special variables
$0          # Script name
$1, $2...   # Positional arguments
$#          # Number of arguments
$@          # All arguments as separate strings
$*          # All arguments as single string
$?          # Exit status of last command
$$          # Current process ID
$!          # PID of last background process
```

### String Operations
```bash
str="Hello World"
${#str}                 # Length: 11
${str:0:5}              # Substring: Hello
${str/World/DevOps}     # Replace: Hello DevOps
${str,,}                # Lowercase: hello world
${str^^}                # Uppercase: HELLO WORLD
${str:-default}         # Default if unset
${str:?error message}   # Error if unset
```

### Arrays
```bash
arr=(one two three)
arr[3]="four"           # Add element
${arr[0]}               # First element
${arr[@]}               # All elements
${#arr[@]}              # Array length
${!arr[@]}              # Array indices

# Associative arrays
declare -A map
map[key]="value"
${map[key]}             # Access by key
```

### Conditionals
```bash
# If statement
if [[ condition ]]; then
    commands
elif [[ condition ]]; then
    commands
else
    commands
fi

# Test operators (strings)
[[ -z "$str" ]]         # Empty string
[[ -n "$str" ]]         # Non-empty string
[[ "$a" == "$b" ]]      # Equal
[[ "$a" != "$b" ]]      # Not equal
[[ "$a" =~ regex ]]     # Regex match

# Test operators (numbers)
[[ $a -eq $b ]]         # Equal
[[ $a -ne $b ]]         # Not equal
[[ $a -lt $b ]]         # Less than
[[ $a -le $b ]]         # Less or equal
[[ $a -gt $b ]]         # Greater than
[[ $a -ge $b ]]         # Greater or equal

# File tests
[[ -f file ]]           # Regular file exists
[[ -d dir ]]            # Directory exists
[[ -e path ]]           # Path exists
[[ -r file ]]           # Readable
[[ -w file ]]           # Writable
[[ -x file ]]           # Executable
[[ -s file ]]           # Non-empty file
[[ file1 -nt file2 ]]   # Newer than

# Logical operators
[[ cond1 && cond2 ]]    # AND
[[ cond1 || cond2 ]]    # OR
[[ ! condition ]]       # NOT
```

### Loops
```bash
# For loop
for i in 1 2 3 4 5; do
    echo "$i"
done

for file in *.txt; do
    echo "$file"
done

for ((i=0; i<10; i++)); do
    echo "$i"
done

# While loop
while [[ condition ]]; do
    commands
done

# Read file line by line
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Until loop
until [[ condition ]]; do
    commands
done
```

### Functions
```bash
function_name() {
    local var="local variable"
    echo "Arg 1: $1"
    return 0
}

# Call function
function_name "argument"
result=$?  # Get return value
```

### Input/Output
```bash
echo "Hello"                    # Print with newline
printf "Name: %s\n" "$name"     # Formatted output
read -p "Enter: " input         # Read user input
read -s -p "Password: " pass    # Silent input
read -t 5 input                 # Timeout after 5 seconds

# Redirection
cmd > file                      # Stdout to file
cmd >> file                     # Append stdout
cmd 2> file                     # Stderr to file
cmd &> file                     # Both to file
cmd < file                      # File as stdin
cmd1 | cmd2                     # Pipe
```

---

## Practical DevOps Scripts

### Health Check Script
```bash
#!/bin/bash
set -euo pipefail

SERVICES=("nginx" "docker" "postgresql")

for service in "${SERVICES[@]}"; do
    if systemctl is-active --quiet "$service"; then
        echo "[OK] $service is running"
    else
        echo "[FAIL] $service is NOT running"
        systemctl start "$service"
    fi
done
```

### Log Rotation Script
```bash
#!/bin/bash
LOG_DIR="/var/log/myapp"
DAYS_TO_KEEP=7

find "$LOG_DIR" -name "*.log" -mtime +$DAYS_TO_KEEP -delete
echo "Cleaned logs older than $DAYS_TO_KEEP days"
```

### Backup Script
```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/backup"
SOURCE="/var/www"
DATE=$(date +%Y%m%d_%H%M%S)
FILENAME="backup_${DATE}.tar.gz"

tar -czf "${BACKUP_DIR}/${FILENAME}" "$SOURCE"
echo "Backup created: ${BACKUP_DIR}/${FILENAME}"

# Keep only last 5 backups
cd "$BACKUP_DIR" && ls -t | tail -n +6 | xargs -r rm --
```

### Deployment Script
```bash
#!/bin/bash
set -euo pipefail

APP_DIR="/var/www/app"
REPO_URL="git@github.com:user/app.git"
BRANCH="${1:-main}"

echo "Deploying branch: $BRANCH"

cd "$APP_DIR"
git fetch origin
git checkout "$BRANCH"
git pull origin "$BRANCH"

# Build and restart
docker-compose build
docker-compose up -d

echo "Deployment complete!"
```

### Environment Setup Script
```bash
#!/bin/bash
set -euo pipefail

# Check if running as root
if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root"
    exit 1
fi

# Install packages
apt-get update
apt-get install -y \
    curl \
    git \
    docker.io \
    docker-compose

# Start services
systemctl enable docker
systemctl start docker

echo "Environment setup complete!"
```

---

## Interview Q&A

### Q1: What does `set -euo pipefail` do?
**A:**
- `-e`: Exit immediately on command failure
- `-u`: Treat unset variables as error
- `-o pipefail`: Return exit status of first failed command in pipe

### Q2: What is the difference between `$@` and `$*`?
**A:**
- `$@`: Each argument as separate quoted string
- `$*`: All arguments as single string
When quoted: `"$@"` preserves argument separation, `"$*"` combines them.

### Q3: How do you handle errors in bash scripts?
**A:**
```bash
# Use set -e or explicit checks
set -e

# Or use trap for cleanup
trap 'echo "Error on line $LINENO"; cleanup' ERR

# Or check each command
if ! command; then
    echo "Command failed"
    exit 1
fi
```

### Q4: What is command substitution?
**A:**
```bash
# Modern syntax (preferred)
result=$(command)

# Legacy syntax
result=`command`
```

### Q5: How do you pass arguments to a function?
**A:**
```bash
greet() {
    local name=$1
    local age=$2
    echo "Hello $name, you are $age years old"
}

greet "John" 30
```

### Q6: What is the difference between `[` and `[[`?
**A:**
- `[` (test): POSIX compliant, more portable
- `[[`: Bash-specific, more features (regex, no word splitting, pattern matching)
Use `[[` in bash scripts for safer string comparisons.

### Q7: How do you debug a bash script?
**A:**
```bash
# Enable debug mode
set -x              # Print commands before execution
bash -x script.sh   # Run with debug

# Selective debugging
set -x
problematic_code
set +x

# Verbose mode
set -v              # Print lines as read
```

### Q8: How do you read a file line by line?
**A:**
```bash
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Or for command output
while IFS= read -r line; do
    echo "$line"
done < <(command)
```

### Q9: What is a here document (heredoc)?
**A:**
```bash
cat << EOF
Multi-line
text content
with $variable expansion
EOF

# Without variable expansion
cat << 'EOF'
$variable stays literal
EOF
```

### Q10: How do you make a script executable?
**A:**
```bash
chmod +x script.sh
./script.sh

# Or run with interpreter
bash script.sh
```

---

## Common Patterns

### Argument Parsing
```bash
while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            echo "Usage: $0 [-v] [-o output]"
            exit 0
            ;;
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -o|--output)
            OUTPUT="$2"
            shift 2
            ;;
        *)
            echo "Unknown option: $1"
            exit 1
            ;;
    esac
done
```

### Logging Function
```bash
log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*"
}

log INFO "Script started"
log ERROR "Something went wrong"
```

### Retry Logic
```bash
retry() {
    local max_attempts=$1
    local delay=$2
    shift 2
    local attempt=1

    until "$@"; do
        if ((attempt >= max_attempts)); then
            echo "Failed after $attempt attempts"
            return 1
        fi
        echo "Attempt $attempt failed, retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
    done
}

retry 3 5 curl -f http://example.com
```

---

## Best Practices

1. **Always use `set -euo pipefail`** - Fail fast on errors
2. **Quote variables** - Prevent word splitting: `"$var"`
3. **Use `[[` over `[`** - Better string handling
4. **Use `local` in functions** - Avoid variable leakage
5. **Check return codes** - Handle failures gracefully
6. **Add logging** - Timestamp and severity levels
7. **Use shellcheck** - Static analysis tool
8. **Document scripts** - Header with description and usage
