# Error Handling in Shell Scripts

## Exit Codes

Every command returns an exit code (0-255).

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Misuse of shell command |
| 126 | Permission denied |
| 127 | Command not found |
| 128 | Invalid exit argument |
| 128+n | Fatal signal n |
| 130 | Terminated by Ctrl+C |

### Checking Exit Codes

```bash
#!/bin/bash

# Check last command's exit code
ls /nonexistent 2>/dev/null
echo "Exit code: $?"

# Conditional based on exit code
if ls /tmp >/dev/null 2>&1; then
    echo "Success"
else
    echo "Failed"
fi

# Using && and ||
mkdir /tmp/test && echo "Created" || echo "Failed"
```

### Setting Exit Codes

```bash
#!/bin/bash

# Exit with specific code
exit 0    # Success
exit 1    # General error

# Return from function
my_function() {
    if [ -z "$1" ]; then
        return 1
    fi
    return 0
}

# Custom exit codes
readonly E_SUCCESS=0
readonly E_INVALID_ARGS=1
readonly E_FILE_NOT_FOUND=2
readonly E_PERMISSION_DENIED=3

check_file() {
    local file="$1"
    [ -z "$file" ] && return $E_INVALID_ARGS
    [ ! -f "$file" ] && return $E_FILE_NOT_FOUND
    [ ! -r "$file" ] && return $E_PERMISSION_DENIED
    return $E_SUCCESS
}
```

## set Options for Error Handling

### set -e (Exit on Error)

```bash
#!/bin/bash
set -e

# Script exits immediately if any command fails
echo "Step 1"
false           # This will cause script to exit
echo "Step 2"   # Never reached
```

### set -u (Unset Variables)

```bash
#!/bin/bash
set -u

# Script exits if undefined variable is used
echo "$undefined_var"   # Error: unbound variable
```

### set -o pipefail

```bash
#!/bin/bash
set -o pipefail

# Pipeline returns exit code of last failed command
# Without pipefail, only last command's exit code is returned
false | true
echo $?    # Returns 1 (false's exit code)
```

### Combined Settings

```bash
#!/bin/bash

# Best practice: combine all three
set -euo pipefail

# Or use short form
set -eu -o pipefail

# With errexit exception
set -e
command_that_may_fail || true    # Won't exit script
set +e                           # Disable errexit temporarily
risky_command
set -e                           # Re-enable
```

### Handling set -e Exceptions

```bash
#!/bin/bash
set -e

# Method 1: Using || true
grep "pattern" file.txt || true

# Method 2: Conditional check
if ! grep -q "pattern" file.txt; then
    echo "Pattern not found"
fi

# Method 3: Capture exit code
set +e
grep "pattern" file.txt
result=$?
set -e
if [ $result -ne 0 ]; then
    echo "grep failed with code $result"
fi
```

## trap Command

`trap` catches signals and executes cleanup code.

### Basic Syntax

```bash
trap 'commands' SIGNAL
```

### Common Signals

| Signal | Number | Description |
|--------|--------|-------------|
| EXIT | 0 | Script exit |
| HUP | 1 | Hangup |
| INT | 2 | Interrupt (Ctrl+C) |
| QUIT | 3 | Quit |
| TERM | 15 | Termination |
| ERR | - | Command error |
| DEBUG | - | Before each command |

### Cleanup on Exit

```bash
#!/bin/bash

TMPFILE=""

cleanup() {
    echo "Cleaning up..."
    [ -n "$TMPFILE" ] && rm -f "$TMPFILE"
}

trap cleanup EXIT

TMPFILE=$(mktemp)
echo "Working with $TMPFILE"
# Script continues...
# cleanup runs automatically on exit
```

### Handle Ctrl+C

```bash
#!/bin/bash

handle_interrupt() {
    echo -e "\nInterrupted! Cleaning up..."
    # Cleanup code here
    exit 1
}

trap handle_interrupt INT

echo "Press Ctrl+C to interrupt"
while true; do
    sleep 1
    echo "Working..."
done
```

### Error Trap

```bash
#!/bin/bash
set -e

error_handler() {
    local line_no=$1
    local error_code=$2
    echo "Error on line $line_no (exit code: $error_code)"
}

trap 'error_handler ${LINENO} $?' ERR

# Commands here
echo "Running..."
false    # Triggers error trap
echo "Never reached"
```

### Multiple Traps

```bash
#!/bin/bash

cleanup() {
    echo "Cleanup called"
}

interrupt_handler() {
    echo "Interrupted!"
    cleanup
    exit 1
}

trap cleanup EXIT
trap interrupt_handler INT TERM

echo "Running..."
sleep 100
```

## Error Logging

### Basic Logging

```bash
#!/bin/bash

LOGFILE="/var/log/myscript.log"

log() {
    local level="$1"
    shift
    local message="$*"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $message" >> "$LOGFILE"
}

log_info()  { log "INFO" "$@"; }
log_warn()  { log "WARN" "$@"; }
log_error() { log "ERROR" "$@"; }

# Usage
log_info "Script started"
log_error "Something went wrong"
```

### Logging to stderr

```bash
#!/bin/bash

# Log errors to stderr
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
info "Processing file..."    # Goes to stdout
error "File not found"       # Goes to stderr
```

### Comprehensive Logging

```bash
#!/bin/bash

# Log levels
readonly LOG_LEVEL_DEBUG=0
readonly LOG_LEVEL_INFO=1
readonly LOG_LEVEL_WARN=2
readonly LOG_LEVEL_ERROR=3

LOG_LEVEL=${LOG_LEVEL:-$LOG_LEVEL_INFO}
LOGFILE="${LOGFILE:-/tmp/script.log}"

log() {
    local level=$1
    local level_name=$2
    shift 2
    local message="$*"

    if [ $level -ge $LOG_LEVEL ]; then
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        local log_line="[$timestamp] [$level_name] $message"

        echo "$log_line" >> "$LOGFILE"

        # Also print to appropriate stream
        if [ $level -ge $LOG_LEVEL_WARN ]; then
            echo "$log_line" >&2
        else
            echo "$log_line"
        fi
    fi
}

log_debug() { log $LOG_LEVEL_DEBUG "DEBUG" "$@"; }
log_info()  { log $LOG_LEVEL_INFO "INFO" "$@"; }
log_warn()  { log $LOG_LEVEL_WARN "WARN" "$@"; }
log_error() { log $LOG_LEVEL_ERROR "ERROR" "$@"; }

# Usage
LOG_LEVEL=$LOG_LEVEL_DEBUG    # Show all messages
log_debug "Detailed debug info"
log_info "Processing started"
log_warn "Low disk space"
log_error "Operation failed"
```

## Debugging

### Debug Mode

```bash
#!/bin/bash

# Enable debug mode
set -x    # Print commands as they execute

# Or run script with debug
# bash -x script.sh

# Disable debug mode
set +x
```

### Selective Debugging

```bash
#!/bin/bash

DEBUG=${DEBUG:-false}

debug() {
    if [ "$DEBUG" = true ]; then
        echo "[DEBUG] $*" >&2
    fi
}

# Run with: DEBUG=true ./script.sh
debug "Variable x = $x"
```

### Verbose Mode

```bash
#!/bin/bash

VERBOSE=${VERBOSE:-false}

verbose() {
    if [ "$VERBOSE" = true ]; then
        echo "$*"
    fi
}

# Usage
verbose "Processing file: $filename"
```

### Debug Functions

```bash
#!/bin/bash

# Print variable and its value
debug_var() {
    local var_name="$1"
    eval "local var_value=\$$var_name"
    echo "[DEBUG] $var_name = $var_value" >&2
}

# Print current function and line
debug_location() {
    echo "[DEBUG] Function: ${FUNCNAME[1]}, Line: ${BASH_LINENO[0]}" >&2
}

# Print stack trace
print_stack_trace() {
    local i=0
    echo "Stack trace:" >&2
    while caller $i; do
        ((i++))
    done
}

# Usage
my_function() {
    debug_location
    local x=10
    debug_var x
}

my_function
```

## Error Handling Patterns

### Try-Catch Pattern

```bash
#!/bin/bash

# Simulated try-catch
try() {
    [[ $- = *e* ]]; SAVED_OPT_E=$?
    set +e
}

catch() {
    export EXIT_CODE=$?
    (( $SAVED_OPT_E )) && set +e
    return $EXIT_CODE
}

throw() {
    exit "$1"
}

# Usage
try
(
    echo "Trying risky operation..."
    false    # Simulated error
)
catch || {
    echo "Caught error with code $EXIT_CODE"
}
```

### Die Function

```bash
#!/bin/bash

die() {
    local message="${1:-An error occurred}"
    local code="${2:-1}"
    echo "[FATAL] $message" >&2
    exit "$code"
}

# Usage
[ -f "required.conf" ] || die "Config file not found" 2
```

### Assert Function

```bash
#!/bin/bash

assert() {
    local condition="$1"
    local message="${2:-Assertion failed}"

    if ! eval "$condition"; then
        echo "[ASSERT] $message" >&2
        echo "[ASSERT] Condition: $condition" >&2
        exit 1
    fi
}

# Usage
count=5
assert '[ $count -gt 0 ]' "Count must be positive"
assert '[ -f "data.txt" ]' "Data file must exist"
```

### Retry Pattern

```bash
#!/bin/bash

retry() {
    local max_attempts="${1:-3}"
    local delay="${2:-5}"
    local command="${@:3}"
    local attempt=1

    while [ $attempt -le $max_attempts ]; do
        echo "Attempt $attempt of $max_attempts..."

        if eval "$command"; then
            echo "Success!"
            return 0
        fi

        if [ $attempt -lt $max_attempts ]; then
            echo "Failed, retrying in ${delay}s..."
            sleep "$delay"
        fi

        ((attempt++))
    done

    echo "All $max_attempts attempts failed"
    return 1
}

# Usage
retry 3 5 'curl -sf http://api.example.com/health'
```

## Complete Error Handling Template

```bash
#!/bin/bash
#
# Script: example.sh
# Description: Template with comprehensive error handling
#

set -euo pipefail

# Constants
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
readonly LOGFILE="/tmp/${SCRIPT_NAME%.*}.log"

# Error codes
readonly E_SUCCESS=0
readonly E_GENERAL=1
readonly E_INVALID_ARGS=2
readonly E_FILE_NOT_FOUND=3

# Logging functions
log() {
    local level="$1"; shift
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $*" | tee -a "$LOGFILE"
}

log_info()  { log "INFO" "$@"; }
log_warn()  { log "WARN" "$@" >&2; }
log_error() { log "ERROR" "$@" >&2; }

# Error handler
error_handler() {
    local line_no=$1
    local error_code=$2
    log_error "Script failed at line $line_no with exit code $error_code"
    cleanup
}

# Cleanup function
cleanup() {
    log_info "Performing cleanup..."
    # Add cleanup code here
}

# Set traps
trap 'error_handler ${LINENO} $?' ERR
trap cleanup EXIT
trap 'log_warn "Received interrupt signal"; exit 130' INT TERM

# Usage function
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [options] <argument>

Options:
    -h, --help      Show this help message
    -v, --verbose   Enable verbose output
    -d, --debug     Enable debug mode

Arguments:
    argument        Required argument

Example:
    $SCRIPT_NAME -v input.txt
EOF
    exit $E_SUCCESS
}

# Die function
die() {
    log_error "$1"
    exit "${2:-$E_GENERAL}"
}

# Main function
main() {
    local verbose=false
    local debug=false

    # Parse arguments
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -h|--help) usage ;;
            -v|--verbose) verbose=true; shift ;;
            -d|--debug) debug=true; set -x; shift ;;
            -*) die "Unknown option: $1" $E_INVALID_ARGS ;;
            *) break ;;
        esac
    done

    # Check required arguments
    [[ $# -lt 1 ]] && die "Missing required argument" $E_INVALID_ARGS

    local input="$1"
    [[ ! -f "$input" ]] && die "File not found: $input" $E_FILE_NOT_FOUND

    # Main logic
    log_info "Script started"
    $verbose && log_info "Verbose mode enabled"

    # Your code here

    log_info "Script completed successfully"
}

# Run main
main "$@"
```

## Quick Reference

| Feature | Syntax |
|---------|--------|
| Check exit code | `$?` |
| Exit with code | `exit 1` |
| Exit on error | `set -e` |
| Unset variable error | `set -u` |
| Pipeline fail | `set -o pipefail` |
| Trap signals | `trap 'cmd' SIGNAL` |
| Debug mode | `set -x` |
| Disable option | `set +e` |

---

**Previous:** [11-text-processing.md](11-text-processing.md) | **Next:** [13-process-management.md](13-process-management.md)
