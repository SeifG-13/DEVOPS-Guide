# Shell Scripting Best Practices

## Code Style

### Shebang and Header

```bash
#!/bin/bash
#
# Script Name: script-name.sh
# Description: Brief description of what this script does
# Author: Your Name
# Date: 2024-01-01
# Version: 1.0
# Usage: ./script-name.sh [options] <arguments>
#
# Dependencies: curl, jq
# Exit Codes:
#   0 - Success
#   1 - General error
#   2 - Invalid arguments
#

set -euo pipefail
```

### Naming Conventions

```bash
# Variables: lowercase with underscores
user_name="john"
max_retries=3
config_file="/etc/app/config.ini"

# Constants: uppercase with underscores
readonly MAX_CONNECTIONS=100
readonly DEFAULT_TIMEOUT=30
readonly CONFIG_DIR="/etc/myapp"

# Functions: lowercase with underscores
get_user_input() {
    # ...
}

validate_email() {
    # ...
}

# Use descriptive names
# Bad
x=10
fn() { ... }

# Good
retry_count=10
process_user_data() { ... }
```

### Indentation and Formatting

```bash
#!/bin/bash

# Use consistent indentation (2 or 4 spaces)
if [ condition ]; then
    for item in list; do
        if [ another_condition ]; then
            command1
            command2
        fi
    done
fi

# Long commands - use backslash for continuation
curl -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $token" \
    -d '{"key": "value"}' \
    "https://api.example.com/endpoint"

# Long conditionals
if [ "$var1" = "value1" ] && \
   [ "$var2" = "value2" ] && \
   [ "$var3" = "value3" ]; then
    echo "All conditions met"
fi

# Long pipelines
cat file.txt |
    grep "pattern" |
    sort |
    uniq -c |
    head -10
```

## Quoting

### Always Quote Variables

```bash
#!/bin/bash

# Always quote variables to prevent word splitting
name="John Doe"

# Bad - word splitting occurs
echo $name     # Works but risky

# Good - properly quoted
echo "$name"

# Especially important with file paths
file="/path/with spaces/file.txt"

# Bad
cat $file      # Fails

# Good
cat "$file"    # Works
```

### When to Use Different Quotes

```bash
#!/bin/bash

# Double quotes: allow variable expansion
name="John"
echo "Hello, $name"    # Hello, John

# Single quotes: literal strings
echo 'Hello, $name'    # Hello, $name

# Escape within double quotes
echo "She said \"Hello\""

# Command substitution in quotes
echo "Today is $(date)"
```

## Error Handling

### Use Strict Mode

```bash
#!/bin/bash

# Always use these at the start
set -e          # Exit on error
set -u          # Error on undefined variables
set -o pipefail # Pipeline fails if any command fails

# Or combined
set -euo pipefail
```

### Proper Error Messages

```bash
#!/bin/bash

# Write errors to stderr
error() {
    echo "[ERROR] $*" >&2
}

warn() {
    echo "[WARN] $*" >&2
}

info() {
    echo "[INFO] $*"
}

# Usage
error "Configuration file not found"
warn "Using default settings"
info "Processing started"
```

### Always Check Return Values

```bash
#!/bin/bash

# Check command success
if ! command -v curl &>/dev/null; then
    echo "Error: curl is required but not installed"
    exit 1
fi

# Check file operations
if ! cp source dest; then
    echo "Error: Failed to copy file"
    exit 1
fi

# Or use || for quick checks
cd /some/directory || { echo "Failed to change directory"; exit 1; }
```

## Security Considerations

### Never Trust User Input

```bash
#!/bin/bash

# Validate input
validate_username() {
    local username="$1"

    # Check for valid characters only
    if [[ ! "$username" =~ ^[a-zA-Z0-9_-]+$ ]]; then
        echo "Invalid username format"
        return 1
    fi

    # Check length
    if [ ${#username} -lt 3 ] || [ ${#username} -gt 32 ]; then
        echo "Username must be 3-32 characters"
        return 1
    fi

    return 0
}

# Sanitize before using in commands
user_input="$1"
sanitized=$(echo "$user_input" | tr -cd '[:alnum:]_-')
```

### Avoid Command Injection

```bash
#!/bin/bash

# Bad - vulnerable to injection
user_file="$1"
cat $user_file           # Dangerous!

# Bad - command injection possible
eval "ls $user_input"    # Never use eval with user input

# Good - quote variables
cat "$user_file"

# Good - use arrays for commands
files=("$@")
for file in "${files[@]}"; do
    cat "$file"
done

# Good - validate before use
if [[ "$user_file" =~ ^[a-zA-Z0-9/_.-]+$ ]] && [ -f "$user_file" ]; then
    cat "$user_file"
fi
```

### Handle Sensitive Data

```bash
#!/bin/bash

# Don't echo passwords
read -sp "Password: " password
echo

# Don't store passwords in variables if possible
# If needed, unset after use
unset password

# Use environment variables for secrets
api_key="${API_KEY:?API_KEY must be set}"

# Never log sensitive data
# Bad
echo "Connecting with password: $password"

# Good
echo "Connecting to database..."
```

### Secure Temporary Files

```bash
#!/bin/bash

# Always use mktemp
tmpfile=$(mktemp) || exit 1

# Set restrictive permissions
chmod 600 "$tmpfile"

# Clean up on exit
trap 'rm -f "$tmpfile"' EXIT

# Use temp directories for multiple files
tmpdir=$(mktemp -d) || exit 1
trap 'rm -rf "$tmpdir"' EXIT
```

## Performance Tips

### Avoid Unnecessary Subshells

```bash
#!/bin/bash

# Bad - creates subshell for each iteration
cat file.txt | while read -r line; do
    count=$((count + 1))
done
echo $count    # count is 0 (subshell)

# Good - no subshell
while read -r line; do
    count=$((count + 1))
done < file.txt
echo $count    # Correct count

# Good - process substitution
while read -r line; do
    count=$((count + 1))
done < <(some_command)
```

### Use Built-in Commands

```bash
#!/bin/bash

# Bad - external commands
result=$(echo "$var" | tr '[:lower:]' '[:upper:]')
length=$(echo "$var" | wc -c)

# Good - bash built-ins
result="${var^^}"
length=${#var}

# Bad - external grep
if echo "$string" | grep -q "pattern"; then

# Good - bash pattern matching
if [[ "$string" == *"pattern"* ]]; then
```

### Process Files Efficiently

```bash
#!/bin/bash

# Bad - read entire file into memory
content=$(cat largefile.txt)

# Good - process line by line
while IFS= read -r line; do
    process "$line"
done < largefile.txt

# For simple operations, use awk/sed
# They're faster than bash loops
awk '{sum += $1} END {print sum}' data.txt
```

## Portability

### POSIX Compatibility

```bash
#!/bin/sh  # Use /bin/sh for portable scripts

# Portable constructs
test -f "$file"         # Instead of [[ ]]
command -v cmd          # Instead of which or type
$(command)              # Instead of `command`

# Avoid bashisms in portable scripts
# - [[ ]] (use [ ])
# - arrays (not POSIX)
# - ${var//pattern/replacement}
# - <<<  here-strings
# - process substitution <()
```

### Handle Different Systems

```bash
#!/bin/bash

# Detect OS
case "$(uname -s)" in
    Linux*)  OS="Linux" ;;
    Darwin*) OS="macOS" ;;
    CYGWIN*) OS="Windows" ;;
    *)       OS="Unknown" ;;
esac

# Use appropriate commands
if [ "$OS" = "macOS" ]; then
    stat -f%z "$file"    # macOS syntax
else
    stat -c%s "$file"    # Linux syntax
fi

# Check for command availability
if command -v gdate &>/dev/null; then
    DATE_CMD="gdate"    # GNU date on macOS
else
    DATE_CMD="date"
fi
```

## Documentation

### Comment Effectively

```bash
#!/bin/bash

# Document WHY, not WHAT
# Bad comment
count=$((count + 1))  # Increment count

# Good comment
# Retry counter - we retry up to 3 times due to flaky API
count=$((count + 1))

# Document complex logic
# Calculate exponential backoff delay
# Formula: min(max_delay, base_delay * 2^attempt)
delay=$((base_delay * (2 ** attempt)))
[ $delay -gt $max_delay ] && delay=$max_delay

# Document function purpose
# Validates email format and checks domain MX records
# Args: $1 - email address
# Returns: 0 if valid, 1 if invalid
validate_email() {
    local email="$1"
    # ...
}
```

### Usage Documentation

```bash
#!/bin/bash

usage() {
    cat << EOF
Usage: $(basename "$0") [OPTIONS] <argument>

Description:
    Brief description of what this script does.

Options:
    -h, --help      Show this help message
    -v, --verbose   Enable verbose output
    -d, --debug     Enable debug mode
    -f, --force     Force operation without confirmation
    -o FILE         Output to FILE instead of stdout

Arguments:
    argument        Description of required argument

Environment Variables:
    CONFIG_FILE     Path to configuration file
    DEBUG           Set to 'true' for debug output

Examples:
    $(basename "$0") -v input.txt
    $(basename "$0") --output=result.txt data.csv
    CONFIG_FILE=/etc/app.conf $(basename "$0") process

Exit Codes:
    0   Success
    1   General error
    2   Invalid arguments
    3   File not found

EOF
    exit "${1:-0}"
}
```

## Use ShellCheck

### Install and Run

```bash
# Install shellcheck
# Ubuntu/Debian
apt install shellcheck

# macOS
brew install shellcheck

# Run on script
shellcheck script.sh

# Integrate with editor (VS Code, vim, etc.)
```

### Common ShellCheck Warnings

```bash
#!/bin/bash

# SC2086: Double quote to prevent globbing
# Bad
echo $var
# Good
echo "$var"

# SC2046: Quote command substitution
# Bad
files=$(ls *.txt)
# Good
files="$(ls *.txt)"

# SC2164: Use || exit after cd
# Bad
cd /some/dir
# Good
cd /some/dir || exit

# SC2155: Declare and assign separately
# Bad
local var=$(command)
# Good
local var
var=$(command)
```

## Script Template

```bash
#!/bin/bash
#
# Script: template.sh
# Description: A template for shell scripts
# Author: Your Name
# Version: 1.0
#

set -euo pipefail

# Constants
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
readonly VERSION="1.0"

# Default values
VERBOSE=false
DEBUG=false

# Logging
log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
debug() { $DEBUG && echo "[DEBUG] $*" >&2 || true; }

# Usage
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [options] <argument>
Options:
    -h, --help      Show help
    -v, --verbose   Verbose output
    -d, --debug     Debug mode
    --version       Show version
EOF
    exit "${1:-0}"
}

# Cleanup
cleanup() {
    debug "Cleaning up..."
    # Add cleanup code here
}
trap cleanup EXIT

# Parse arguments
while [[ $# -gt 0 ]]; do
    case "$1" in
        -h|--help) usage ;;
        -v|--verbose) VERBOSE=true; shift ;;
        -d|--debug) DEBUG=true; set -x; shift ;;
        --version) echo "$VERSION"; exit 0 ;;
        --) shift; break ;;
        -*) error "Unknown option: $1"; usage 1 ;;
        *) break ;;
    esac
done

# Validate arguments
[[ $# -lt 1 ]] && { error "Missing argument"; usage 1; }

# Main logic
main() {
    local arg="$1"
    log "Starting with argument: $arg"

    # Your code here

    log "Completed successfully"
}

main "$@"
```

## Quick Reference Checklist

| Category | Best Practice |
|----------|---------------|
| Header | Shebang, description, author |
| Safety | `set -euo pipefail` |
| Variables | Quote all variables |
| Functions | Use local variables |
| Errors | Check return values |
| Security | Validate user input |
| Cleanup | Use trap for cleanup |
| Docs | Usage function, comments |
| Testing | Use ShellCheck |

---

**Previous:** [14-practical-scripts.md](14-practical-scripts.md)

---

## Congratulations!

You've completed the Shell Scripts section of the DevOps roadmap. You now understand:

- Shell scripting fundamentals and syntax
- Variables, operators, and control structures
- Functions and arrays
- Text processing with grep, sed, and awk
- Error handling and debugging
- Process management and signals
- Practical scripting patterns

**Next Topic:** [05-Docker](../05-Docker/)
