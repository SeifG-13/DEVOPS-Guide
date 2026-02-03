# Functions in Shell Scripts

## What are Functions?

Functions are reusable blocks of code that perform specific tasks.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Function Structure                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   function_name() {                                              │
│       # Local variables                                          │
│       # Commands                                                 │
│       # Return value                                             │
│   }                                                              │
│                                                                  │
│   function_name arg1 arg2    # Call function                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Defining Functions

### Syntax Options

```bash
#!/bin/bash

# Method 1: Standard syntax
function_name() {
    commands
}

# Method 2: Using function keyword
function function_name {
    commands
}

# Method 3: Combining both
function function_name() {
    commands
}
```

### Simple Examples

```bash
#!/bin/bash

# Define function
greet() {
    echo "Hello, World!"
}

# Call function
greet
```

## Function Parameters

Functions receive arguments via positional parameters.

```bash
#!/bin/bash

greet_user() {
    echo "Hello, $1!"
    echo "Your age is $2"
}

# Call with arguments
greet_user "John" 25
greet_user "Jane" 30
```

### All Parameters

```bash
#!/bin/bash

show_params() {
    echo "Function name: $0"       # Script name (not function)
    echo "First param: $1"
    echo "Second param: $2"
    echo "All params (\$@): $@"
    echo "All params (\$*): $*"
    echo "Param count: $#"
}

show_params "one" "two" "three"
```

### Processing Multiple Arguments

```bash
#!/bin/bash

# Sum all arguments
sum_all() {
    local total=0
    for num in "$@"; do
        ((total += num))
    done
    echo $total
}

result=$(sum_all 10 20 30 40)
echo "Sum: $result"

# Print each argument
print_all() {
    local count=1
    for arg in "$@"; do
        echo "Arg $count: $arg"
        ((count++))
    done
}

print_all "apple" "banana" "cherry"
```

## Return Values

### Using return

`return` sets the exit status (0-255).

```bash
#!/bin/bash

# Return success/failure
is_even() {
    if [ $(($1 % 2)) -eq 0 ]; then
        return 0    # Success (true)
    else
        return 1    # Failure (false)
    fi
}

# Check return value
if is_even 4; then
    echo "4 is even"
fi

if ! is_even 5; then
    echo "5 is odd"
fi

# Or check $?
is_even 6
if [ $? -eq 0 ]; then
    echo "6 is even"
fi
```

### Returning Data

Use `echo` to return data.

```bash
#!/bin/bash

# Return a string
get_greeting() {
    echo "Hello, $1!"
}

# Capture output
message=$(get_greeting "World")
echo "$message"

# Return calculated value
calculate_square() {
    local num=$1
    echo $((num * num))
}

result=$(calculate_square 5)
echo "Square of 5 is $result"
```

### Return vs Echo

```bash
#!/bin/bash

# return: Exit status (0-255)
# echo: Output data

validate_age() {
    local age=$1
    if [ $age -ge 0 ] && [ $age -le 150 ]; then
        echo "$age is valid"
        return 0
    else
        echo "$age is invalid"
        return 1
    fi
}

# Both output and status
message=$(validate_age 25)
status=$?
echo "Message: $message"
echo "Status: $status"
```

## Local Variables

Variables declared with `local` are scoped to the function.

```bash
#!/bin/bash

global_var="I am global"

my_function() {
    local local_var="I am local"
    global_var="Modified global"

    echo "Inside function:"
    echo "  local_var: $local_var"
    echo "  global_var: $global_var"
}

echo "Before function:"
echo "  global_var: $global_var"

my_function

echo "After function:"
echo "  global_var: $global_var"
echo "  local_var: $local_var"    # Empty - not accessible
```

### Best Practice: Always Use Local

```bash
#!/bin/bash

# Bad - modifies global state
bad_function() {
    result=100    # Creates/modifies global variable
}

# Good - uses local variables
good_function() {
    local result=100
    echo $result
}

# Capture return value
value=$(good_function)
echo "Value: $value"
```

## Recursive Functions

Functions can call themselves.

```bash
#!/bin/bash

# Factorial calculation
factorial() {
    local n=$1
    if [ $n -le 1 ]; then
        echo 1
    else
        local prev=$(factorial $((n - 1)))
        echo $((n * prev))
    fi
}

echo "5! = $(factorial 5)"    # 120

# Directory tree
print_tree() {
    local dir="$1"
    local prefix="$2"

    for item in "$dir"/*; do
        if [ -e "$item" ]; then
            echo "${prefix}$(basename "$item")"
            if [ -d "$item" ]; then
                print_tree "$item" "  $prefix"
            fi
        fi
    done
}

print_tree "/path/to/dir" ""
```

### Recursion Depth Limit

```bash
#!/bin/bash

# Bash has a recursion limit
# FUNCNEST controls maximum nesting (default varies)

# Check/set limit
echo "Current limit: $FUNCNEST"
FUNCNEST=100    # Set limit

# Handle potential stack overflow
safe_recursive() {
    local depth=$1
    if [ $depth -gt 50 ]; then
        echo "Max depth reached"
        return 1
    fi
    # ... recursive call
}
```

## Function Libraries

Create reusable function files.

### Library File (utils.sh)

```bash
#!/bin/bash
# utils.sh - Utility functions library

# Logging functions
log_info() {
    echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" >&2
}

log_debug() {
    [ "$DEBUG" = "true" ] && echo "[DEBUG] $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# String functions
trim() {
    local var="$1"
    var="${var#"${var%%[![:space:]]*}"}"
    var="${var%"${var##*[![:space:]]}"}"
    echo "$var"
}

to_lower() {
    echo "$1" | tr '[:upper:]' '[:lower:]'
}

to_upper() {
    echo "$1" | tr '[:lower:]' '[:upper:]'
}

# Validation functions
is_number() {
    [[ $1 =~ ^[0-9]+$ ]]
}

is_email() {
    [[ $1 =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]
}

file_exists() {
    [ -f "$1" ]
}

dir_exists() {
    [ -d "$1" ]
}
```

### Using the Library

```bash
#!/bin/bash
# main.sh - Main script

# Source the library
source ./utils.sh
# or
. ./utils.sh

# Now use the functions
log_info "Script started"

email="test@example.com"
if is_email "$email"; then
    log_info "Valid email: $email"
else
    log_error "Invalid email: $email"
fi

name="  John Doe  "
trimmed=$(trim "$name")
echo "Trimmed: '$trimmed'"

log_info "Script completed"
```

## Advanced Function Techniques

### Named Parameters (Associative Array)

```bash
#!/bin/bash

# Bash 4.0+
create_user() {
    local -A params
    local key value

    # Parse key=value arguments
    for arg in "$@"; do
        key="${arg%%=*}"
        value="${arg#*=}"
        params[$key]="$value"
    done

    echo "Creating user:"
    echo "  Name: ${params[name]}"
    echo "  Email: ${params[email]}"
    echo "  Role: ${params[role]:-user}"    # Default: user
}

create_user name="John" email="john@example.com" role="admin"
```

### Function Pointers (Indirect Calls)

```bash
#!/bin/bash

say_hello() {
    echo "Hello!"
}

say_goodbye() {
    echo "Goodbye!"
}

# Store function name in variable
action="say_hello"
$action    # Calls say_hello

# Choose function dynamically
run_action() {
    local func_name=$1
    shift
    $func_name "$@"
}

run_action say_hello
run_action say_goodbye
```

### Functions with Options

```bash
#!/bin/bash

backup() {
    local source=""
    local dest=""
    local compress=false
    local verbose=false

    # Parse options
    while [[ $# -gt 0 ]]; do
        case $1 in
            -s|--source)
                source="$2"
                shift 2
                ;;
            -d|--dest)
                dest="$2"
                shift 2
                ;;
            -c|--compress)
                compress=true
                shift
                ;;
            -v|--verbose)
                verbose=true
                shift
                ;;
            *)
                echo "Unknown option: $1"
                return 1
                ;;
        esac
    done

    # Validate
    if [ -z "$source" ] || [ -z "$dest" ]; then
        echo "Usage: backup -s <source> -d <dest> [-c] [-v]"
        return 1
    fi

    # Execute
    $verbose && echo "Backing up $source to $dest"
    if $compress; then
        tar -czf "$dest/backup.tar.gz" "$source"
    else
        cp -r "$source" "$dest"
    fi
    $verbose && echo "Backup complete"
}

# Usage
backup -s /home/user/data -d /backup -c -v
```

## Practical Examples

### Configuration Loader

```bash
#!/bin/bash

load_config() {
    local config_file="$1"

    if [ ! -f "$config_file" ]; then
        echo "Config file not found: $config_file" >&2
        return 1
    fi

    while IFS='=' read -r key value; do
        # Skip comments and empty lines
        [[ $key =~ ^#.*$ ]] && continue
        [[ -z $key ]] && continue

        # Remove quotes from value
        value="${value%\"}"
        value="${value#\"}"

        # Export as environment variable
        export "$key=$value"
    done < "$config_file"
}

# config.ini:
# DB_HOST="localhost"
# DB_PORT=5432
# DB_NAME="myapp"

load_config "config.ini"
echo "Database: $DB_HOST:$DB_PORT/$DB_NAME"
```

### HTTP Client Functions

```bash
#!/bin/bash

http_get() {
    local url="$1"
    curl -s "$url"
}

http_post() {
    local url="$1"
    local data="$2"
    curl -s -X POST -H "Content-Type: application/json" -d "$data" "$url"
}

http_status() {
    local url="$1"
    curl -s -o /dev/null -w "%{http_code}" "$url"
}

# Usage
response=$(http_get "https://api.example.com/users")
status=$(http_status "https://api.example.com/health")

if [ "$status" = "200" ]; then
    echo "API is healthy"
fi
```

### Error Handler

```bash
#!/bin/bash

# Global error handler
error_handler() {
    local line_no=$1
    local error_code=$2
    echo "Error on line $line_no (exit code: $error_code)" >&2
    # Cleanup actions here
}

# Set trap
trap 'error_handler ${LINENO} $?' ERR

# Function with error handling
safe_operation() {
    local result
    if ! result=$(risky_command 2>&1); then
        log_error "Operation failed: $result"
        return 1
    fi
    echo "$result"
}
```

## Quick Reference

| Syntax | Description |
|--------|-------------|
| `func() { }` | Define function |
| `func arg1 arg2` | Call with arguments |
| `$1, $2, ...` | Access parameters |
| `$@` | All parameters |
| `$#` | Parameter count |
| `local var` | Local variable |
| `return n` | Return exit status |
| `echo value` | Return data |
| `source file` | Load library |

---

**Previous:** [06-loops.md](06-loops.md) | **Next:** [08-arrays.md](08-arrays.md)
