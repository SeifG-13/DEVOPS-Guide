# Arrays in Shell Scripts

## What are Arrays?

Arrays store multiple values in a single variable.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Array Types in Bash                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Indexed Arrays            Associative Arrays                   │
│   ┌───┬───┬───┬───┐        ┌──────┬─────────┐                   │
│   │ 0 │ 1 │ 2 │ 3 │        │ key  │  value  │                   │
│   ├───┼───┼───┼───┤        ├──────┼─────────┤                   │
│   │ a │ b │ c │ d │        │name  │ John    │                   │
│   └───┴───┴───┴───┘        │age   │ 25      │                   │
│                             │city  │ NYC     │                   │
│   arr[0]=a                  └──────┴─────────┘                   │
│   arr[1]=b                                                       │
│                             dict[name]=John                      │
│                             dict[age]=25                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Indexed Arrays

### Creating Arrays

```bash
#!/bin/bash

# Method 1: Declare and assign
fruits=("apple" "banana" "orange" "grape")

# Method 2: Assign individually
colors[0]="red"
colors[1]="green"
colors[2]="blue"

# Method 3: Declare explicitly
declare -a numbers
numbers=(1 2 3 4 5)

# Method 4: From string
string="one two three"
words=($string)

# Method 5: From command output
files=($(ls *.txt 2>/dev/null))
```

### Accessing Elements

```bash
#!/bin/bash

fruits=("apple" "banana" "orange" "grape")

# Single element
echo ${fruits[0]}    # apple
echo ${fruits[1]}    # banana
echo ${fruits[2]}    # orange

# Last element
echo ${fruits[-1]}   # grape (Bash 4.3+)

# All elements
echo ${fruits[@]}    # apple banana orange grape
echo ${fruits[*]}    # apple banana orange grape

# Number of elements
echo ${#fruits[@]}   # 4

# Length of specific element
echo ${#fruits[0]}   # 5 (length of "apple")
```

### Array Indices

```bash
#!/bin/bash

fruits=("apple" "banana" "orange")

# Get all indices
echo ${!fruits[@]}    # 0 1 2

# Iterate with indices
for i in ${!fruits[@]}; do
    echo "Index $i: ${fruits[$i]}"
done
```

### Modifying Arrays

```bash
#!/bin/bash

fruits=("apple" "banana" "orange")

# Add element
fruits+=("grape")                    # Append
fruits[${#fruits[@]}]="melon"        # Add at end
fruits[10]="mango"                   # Sparse array (index 10)

# Modify element
fruits[1]="blueberry"                # Replace banana

# Remove element
unset fruits[2]                      # Remove orange

# Print array
echo "Array: ${fruits[@]}"
echo "Indices: ${!fruits[@]}"
```

### Array Slicing

```bash
#!/bin/bash

arr=(0 1 2 3 4 5 6 7 8 9)

# Slice syntax: ${arr[@]:start:length}

echo "${arr[@]:2:3}"     # 2 3 4 (start at 2, get 3 elements)
echo "${arr[@]:5}"       # 5 6 7 8 9 (from index 5 to end)
echo "${arr[@]::3}"      # 0 1 2 (first 3 elements)

# Negative offset (from end)
echo "${arr[@]: -3}"     # 7 8 9 (last 3 elements, note the space)
```

### Copying Arrays

```bash
#!/bin/bash

original=(1 2 3 4 5)

# Copy entire array
copy=("${original[@]}")

# Verify
echo "Original: ${original[@]}"
echo "Copy: ${copy[@]}"

# Modify copy (doesn't affect original)
copy[0]=100
echo "Modified copy: ${copy[@]}"
echo "Original still: ${original[@]}"
```

## Associative Arrays (Bash 4.0+)

Associative arrays use string keys instead of numeric indices.

### Creating Associative Arrays

```bash
#!/bin/bash

# Must declare with -A
declare -A user

# Assign values
user[name]="John"
user[age]=25
user[city]="New York"
user[email]="john@example.com"

# Or inline
declare -A config=(
    [host]="localhost"
    [port]="8080"
    [debug]="true"
)
```

### Accessing Elements

```bash
#!/bin/bash

declare -A user=(
    [name]="John"
    [age]=25
    [city]="New York"
)

# Single value
echo ${user[name]}      # John
echo ${user[age]}       # 25

# All values
echo ${user[@]}         # John 25 New York (order not guaranteed)

# All keys
echo ${!user[@]}        # name age city (order not guaranteed)

# Number of elements
echo ${#user[@]}        # 3

# Check if key exists
if [[ -v user[name] ]]; then
    echo "Key 'name' exists"
fi
```

### Iterating Associative Arrays

```bash
#!/bin/bash

declare -A config=(
    [host]="localhost"
    [port]="8080"
    [protocol]="https"
)

# Iterate over keys
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# Iterate over values only
for value in "${config[@]}"; do
    echo "Value: $value"
done
```

### Modifying Associative Arrays

```bash
#!/bin/bash

declare -A data

# Add entries
data[key1]="value1"
data[key2]="value2"

# Update entry
data[key1]="new_value1"

# Remove entry
unset data[key2]

# Clear all
unset data
declare -A data    # Recreate empty
```

## Array Operations

### Joining Array Elements

```bash
#!/bin/bash

arr=("apple" "banana" "orange")

# Join with space (default)
echo "${arr[*]}"

# Join with custom separator
IFS=','
echo "${arr[*]}"    # apple,banana,orange
unset IFS

# Function to join
join_by() {
    local IFS="$1"
    shift
    echo "$*"
}

result=$(join_by ',' "${arr[@]}")
echo "$result"    # apple,banana,orange
```

### Searching in Arrays

```bash
#!/bin/bash

arr=("apple" "banana" "orange" "grape")

# Check if element exists
contains() {
    local element="$1"
    shift
    local arr=("$@")

    for item in "${arr[@]}"; do
        if [[ "$item" == "$element" ]]; then
            return 0
        fi
    done
    return 1
}

if contains "banana" "${arr[@]}"; then
    echo "Found banana"
fi

# Find index of element
find_index() {
    local element="$1"
    shift
    local arr=("$@")

    for i in "${!arr[@]}"; do
        if [[ "${arr[$i]}" == "$element" ]]; then
            echo $i
            return 0
        fi
    done
    echo -1
    return 1
}

index=$(find_index "orange" "${arr[@]}")
echo "Index of orange: $index"    # 2
```

### Sorting Arrays

```bash
#!/bin/bash

arr=("banana" "apple" "cherry" "date")

# Sort array
sorted=($(printf '%s\n' "${arr[@]}" | sort))
echo "Sorted: ${sorted[@]}"

# Sort numerically
numbers=(10 2 33 4 5)
sorted_nums=($(printf '%s\n' "${numbers[@]}" | sort -n))
echo "Sorted numbers: ${sorted_nums[@]}"

# Reverse sort
reversed=($(printf '%s\n' "${arr[@]}" | sort -r))
echo "Reversed: ${reversed[@]}"
```

### Filtering Arrays

```bash
#!/bin/bash

numbers=(1 2 3 4 5 6 7 8 9 10)

# Filter even numbers
evens=()
for num in "${numbers[@]}"; do
    if (( num % 2 == 0 )); then
        evens+=("$num")
    fi
done
echo "Even numbers: ${evens[@]}"

# Filter with pattern
files=("file1.txt" "file2.log" "file3.txt" "file4.log")
txt_files=()
for file in "${files[@]}"; do
    if [[ "$file" == *.txt ]]; then
        txt_files+=("$file")
    fi
done
echo "Text files: ${txt_files[@]}"
```

### Mapping Arrays

```bash
#!/bin/bash

numbers=(1 2 3 4 5)

# Double each element
doubled=()
for num in "${numbers[@]}"; do
    doubled+=($((num * 2)))
done
echo "Doubled: ${doubled[@]}"

# Apply function to each
to_upper() {
    echo "$1" | tr '[:lower:]' '[:upper:]'
}

words=("hello" "world" "bash")
upper_words=()
for word in "${words[@]}"; do
    upper_words+=("$(to_upper "$word")")
done
echo "Upper: ${upper_words[@]}"
```

## Practical Examples

### Command Line Arguments as Array

```bash
#!/bin/bash

# Store all arguments in array
args=("$@")

echo "Number of arguments: ${#args[@]}"

for i in "${!args[@]}"; do
    echo "Arg $i: ${args[$i]}"
done
```

### Reading File into Array

```bash
#!/bin/bash

# Read lines into array
mapfile -t lines < input.txt
# or
readarray -t lines < input.txt

echo "Total lines: ${#lines[@]}"

for line in "${lines[@]}"; do
    echo "Line: $line"
done

# Read with custom delimiter
mapfile -t -d ',' items < data.csv
```

### Stack Implementation

```bash
#!/bin/bash

declare -a stack

push() {
    stack+=("$1")
}

pop() {
    if [ ${#stack[@]} -eq 0 ]; then
        echo "Stack is empty"
        return 1
    fi
    local index=$((${#stack[@]} - 1))
    echo "${stack[$index]}"
    unset stack[$index]
}

peek() {
    if [ ${#stack[@]} -eq 0 ]; then
        echo "Stack is empty"
        return 1
    fi
    echo "${stack[-1]}"
}

# Usage
push "first"
push "second"
push "third"

echo "Peek: $(peek)"     # third
echo "Pop: $(pop)"       # third
echo "Pop: $(pop)"       # second
echo "Stack: ${stack[@]}" # first
```

### Configuration Parser

```bash
#!/bin/bash

declare -A config

parse_config() {
    local file="$1"

    while IFS='=' read -r key value; do
        # Skip empty lines and comments
        [[ -z "$key" || "$key" =~ ^[[:space:]]*# ]] && continue

        # Trim whitespace
        key=$(echo "$key" | xargs)
        value=$(echo "$value" | xargs)

        # Remove quotes
        value="${value%\"}"
        value="${value#\"}"

        config[$key]="$value"
    done < "$file"
}

# Parse config
parse_config "app.conf"

# Access values
echo "Host: ${config[host]}"
echo "Port: ${config[port]}"

# Iterate all settings
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done
```

### Multi-dimensional Array (Simulated)

```bash
#!/bin/bash

# Bash doesn't have true multi-dimensional arrays
# Simulate with associative array

declare -A matrix

# Set values (row,col format)
matrix[0,0]=1
matrix[0,1]=2
matrix[0,2]=3
matrix[1,0]=4
matrix[1,1]=5
matrix[1,2]=6

# Access values
rows=2
cols=3

echo "Matrix:"
for ((i=0; i<rows; i++)); do
    for ((j=0; j<cols; j++)); do
        printf "%s " "${matrix[$i,$j]}"
    done
    echo
done
```

### Process Manager

```bash
#!/bin/bash

declare -A processes

start_process() {
    local name="$1"
    local cmd="$2"

    $cmd &
    processes[$name]=$!
    echo "Started $name with PID ${processes[$name]}"
}

stop_process() {
    local name="$1"

    if [[ -v processes[$name] ]]; then
        kill ${processes[$name]} 2>/dev/null
        unset processes[$name]
        echo "Stopped $name"
    else
        echo "Process $name not found"
    fi
}

list_processes() {
    echo "Running processes:"
    for name in "${!processes[@]}"; do
        if kill -0 ${processes[$name]} 2>/dev/null; then
            echo "  $name (PID: ${processes[$name]}) - Running"
        else
            echo "  $name (PID: ${processes[$name]}) - Stopped"
        fi
    done
}

# Usage
start_process "worker1" "sleep 100"
start_process "worker2" "sleep 100"
list_processes
stop_process "worker1"
```

## Quick Reference

| Operation | Indexed Array | Associative Array |
|-----------|---------------|-------------------|
| Declare | `declare -a arr` | `declare -A arr` |
| Assign | `arr=(a b c)` | `arr=([k1]=v1 [k2]=v2)` |
| Access | `${arr[0]}` | `${arr[key]}` |
| All elements | `${arr[@]}` | `${arr[@]}` |
| All keys/indices | `${!arr[@]}` | `${!arr[@]}` |
| Length | `${#arr[@]}` | `${#arr[@]}` |
| Append | `arr+=(val)` | `arr[key]=val` |
| Delete | `unset arr[0]` | `unset arr[key]` |
| Slice | `${arr[@]:1:2}` | N/A |

---

**Previous:** [07-functions.md](07-functions.md) | **Next:** [09-string-manipulation.md](09-string-manipulation.md)
