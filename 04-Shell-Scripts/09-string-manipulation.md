# String Manipulation in Shell Scripts

## String Basics

```bash
#!/bin/bash

# String assignment
str="Hello, World!"

# String with spaces
name="John Doe"

# Single quotes (literal)
literal='$HOME is not expanded'

# Double quotes (allows expansion)
expanded="Your home is $HOME"
```

## String Length

```bash
#!/bin/bash

str="Hello, World!"

# Get string length
echo ${#str}           # 13

# Using expr (older method)
echo $(expr length "$str")    # 13

# Check if string is empty
if [ -z "$str" ]; then
    echo "String is empty"
fi

# Check if string is not empty
if [ -n "$str" ]; then
    echo "String has content"
fi
```

## Substring Extraction

```bash
#!/bin/bash

str="Hello, World!"

# Syntax: ${string:position:length}

# From position to end
echo ${str:7}          # World!

# Specific length
echo ${str:0:5}        # Hello
echo ${str:7:5}        # World

# Negative index (from end) - Bash 4.2+
echo ${str: -6}        # World! (note the space before -)
echo ${str: -6:5}      # World

# Last N characters
echo ${str: -1}        # !
echo ${str: -3}        # ld!
```

## String Replacement

### Replace First Occurrence

```bash
#!/bin/bash

str="hello world world"

# Replace first occurrence
echo ${str/world/universe}    # hello universe world

# Syntax: ${string/pattern/replacement}
```

### Replace All Occurrences

```bash
#!/bin/bash

str="hello world world"

# Replace all occurrences
echo ${str//world/universe}   # hello universe universe

# Syntax: ${string//pattern/replacement}
```

### Delete Pattern

```bash
#!/bin/bash

str="hello world"

# Delete pattern (replace with nothing)
echo ${str/world/}      # hello
echo ${str// /}         # helloworld (remove all spaces)
```

### Replace at Beginning or End

```bash
#!/bin/bash

str="hello world"

# Replace at beginning
echo ${str/#hello/hi}   # hi world

# Replace at end
echo ${str/%world/universe}   # hello universe

# Only matches if pattern is at start/end
echo ${str/#world/X}    # hello world (no change - world not at start)
```

## Removing Substrings

### Remove from Beginning

```bash
#!/bin/bash

path="/home/user/documents/file.txt"

# Remove shortest match from beginning
echo ${path#*/}         # home/user/documents/file.txt

# Remove longest match from beginning
echo ${path##*/}        # file.txt
```

### Remove from End

```bash
#!/bin/bash

filename="document.backup.tar.gz"

# Remove shortest match from end
echo ${filename%.*}     # document.backup.tar

# Remove longest match from end
echo ${filename%%.*}    # document
```

### Practical Examples

```bash
#!/bin/bash

# Get filename from path
path="/home/user/scripts/backup.sh"
filename=${path##*/}
echo "Filename: $filename"    # backup.sh

# Get directory from path
directory=${path%/*}
echo "Directory: $directory"  # /home/user/scripts

# Get file extension
file="document.tar.gz"
extension=${file##*.}
echo "Extension: $extension"  # gz

# Get filename without extension
name=${file%.*}
echo "Name: $name"           # document.tar

# Get base name (no path, no extension)
path="/home/user/document.txt"
base=${path##*/}
base=${base%.*}
echo "Base: $base"           # document
```

## Case Conversion (Bash 4.0+)

```bash
#!/bin/bash

str="Hello World"

# To uppercase
echo ${str^^}           # HELLO WORLD

# To lowercase
echo ${str,,}           # hello world

# First character uppercase
echo ${str^}            # Hello World

# First character lowercase
echo ${str,}            # hello World

# Toggle case (Bash 4.4+)
echo ${str~~}           # hELLO wORLD
```

### Selective Case Conversion

```bash
#!/bin/bash

str="Hello World"

# Convert specific characters
echo ${str^^[aeiou]}    # HEllO WOrld (vowels uppercase)
echo ${str,,[HW]}       # hello world (H and W lowercase)
```

### Using tr Command

```bash
#!/bin/bash

str="Hello World"

# To uppercase
echo "$str" | tr '[:lower:]' '[:upper:]'

# To lowercase
echo "$str" | tr '[:upper:]' '[:lower:]'

# Swap case
echo "$str" | tr '[:upper:][:lower:]' '[:lower:][:upper:]'
```

## String Comparison

```bash
#!/bin/bash

str1="hello"
str2="world"
str3="hello"

# Equality
if [ "$str1" = "$str3" ]; then
    echo "Strings are equal"
fi

# Inequality
if [ "$str1" != "$str2" ]; then
    echo "Strings are not equal"
fi

# Empty check
if [ -z "$str1" ]; then
    echo "String is empty"
fi

# Not empty check
if [ -n "$str1" ]; then
    echo "String is not empty"
fi

# Lexicographic comparison (use [[ ]])
if [[ "$str1" < "$str2" ]]; then
    echo "$str1 comes before $str2"
fi
```

## Pattern Matching

### Glob Patterns

```bash
#!/bin/bash

str="hello_world.txt"

# Using [[ ]] with patterns
if [[ "$str" == hello* ]]; then
    echo "Starts with hello"
fi

if [[ "$str" == *.txt ]]; then
    echo "Ends with .txt"
fi

if [[ "$str" == *_* ]]; then
    echo "Contains underscore"
fi

# case statement patterns
case "$str" in
    *.txt) echo "Text file" ;;
    *.sh)  echo "Shell script" ;;
    *)     echo "Other file" ;;
esac
```

### Regex Matching

```bash
#!/bin/bash

# Using =~ operator (Bash 3.0+)

email="user@example.com"

if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Valid email"
fi

# Capture groups with BASH_REMATCH
str="Error: code 404"
if [[ "$str" =~ code\ ([0-9]+) ]]; then
    echo "Found code: ${BASH_REMATCH[1]}"    # 404
fi

# Multiple captures
date="2024-01-15"
if [[ "$date" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    echo "Year: ${BASH_REMATCH[1]}"   # 2024
    echo "Month: ${BASH_REMATCH[2]}"  # 01
    echo "Day: ${BASH_REMATCH[3]}"    # 15
fi
```

## String Concatenation

```bash
#!/bin/bash

str1="Hello"
str2="World"

# Simple concatenation
result="$str1 $str2"
echo $result              # Hello World

# Without space
result="$str1$str2"
echo $result              # HelloWorld

# Append to variable
str="Hello"
str+=" World"
echo $str                 # Hello World

# Multiple strings
first="One"
second="Two"
third="Three"
combined="${first}-${second}-${third}"
echo $combined            # One-Two-Three
```

## Trimming Whitespace

```bash
#!/bin/bash

str="   Hello World   "

# Using parameter expansion
# Remove leading whitespace
trimmed="${str#"${str%%[![:space:]]*}"}"

# Remove trailing whitespace
trimmed="${trimmed%"${trimmed##*[![:space:]]}"}"

echo "[$trimmed]"    # [Hello World]

# Using xargs (simple method)
trimmed=$(echo "$str" | xargs)
echo "[$trimmed]"    # [Hello World]

# Using sed
trimmed=$(echo "$str" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
echo "[$trimmed]"

# Function to trim
trim() {
    local var="$1"
    var="${var#"${var%%[![:space:]]*}"}"
    var="${var%"${var##*[![:space:]]}"}"
    echo "$var"
}

result=$(trim "$str")
echo "[$result]"
```

## Splitting Strings

```bash
#!/bin/bash

# Split by delimiter
str="apple,banana,orange,grape"

# Using IFS
IFS=',' read -ra fruits <<< "$str"
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Using tr and loop
for fruit in $(echo "$str" | tr ',' ' '); do
    echo "$fruit"
done

# Split into array
IFS=',' arr=($str)
echo "First: ${arr[0]}"
echo "Count: ${#arr[@]}"

# Restore IFS
unset IFS
```

### Split Path

```bash
#!/bin/bash

# Split PATH into array
IFS=':' read -ra paths <<< "$PATH"

echo "Paths in PATH:"
for p in "${paths[@]}"; do
    echo "  $p"
done
```

## String Functions Library

```bash
#!/bin/bash
# string_utils.sh

# Get string length
str_length() {
    echo ${#1}
}

# Convert to uppercase
str_upper() {
    echo "${1^^}"
}

# Convert to lowercase
str_lower() {
    echo "${1,,}"
}

# Trim whitespace
str_trim() {
    local var="$1"
    var="${var#"${var%%[![:space:]]*}"}"
    var="${var%"${var##*[![:space:]]}"}"
    echo "$var"
}

# Check if string contains substring
str_contains() {
    [[ "$1" == *"$2"* ]]
}

# Check if string starts with prefix
str_starts_with() {
    [[ "$1" == "$2"* ]]
}

# Check if string ends with suffix
str_ends_with() {
    [[ "$1" == *"$2" ]]
}

# Reverse string
str_reverse() {
    echo "$1" | rev
}

# Repeat string n times
str_repeat() {
    local str="$1"
    local n="$2"
    local result=""
    for ((i=0; i<n; i++)); do
        result+="$str"
    done
    echo "$result"
}

# Replace all occurrences
str_replace() {
    echo "${1//$2/$3}"
}

# Extract substring
str_substring() {
    echo "${1:$2:$3}"
}

# Split string into array (result in SPLIT_RESULT)
str_split() {
    local IFS="$2"
    read -ra SPLIT_RESULT <<< "$1"
}

# Join array with delimiter
str_join() {
    local IFS="$1"
    shift
    echo "$*"
}

# Pad left
str_pad_left() {
    printf "%${2}s" "$1"
}

# Pad right
str_pad_right() {
    printf "%-${2}s" "$1"
}

# Usage examples
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    echo "String length: $(str_length 'hello')"
    echo "Uppercase: $(str_upper 'hello')"
    echo "Lowercase: $(str_lower 'HELLO')"
    echo "Trimmed: [$(str_trim '  hello  ')]"
    echo "Contains 'ell': $(str_contains 'hello' 'ell' && echo 'yes' || echo 'no')"
    echo "Reversed: $(str_reverse 'hello')"
    echo "Repeated: $(str_repeat 'ab' 3)"
fi
```

## Quick Reference

| Operation | Syntax | Example |
|-----------|--------|---------|
| Length | `${#string}` | `${#str}` |
| Substring | `${string:pos:len}` | `${str:0:5}` |
| Replace first | `${string/old/new}` | `${str/a/b}` |
| Replace all | `${string//old/new}` | `${str//a/b}` |
| Remove prefix | `${string#pattern}` | `${str#*/}` |
| Remove prefix (greedy) | `${string##pattern}` | `${str##*/}` |
| Remove suffix | `${string%pattern}` | `${str%.*}` |
| Remove suffix (greedy) | `${string%%pattern}` | `${str%%.*}` |
| Uppercase | `${string^^}` | `${str^^}` |
| Lowercase | `${string,,}` | `${str,,}` |
| Default value | `${string:-default}` | `${str:-"none"}` |

---

**Previous:** [08-arrays.md](08-arrays.md) | **Next:** [10-file-operations.md](10-file-operations.md)
