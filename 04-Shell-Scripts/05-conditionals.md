# Conditionals in Shell Scripts

## if Statement

### Basic Syntax

```bash
if [ condition ]; then
    # commands
fi

# Or on multiple lines
if [ condition ]
then
    commands
fi
```

### Simple Example

```bash
#!/bin/bash

age=18

if [ $age -ge 18 ]; then
    echo "You are an adult"
fi
```

## if-else Statement

```bash
#!/bin/bash

age=15

if [ $age -ge 18 ]; then
    echo "You are an adult"
else
    echo "You are a minor"
fi
```

## if-elif-else Statement

```bash
#!/bin/bash

score=75

if [ $score -ge 90 ]; then
    echo "Grade: A"
elif [ $score -ge 80 ]; then
    echo "Grade: B"
elif [ $score -ge 70 ]; then
    echo "Grade: C"
elif [ $score -ge 60 ]; then
    echo "Grade: D"
else
    echo "Grade: F"
fi
```

## Nested if Statements

```bash
#!/bin/bash

age=25
has_license=true

if [ $age -ge 18 ]; then
    echo "Age requirement met"
    if [ "$has_license" = true ]; then
        echo "You can drive"
    else
        echo "Get a license first"
    fi
else
    echo "Too young to drive"
fi
```

## test Command

The `[ ]` is actually the `test` command.

```bash
#!/bin/bash

# These are equivalent
if test -f "/etc/passwd"; then
    echo "File exists"
fi

if [ -f "/etc/passwd" ]; then
    echo "File exists"
fi
```

## [[ ]] vs [ ]

`[[ ]]` is Bash-specific with enhanced features.

```bash
#!/bin/bash

name="John Doe"

# [ ] requires careful quoting
if [ "$name" = "John Doe" ]; then
    echo "Match"
fi

# [[ ]] handles spaces better
if [[ $name = "John Doe" ]]; then
    echo "Match"
fi

# [[ ]] supports pattern matching
if [[ $name == J* ]]; then
    echo "Starts with J"
fi

# [[ ]] supports regex
if [[ $name =~ ^[A-Z] ]]; then
    echo "Starts with uppercase"
fi
```

## Compound Conditions

### AND Conditions

```bash
#!/bin/bash

age=25
income=50000

# Using -a (older style)
if [ $age -ge 18 -a $income -ge 30000 ]; then
    echo "Eligible for loan"
fi

# Using && with [[ ]]
if [[ $age -ge 18 && $income -ge 30000 ]]; then
    echo "Eligible for loan"
fi

# Using && between [ ]
if [ $age -ge 18 ] && [ $income -ge 30000 ]; then
    echo "Eligible for loan"
fi
```

### OR Conditions

```bash
#!/bin/bash

day="Saturday"

# Using -o (older style)
if [ "$day" = "Saturday" -o "$day" = "Sunday" ]; then
    echo "Weekend!"
fi

# Using || with [[ ]]
if [[ "$day" == "Saturday" || "$day" == "Sunday" ]]; then
    echo "Weekend!"
fi

# Using || between [ ]
if [ "$day" = "Saturday" ] || [ "$day" = "Sunday" ]; then
    echo "Weekend!"
fi
```

### NOT Condition

```bash
#!/bin/bash

file="data.txt"

# Negation
if [ ! -f "$file" ]; then
    echo "File does not exist"
fi

if ! [ -f "$file" ]; then
    echo "File does not exist"
fi

# With [[ ]]
if [[ ! -f "$file" ]]; then
    echo "File does not exist"
fi
```

### Complex Conditions

```bash
#!/bin/bash

age=30
country="USA"
income=75000

# Multiple conditions
if [[ ($age -ge 18 && $age -le 65) && ($country == "USA" || $country == "Canada") && $income -ge 50000 ]]; then
    echo "All conditions met"
fi
```

## case Statement

### Basic Syntax

```bash
case $variable in
    pattern1)
        commands
        ;;
    pattern2)
        commands
        ;;
    *)
        default commands
        ;;
esac
```

### Simple Example

```bash
#!/bin/bash

fruit="apple"

case $fruit in
    apple)
        echo "It's an apple"
        ;;
    banana)
        echo "It's a banana"
        ;;
    orange)
        echo "It's an orange"
        ;;
    *)
        echo "Unknown fruit"
        ;;
esac
```

### Multiple Patterns

```bash
#!/bin/bash

char="A"

case $char in
    [a-z])
        echo "Lowercase letter"
        ;;
    [A-Z])
        echo "Uppercase letter"
        ;;
    [0-9])
        echo "Digit"
        ;;
    *)
        echo "Special character"
        ;;
esac
```

### OR Patterns

```bash
#!/bin/bash

response="yes"

case $response in
    yes|y|Y|YES)
        echo "Affirmative"
        ;;
    no|n|N|NO)
        echo "Negative"
        ;;
    *)
        echo "Invalid response"
        ;;
esac
```

### Glob Patterns

```bash
#!/bin/bash

filename="document.pdf"

case $filename in
    *.txt)
        echo "Text file"
        ;;
    *.pdf)
        echo "PDF document"
        ;;
    *.jpg|*.png|*.gif)
        echo "Image file"
        ;;
    *.sh)
        echo "Shell script"
        ;;
    *)
        echo "Unknown file type"
        ;;
esac
```

### Fall-through (;;&) and Continue (;&)

```bash
#!/bin/bash
# Bash 4.0+ only

grade="B"

# Fall-through with ;;&
case $grade in
    A)
        echo "Excellent"
        ;;&
    A|B)
        echo "Good"
        ;;&
    A|B|C)
        echo "Passing"
        ;;
esac

# Output for B:
# Good
# Passing
```

## Practical Examples

### Menu System

```bash
#!/bin/bash

show_menu() {
    echo "========================"
    echo "   System Admin Menu"
    echo "========================"
    echo "1. Show disk usage"
    echo "2. Show memory usage"
    echo "3. Show system uptime"
    echo "4. Show logged in users"
    echo "5. Exit"
    echo "========================"
}

while true; do
    show_menu
    read -p "Enter choice [1-5]: " choice

    case $choice in
        1)
            echo "Disk Usage:"
            df -h
            ;;
        2)
            echo "Memory Usage:"
            free -h
            ;;
        3)
            echo "System Uptime:"
            uptime
            ;;
        4)
            echo "Logged in Users:"
            who
            ;;
        5)
            echo "Goodbye!"
            exit 0
            ;;
        *)
            echo "Invalid option. Please try again."
            ;;
    esac

    echo
    read -p "Press Enter to continue..."
done
```

### File Type Checker

```bash
#!/bin/bash

check_file() {
    local file="$1"

    if [ ! -e "$file" ]; then
        echo "Error: '$file' does not exist"
        return 1
    fi

    echo "Checking: $file"

    if [ -f "$file" ]; then
        echo "Type: Regular file"

        if [ -r "$file" ]; then
            echo "  - Readable: Yes"
        else
            echo "  - Readable: No"
        fi

        if [ -w "$file" ]; then
            echo "  - Writable: Yes"
        else
            echo "  - Writable: No"
        fi

        if [ -x "$file" ]; then
            echo "  - Executable: Yes"
        else
            echo "  - Executable: No"
        fi

        if [ -s "$file" ]; then
            echo "  - Size: $(stat -f%z "$file" 2>/dev/null || stat -c%s "$file" 2>/dev/null) bytes"
        else
            echo "  - Size: Empty"
        fi

    elif [ -d "$file" ]; then
        echo "Type: Directory"
    elif [ -L "$file" ]; then
        echo "Type: Symbolic link"
        echo "  Points to: $(readlink "$file")"
    else
        echo "Type: Other (device, socket, etc.)"
    fi
}

# Run with argument or prompt
if [ $# -gt 0 ]; then
    check_file "$1"
else
    read -p "Enter file path: " filepath
    check_file "$filepath"
fi
```

### User Authentication

```bash
#!/bin/bash

# Simple authentication example
authenticate_user() {
    local username password

    read -p "Username: " username
    read -sp "Password: " password
    echo

    # In real scripts, use proper authentication!
    if [[ "$username" == "admin" && "$password" == "secret123" ]]; then
        echo "Login successful!"
        return 0
    else
        echo "Invalid credentials"
        return 1
    fi
}

# Main
max_attempts=3
attempts=0

while [ $attempts -lt $max_attempts ]; do
    if authenticate_user; then
        echo "Welcome to the system"
        exit 0
    fi

    ((attempts++))
    remaining=$((max_attempts - attempts))

    if [ $remaining -gt 0 ]; then
        echo "Attempts remaining: $remaining"
    fi
done

echo "Too many failed attempts. Access denied."
exit 1
```

### Service Status Checker

```bash
#!/bin/bash

check_service() {
    local service="$1"

    if systemctl is-active --quiet "$service" 2>/dev/null; then
        echo "[RUNNING] $service"
        return 0
    elif systemctl is-enabled --quiet "$service" 2>/dev/null; then
        echo "[STOPPED] $service (but enabled)"
        return 1
    else
        echo "[UNKNOWN] $service"
        return 2
    fi
}

services=("sshd" "nginx" "docker" "mysql")

echo "Service Status Check"
echo "===================="

for service in "${services[@]}"; do
    check_service "$service"
done
```

## One-liner Conditionals

```bash
#!/bin/bash

# Short form for simple conditions
[ -f "file.txt" ] && echo "File exists"
[ -f "file.txt" ] || echo "File missing"

# Combined
[ -f "file.txt" ] && echo "Found" || echo "Not found"

# With commands
[ -d "/backup" ] || mkdir -p /backup

# Check command success
command -v git >/dev/null && echo "Git installed" || echo "Git not found"

# Ternary-like pattern
status=$([ -f "file.txt" ] && echo "exists" || echo "missing")
echo "File $status"
```

## Quick Reference

| Structure | Syntax |
|-----------|--------|
| if | `if [ condition ]; then ... fi` |
| if-else | `if [ ]; then ... else ... fi` |
| if-elif-else | `if [ ]; then ... elif [ ]; then ... else ... fi` |
| case | `case $var in pattern) ... ;; esac` |
| One-liner AND | `[ condition ] && command` |
| One-liner OR | `[ condition ] || command` |

---

**Previous:** [04-operators.md](04-operators.md) | **Next:** [06-loops.md](06-loops.md)
