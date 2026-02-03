# User Input in Shell Scripts

## The read Command

The `read` command reads input from the user or a file.

```bash
#!/bin/bash

# Basic read
echo "Enter your name:"
read name
echo "Hello, $name!"

# Read with prompt (-p)
read -p "Enter your age: " age
echo "You are $age years old"
```

## Read Options

| Option | Description |
|--------|-------------|
| `-p "prompt"` | Display prompt |
| `-s` | Silent mode (hide input) |
| `-n N` | Read N characters |
| `-t N` | Timeout in N seconds |
| `-r` | Raw input (no backslash escaping) |
| `-a array` | Read into array |
| `-d delim` | Use delimiter instead of newline |

### Examples

```bash
#!/bin/bash

# Silent input (passwords)
read -sp "Enter password: " password
echo
echo "Password received"

# Read with timeout
read -t 5 -p "Quick! Enter something (5 sec): " quick_input
if [ $? -ne 0 ]; then
    echo "Timeout!"
fi

# Read specific number of characters
read -n 1 -p "Press any key to continue..."
echo

# Read single character without Enter
read -n 1 -sp "Press Y to confirm: " confirm
echo
if [[ $confirm == "Y" || $confirm == "y" ]]; then
    echo "Confirmed!"
fi

# Raw input (preserve backslashes)
read -r path
echo "Path: $path"
```

### Reading Multiple Variables

```bash
#!/bin/bash

# Read multiple values into separate variables
echo "Enter first and last name:"
read firstname lastname
echo "First: $firstname"
echo "Last: $lastname"

# Extra words go to last variable
echo "Enter three words:"
read word1 word2 rest
echo "Word1: $word1"
echo "Word2: $word2"
echo "Rest: $rest"
```

### Reading into an Array

```bash
#!/bin/bash

# Read line into array
echo "Enter multiple items (space-separated):"
read -a items
echo "First item: ${items[0]}"
echo "All items: ${items[@]}"
echo "Number of items: ${#items[@]}"
```

## Command Line Arguments

### Positional Parameters

```bash
#!/bin/bash
# Usage: ./script.sh arg1 arg2 arg3

echo "Script: $0"
echo "First: $1"
echo "Second: $2"
echo "Third: $3"
echo "All args: $@"
echo "Arg count: $#"
```

### Checking Arguments

```bash
#!/bin/bash

# Check if arguments provided
if [ $# -eq 0 ]; then
    echo "Usage: $0 <filename>"
    exit 1
fi

# Check minimum arguments
if [ $# -lt 2 ]; then
    echo "Error: At least 2 arguments required"
    echo "Usage: $0 <source> <destination>"
    exit 1
fi

filename=$1
echo "Processing: $filename"
```

### Iterating Over Arguments

```bash
#!/bin/bash

# Using $@
echo "Method 1: Using \$@"
for arg in "$@"; do
    echo "  Argument: $arg"
done

# Using positional parameters
echo "Method 2: Using shift"
while [ $# -gt 0 ]; do
    echo "  Argument: $1"
    shift
done
```

## The shift Command

Shifts positional parameters to the left.

```bash
#!/bin/bash
# ./script.sh one two three four

echo "Before shift:"
echo "  \$1=$1, \$2=$2, \$3=$3, \$4=$4"
echo "  Count: $#"

shift

echo "After shift:"
echo "  \$1=$1, \$2=$2, \$3=$3, \$4=$4"
echo "  Count: $#"

shift 2

echo "After shift 2:"
echo "  \$1=$1, \$2=$2"
echo "  Count: $#"
```

```
Output:
Before shift:
  $1=one, $2=two, $3=three, $4=four
  Count: 4
After shift:
  $1=two, $2=three, $3=four, $4=
  Count: 3
After shift 2:
  $1=four, $2=
  Count: 1
```

## Parsing Options with getopts

### Basic getopts

```bash
#!/bin/bash

# Options: -v (verbose), -f <file>, -h (help)
while getopts "vf:h" opt; do
    case $opt in
        v)
            verbose=true
            ;;
        f)
            file="$OPTARG"
            ;;
        h)
            echo "Usage: $0 [-v] [-f file] [-h]"
            exit 0
            ;;
        \?)
            echo "Invalid option: -$OPTARG"
            exit 1
            ;;
        :)
            echo "Option -$OPTARG requires an argument"
            exit 1
            ;;
    esac
done

# Shift past processed options
shift $((OPTIND - 1))

# Remaining arguments
echo "Verbose: $verbose"
echo "File: $file"
echo "Remaining args: $@"
```

### getopts Syntax

| Syntax | Meaning |
|--------|---------|
| `"abc"` | Options -a, -b, -c (no arguments) |
| `"a:"` | Option -a requires argument |
| `"a::"` | Option -a has optional argument |
| `$OPTARG` | Current option's argument |
| `$OPTIND` | Index of next argument |

### Complete Example

```bash
#!/bin/bash

# Default values
verbose=false
output_file=""
config_file="/etc/default.conf"

usage() {
    cat << EOF
Usage: $0 [options] <input_file>

Options:
    -v          Enable verbose mode
    -o FILE     Output file (default: stdout)
    -c FILE     Config file (default: /etc/default.conf)
    -h          Show this help

Example:
    $0 -v -o output.txt input.txt
EOF
    exit 1
}

while getopts "vo:c:h" opt; do
    case $opt in
        v) verbose=true ;;
        o) output_file="$OPTARG" ;;
        c) config_file="$OPTARG" ;;
        h) usage ;;
        \?) echo "Invalid option: -$OPTARG" >&2; usage ;;
        :) echo "Option -$OPTARG requires an argument" >&2; usage ;;
    esac
done

shift $((OPTIND - 1))

# Check required arguments
if [ $# -eq 0 ]; then
    echo "Error: Input file required"
    usage
fi

input_file="$1"

# Main logic
if $verbose; then
    echo "Verbose mode enabled"
    echo "Input: $input_file"
    echo "Output: ${output_file:-stdout}"
    echo "Config: $config_file"
fi
```

## Long Options with getopt

```bash
#!/bin/bash

# Using getopt for long options
OPTS=$(getopt -o vo:h --long verbose,output:,help -n "$0" -- "$@")
if [ $? -ne 0 ]; then
    echo "Failed to parse options"
    exit 1
fi

eval set -- "$OPTS"

verbose=false
output=""

while true; do
    case "$1" in
        -v|--verbose)
            verbose=true
            shift
            ;;
        -o|--output)
            output="$2"
            shift 2
            ;;
        -h|--help)
            echo "Usage: $0 [-v|--verbose] [-o|--output FILE]"
            exit 0
            ;;
        --)
            shift
            break
            ;;
        *)
            echo "Unknown option: $1"
            exit 1
            ;;
    esac
done

echo "Verbose: $verbose"
echo "Output: $output"
echo "Args: $@"
```

## Interactive Menus

### Simple Menu

```bash
#!/bin/bash

echo "Select an option:"
echo "1) Option One"
echo "2) Option Two"
echo "3) Exit"

read -p "Enter choice [1-3]: " choice

case $choice in
    1) echo "You selected Option One" ;;
    2) echo "You selected Option Two" ;;
    3) echo "Exiting..."; exit 0 ;;
    *) echo "Invalid option" ;;
esac
```

### Using select

```bash
#!/bin/bash

PS3="Please select an option: "
options=("Start Service" "Stop Service" "Restart Service" "Status" "Quit")

select opt in "${options[@]}"; do
    case $opt in
        "Start Service")
            echo "Starting service..."
            ;;
        "Stop Service")
            echo "Stopping service..."
            ;;
        "Restart Service")
            echo "Restarting service..."
            ;;
        "Status")
            echo "Checking status..."
            ;;
        "Quit")
            echo "Goodbye!"
            break
            ;;
        *)
            echo "Invalid option $REPLY"
            ;;
    esac
done
```

### Yes/No Confirmation

```bash
#!/bin/bash

confirm() {
    local prompt="${1:-Are you sure?}"
    local response

    read -p "$prompt [y/N]: " response
    case "$response" in
        [yY][eE][sS]|[yY])
            return 0
            ;;
        *)
            return 1
            ;;
    esac
}

# Usage
if confirm "Delete all files?"; then
    echo "Deleting..."
else
    echo "Cancelled"
fi
```

## Reading from Files

```bash
#!/bin/bash

# Read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < "input.txt"

# Read specific columns
while IFS=: read -r user pass uid gid desc home shell; do
    echo "User: $user, Home: $home"
done < /etc/passwd

# Read from command output
while read -r pid cmd; do
    echo "PID: $pid, Command: $cmd"
done < <(ps aux | awk '{print $2, $11}')
```

## Input Validation

```bash
#!/bin/bash

# Validate number
read_number() {
    local prompt="$1"
    local num

    while true; do
        read -p "$prompt" num
        if [[ $num =~ ^[0-9]+$ ]]; then
            echo "$num"
            return 0
        else
            echo "Please enter a valid number" >&2
        fi
    done
}

# Validate email
read_email() {
    local email

    while true; do
        read -p "Enter email: " email
        if [[ $email =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]; then
            echo "$email"
            return 0
        else
            echo "Invalid email format" >&2
        fi
    done
}

# Validate yes/no
read_yes_no() {
    local prompt="$1"
    local response

    while true; do
        read -p "$prompt [yes/no]: " response
        case "$response" in
            yes|y|YES|Y) return 0 ;;
            no|n|NO|N) return 1 ;;
            *) echo "Please answer yes or no" >&2 ;;
        esac
    done
}

# Usage
age=$(read_number "Enter your age: ")
echo "Age: $age"
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `read var` | Read into variable |
| `read -p "prompt"` | Read with prompt |
| `read -s` | Silent input |
| `read -t 5` | 5 second timeout |
| `read -n 1` | Read 1 character |
| `read -a arr` | Read into array |
| `$1, $2, ...` | Positional arguments |
| `$#` | Argument count |
| `$@` | All arguments |
| `shift` | Shift arguments left |
| `getopts` | Parse options |

---

**Previous:** [02-variables.md](02-variables.md) | **Next:** [04-operators.md](04-operators.md)
