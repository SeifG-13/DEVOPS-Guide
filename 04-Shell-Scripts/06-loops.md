# Loops in Shell Scripts

## for Loop

### Basic Syntax

```bash
for variable in list; do
    commands
done
```

### Iterating Over a List

```bash
#!/bin/bash

# List of values
for fruit in apple banana orange grape; do
    echo "Fruit: $fruit"
done

# List with quoted strings
for name in "John Doe" "Jane Smith" "Bob Wilson"; do
    echo "Name: $name"
done
```

### Iterating Over a Range

```bash
#!/bin/bash

# Using brace expansion
for i in {1..5}; do
    echo "Number: $i"
done

# With step
for i in {0..10..2}; do
    echo "Even: $i"
done

# Countdown
for i in {5..1}; do
    echo "Countdown: $i"
done
```

### C-style for Loop

```bash
#!/bin/bash

# Traditional C-style
for ((i=1; i<=5; i++)); do
    echo "Iteration: $i"
done

# Multiple variables
for ((i=0, j=10; i<5; i++, j--)); do
    echo "i=$i, j=$j"
done

# Infinite loop (use carefully)
# for ((;;)); do
#     echo "Forever..."
# done
```

### Iterating Over Files

```bash
#!/bin/bash

# All files in current directory
for file in *; do
    echo "File: $file"
done

# Specific pattern
for file in *.txt; do
    echo "Text file: $file"
done

# Multiple patterns
for file in *.txt *.log *.md; do
    [ -f "$file" ] && echo "Found: $file"
done

# Recursive with globstar (bash 4+)
shopt -s globstar
for file in **/*.txt; do
    echo "Found: $file"
done
```

### Iterating Over Command Output

```bash
#!/bin/bash

# Using command substitution
for user in $(cat /etc/passwd | cut -d: -f1); do
    echo "User: $user"
done

# Safer with while loop for lines
while IFS= read -r user; do
    echo "User: $user"
done < <(cut -d: -f1 /etc/passwd)
```

### Iterating Over Array

```bash
#!/bin/bash

# Array iteration
fruits=("apple" "banana" "orange" "grape")

# By value
for fruit in "${fruits[@]}"; do
    echo "Fruit: $fruit"
done

# By index
for i in "${!fruits[@]}"; do
    echo "Index $i: ${fruits[$i]}"
done
```

## while Loop

### Basic Syntax

```bash
while [ condition ]; do
    commands
done
```

### Simple Counter

```bash
#!/bin/bash

count=1

while [ $count -le 5 ]; do
    echo "Count: $count"
    ((count++))
done
```

### Reading File Line by Line

```bash
#!/bin/bash

# Recommended way to read files
while IFS= read -r line; do
    echo "Line: $line"
done < "input.txt"

# Preserve leading/trailing whitespace
while IFS= read -r line || [ -n "$line" ]; do
    echo "Line: $line"
done < "input.txt"
```

### Reading with Field Separator

```bash
#!/bin/bash

# Reading /etc/passwd
while IFS=: read -r user pass uid gid desc home shell; do
    echo "User: $user, Home: $home, Shell: $shell"
done < /etc/passwd

# Reading CSV
while IFS=, read -r name age city; do
    echo "$name is $age years old from $city"
done < "data.csv"
```

### Process Running Until Condition

```bash
#!/bin/bash

# Wait for file to exist
while [ ! -f "/tmp/ready.flag" ]; do
    echo "Waiting for ready flag..."
    sleep 5
done
echo "Ready flag found!"

# Wait for process to complete
while pgrep -x "backup_process" > /dev/null; do
    echo "Backup still running..."
    sleep 10
done
echo "Backup completed!"
```

### While with Command

```bash
#!/bin/bash

# While command succeeds
while ping -c 1 google.com &>/dev/null; do
    echo "Network is up"
    sleep 60
done
echo "Network is down!"
```

## until Loop

Opposite of while - runs until condition becomes true.

### Basic Syntax

```bash
until [ condition ]; do
    commands
done
```

### Examples

```bash
#!/bin/bash

# Count until 5
count=1
until [ $count -gt 5 ]; do
    echo "Count: $count"
    ((count++))
done

# Wait until server responds
until curl -s http://localhost:8080/health > /dev/null; do
    echo "Waiting for server..."
    sleep 2
done
echo "Server is ready!"

# Wait until file is removed
until [ ! -f "/tmp/lock.file" ]; do
    echo "Lock file exists, waiting..."
    sleep 1
done
echo "Lock released!"
```

## Loop Control

### break

Exit the loop entirely.

```bash
#!/bin/bash

# Break on condition
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        echo "Breaking at $i"
        break
    fi
    echo "Number: $i"
done
echo "Loop ended"

# Break from nested loop (break n)
for i in {1..3}; do
    for j in {1..3}; do
        if [ $j -eq 2 ]; then
            break 2  # Break both loops
        fi
        echo "i=$i, j=$j"
    done
done
```

### continue

Skip to next iteration.

```bash
#!/bin/bash

# Skip even numbers
for i in {1..10}; do
    if [ $((i % 2)) -eq 0 ]; then
        continue
    fi
    echo "Odd: $i"
done

# Continue in nested loop
for i in {1..3}; do
    for j in {1..3}; do
        if [ $j -eq 2 ]; then
            continue 2  # Continue outer loop
        fi
        echo "i=$i, j=$j"
    done
done
```

## Infinite Loops

```bash
#!/bin/bash

# Method 1: while true
while true; do
    echo "Running..."
    sleep 1
done

# Method 2: while :
while :; do
    echo "Running..."
    sleep 1
done

# Method 3: for (())
for ((;;)); do
    echo "Running..."
    sleep 1
done

# With controlled exit
while true; do
    read -p "Continue? (y/n): " answer
    if [ "$answer" = "n" ]; then
        break
    fi
    echo "Continuing..."
done
```

## Practical Examples

### File Processing

```bash
#!/bin/bash

# Process all log files
process_logs() {
    for logfile in /var/log/*.log; do
        if [ -f "$logfile" ]; then
            echo "Processing: $logfile"
            lines=$(wc -l < "$logfile")
            echo "  Lines: $lines"
        fi
    done
}

process_logs
```

### Batch File Renaming

```bash
#!/bin/bash

# Rename all .txt to .bak
for file in *.txt; do
    if [ -f "$file" ]; then
        newname="${file%.txt}.bak"
        echo "Renaming: $file -> $newname"
        mv "$file" "$newname"
    fi
done
```

### Retry Mechanism

```bash
#!/bin/bash

max_retries=3
retry_delay=5

retry_command() {
    local cmd="$1"
    local retries=0

    until eval "$cmd"; do
        ((retries++))
        if [ $retries -ge $max_retries ]; then
            echo "Failed after $max_retries attempts"
            return 1
        fi
        echo "Attempt $retries failed, retrying in ${retry_delay}s..."
        sleep $retry_delay
    done

    echo "Command succeeded"
    return 0
}

# Usage
retry_command "curl -s http://api.example.com/health"
```

### Progress Indicator

```bash
#!/bin/bash

# Simple progress bar
show_progress() {
    local current=$1
    local total=$2
    local width=50
    local percent=$((current * 100 / total))
    local filled=$((current * width / total))
    local empty=$((width - filled))

    printf "\r["
    printf "%${filled}s" | tr ' ' '#'
    printf "%${empty}s" | tr ' ' '-'
    printf "] %3d%%" "$percent"
}

# Usage
total=100
for i in $(seq 1 $total); do
    show_progress $i $total
    sleep 0.05
done
echo
```

### Parallel Processing (Simple)

```bash
#!/bin/bash

# Process files in parallel (limited concurrency)
max_jobs=4
current_jobs=0

for file in *.txt; do
    # Run in background
    (
        echo "Processing $file..."
        sleep 2  # Simulate work
        echo "Done: $file"
    ) &

    ((current_jobs++))

    # Wait if max jobs reached
    if [ $current_jobs -ge $max_jobs ]; then
        wait -n  # Wait for any job to complete (bash 4.3+)
        ((current_jobs--))
    fi
done

# Wait for remaining jobs
wait
echo "All files processed"
```

### Menu with Loop

```bash
#!/bin/bash

while true; do
    echo ""
    echo "================================"
    echo "        Main Menu"
    echo "================================"
    echo "1. Show date and time"
    echo "2. Show disk space"
    echo "3. Show memory usage"
    echo "4. Show uptime"
    echo "5. Exit"
    echo "================================"

    read -p "Select option [1-5]: " choice

    case $choice in
        1) date ;;
        2) df -h ;;
        3) free -h ;;
        4) uptime ;;
        5)
            echo "Goodbye!"
            break
            ;;
        *)
            echo "Invalid option"
            ;;
    esac

    read -p "Press Enter to continue..."
done
```

### Directory Tree Walker

```bash
#!/bin/bash

walk_directory() {
    local dir="$1"
    local indent="$2"

    for item in "$dir"/*; do
        if [ -e "$item" ]; then
            echo "${indent}$(basename "$item")"
            if [ -d "$item" ]; then
                walk_directory "$item" "  $indent"
            fi
        fi
    done
}

# Usage
echo "Directory tree of $(pwd):"
walk_directory "." ""
```

## Loop Comparison

| Loop Type | Use When |
|-----------|----------|
| `for` | Known number of iterations, lists, files |
| `while` | Unknown iterations, condition-based |
| `until` | Run until condition becomes true |

## Quick Reference

| Syntax | Description |
|--------|-------------|
| `for i in list; do..done` | Iterate over list |
| `for ((i=0;i<n;i++))` | C-style for |
| `while [ condition ]; do..done` | While condition true |
| `until [ condition ]; do..done` | Until condition true |
| `break` | Exit loop |
| `break n` | Exit n nested loops |
| `continue` | Skip to next iteration |
| `continue n` | Continue nth outer loop |

---

**Previous:** [05-conditionals.md](05-conditionals.md) | **Next:** [07-functions.md](07-functions.md)
