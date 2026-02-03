# Variables in Shell Scripts

## What are Variables?

Variables store data that can be used and modified throughout a script.

```bash
# Variable assignment (no spaces around =)
name="John"
age=25
path="/home/user"

# Using variables
echo $name
echo ${name}
echo "Hello, $name"
```

## Variable Naming Rules

| Rule | Valid | Invalid |
|------|-------|---------|
| Start with letter or underscore | `name`, `_count` | `1name`, `-var` |
| Alphanumeric and underscore | `user_name`, `file1` | `user-name`, `file.txt` |
| Case sensitive | `Name` ≠ `name` | - |
| No spaces around `=` | `x=10` | `x = 10` |

```bash
# Valid variable names
username="john"
USER_NAME="john"
_private="secret"
file1="data.txt"
myVar123="value"

# Invalid variable names
# 1file="data"      # Cannot start with number
# user-name="john"  # Cannot use hyphen
# my var="value"    # Cannot have spaces
```

## Variable Assignment

```bash
# String assignment
name="John Doe"
greeting='Hello World'

# Integer assignment
count=10
port=8080

# Command output assignment
current_date=$(date)
current_date=`date`    # Old syntax (backticks)

# Empty variable
empty_var=""
empty_var=

# Read-only variable
readonly PI=3.14159
declare -r CONSTANT="unchangeable"
```

## Using Variables

```bash
name="John"

# Basic usage
echo $name
echo ${name}

# Inside strings (double quotes)
echo "Hello, $name"
echo "Hello, ${name}!"

# Concatenation
echo "$name Doe"
echo "${name}Doe"      # Use braces when needed

# Single quotes (literal - no expansion)
echo 'Hello, $name'    # Prints: Hello, $name
```

### When to Use `${}`

```bash
filename="report"

# Without braces - ambiguous
echo "$filename_2024"  # Looks for variable filename_2024

# With braces - clear
echo "${filename}_2024"  # Prints: report_2024
```

## Variable Scope

### Local vs Global

```bash
#!/bin/bash

# Global variable
global_var="I am global"

my_function() {
    # Local variable (only in function)
    local local_var="I am local"

    # Modifying global from function
    global_var="Modified global"

    echo "Inside: $local_var"
    echo "Inside: $global_var"
}

my_function
echo "Outside: $global_var"     # Works
echo "Outside: $local_var"      # Empty (not accessible)
```

### Export for Child Processes

```bash
# Regular variable - not inherited by subshells
name="John"

# Exported variable - inherited by subshells
export name="John"
export PATH="$PATH:/new/path"

# Export existing variable
age=25
export age

# Check exported variables
export -p
```

## Special Variables

### Positional Parameters

```bash
#!/bin/bash
# script.sh arg1 arg2 arg3

echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Third argument: $3"
echo "All arguments: $@"
echo "All arguments: $*"
echo "Number of arguments: $#"
```

| Variable | Description |
|----------|-------------|
| `$0` | Script name |
| `$1` - `$9` | Arguments 1-9 |
| `${10}` | Argument 10+ (need braces) |
| `$#` | Number of arguments |
| `$@` | All arguments (separate words) |
| `$*` | All arguments (single word) |

### Process Variables

| Variable | Description |
|----------|-------------|
| `$$` | Current script PID |
| `$!` | Last background process PID |
| `$?` | Last command exit status |
| `$-` | Current shell options |

```bash
#!/bin/bash

echo "Script PID: $$"

sleep 10 &
echo "Background PID: $!"

ls /nonexistent 2>/dev/null
echo "Exit status: $?"    # Non-zero (error)

ls /tmp > /dev/null
echo "Exit status: $?"    # 0 (success)
```

### `$@` vs `$*`

```bash
#!/bin/bash
# Difference between $@ and $*

echo "Using \$@:"
for arg in "$@"; do
    echo "  $arg"
done

echo "Using \$*:"
for arg in "$*"; do
    echo "  $arg"
done
```

```bash
# Running: ./script.sh "hello world" "foo bar"

# $@ preserves separate arguments:
#   hello world
#   foo bar

# $* combines into one:
#   hello world foo bar
```

## Environment Variables

### Common Environment Variables

| Variable | Description |
|----------|-------------|
| `$HOME` | User home directory |
| `$USER` | Current username |
| `$PATH` | Executable search path |
| `$PWD` | Current directory |
| `$SHELL` | Current shell |
| `$HOSTNAME` | System hostname |
| `$RANDOM` | Random number (0-32767) |
| `$LINENO` | Current line number |

```bash
#!/bin/bash

echo "Home: $HOME"
echo "User: $USER"
echo "Path: $PATH"
echo "Current dir: $PWD"
echo "Shell: $SHELL"
echo "Random: $RANDOM"
echo "Line: $LINENO"
```

### Custom Environment Variables

```bash
# Set for current session
export MY_APP_ENV="production"
export MY_APP_PORT=3000

# Set for single command
MY_VAR="value" ./script.sh

# Unset variable
unset MY_VAR
```

## Default Values

```bash
# Use default if unset or null
echo ${name:-"Default Name"}

# Use default if unset (not null)
echo ${name-"Default"}

# Assign default if unset or null
echo ${name:="Default Name"}

# Error if unset or null
echo ${name:?"Variable not set"}

# Use value if set and not null
echo ${name:+"Alternative"}
```

| Syntax | Description |
|--------|-------------|
| `${var:-default}` | Use default if unset or null |
| `${var:=default}` | Assign default if unset or null |
| `${var:?error}` | Error if unset or null |
| `${var:+value}` | Use value if set and not null |

### Examples

```bash
#!/bin/bash

# Provide defaults for optional parameters
username=${1:-"guest"}
port=${2:-8080}

echo "User: $username"
echo "Port: $port"

# Require a variable
: ${API_KEY:?"API_KEY must be set"}

# Use alternative when set
debug=${DEBUG:+"[DEBUG MODE]"}
echo "${debug}Starting application..."
```

## Variable Substitution

```bash
filename="/home/user/documents/report.txt"

# Get string length
echo ${#filename}         # 32

# Substring extraction
echo ${filename:0:5}      # /home
echo ${filename:6}        # user/documents/report.txt
echo ${filename: -4}      # .txt (note the space)

# Remove from beginning
echo ${filename#*/}       # home/user/documents/report.txt
echo ${filename##*/}      # report.txt (greedy)

# Remove from end
echo ${filename%/*}       # /home/user/documents
echo ${filename%%/*}      # (empty - greedy)

# Replace
echo ${filename/user/admin}     # /home/admin/documents/report.txt
echo ${filename//o/O}           # /hOme/user/dOcuments/repOrt.txt

# Case conversion (bash 4+)
name="Hello World"
echo ${name^^}            # HELLO WORLD
echo ${name,,}            # hello world
echo ${name^}             # Hello world (first char)
```

## Declare Command

```bash
# Declare integer
declare -i number=10
number=number+5
echo $number              # 15

# Declare array
declare -a my_array

# Declare associative array
declare -A my_dict

# Declare read-only
declare -r constant="value"

# Declare exported
declare -x exported_var="value"

# Show variable attributes
declare -p number
```

| Option | Description |
|--------|-------------|
| `-i` | Integer |
| `-a` | Indexed array |
| `-A` | Associative array |
| `-r` | Read-only |
| `-x` | Export |
| `-l` | Lowercase |
| `-u` | Uppercase |

## Quick Reference

| Operation | Syntax |
|-----------|--------|
| Assign | `var="value"` |
| Use | `$var` or `${var}` |
| Default | `${var:-default}` |
| Length | `${#var}` |
| Substring | `${var:start:length}` |
| Replace | `${var/old/new}` |
| Export | `export var` |
| Read-only | `readonly var` |
| Unset | `unset var` |

---

**Previous:** [01-introduction.md](01-introduction.md) | **Next:** [03-user-input.md](03-user-input.md)
