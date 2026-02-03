# Operators in Shell Scripts

## Arithmetic Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `+` | Addition | `$((a + b))` |
| `-` | Subtraction | `$((a - b))` |
| `*` | Multiplication | `$((a * b))` |
| `/` | Division | `$((a / b))` |
| `%` | Modulus | `$((a % b))` |
| `**` | Exponentiation | `$((a ** b))` |

### Arithmetic Operations

```bash
#!/bin/bash

a=10
b=3

# Using $(( ))
echo "Addition: $((a + b))"         # 13
echo "Subtraction: $((a - b))"      # 7
echo "Multiplication: $((a * b))"   # 30
echo "Division: $((a / b))"         # 3
echo "Modulus: $((a % b))"          # 1
echo "Exponent: $((a ** b))"        # 1000

# Using let
let result=a+b
let "result = a + b"
echo "Result: $result"

# Using expr (older method)
result=$(expr $a + $b)
result=`expr $a \* $b`    # Note: * needs escaping with expr
```

### Increment and Decrement

```bash
#!/bin/bash

count=5

# Post-increment
echo $((count++))    # 5 (then count becomes 6)
echo $count          # 6

# Pre-increment
echo $((++count))    # 7 (count becomes 7 first)

# Post-decrement
echo $((count--))    # 7 (then count becomes 6)

# Pre-decrement
echo $((--count))    # 5

# Compound assignment
count=10
((count += 5))       # count = 15
((count -= 3))       # count = 12
((count *= 2))       # count = 24
((count /= 4))       # count = 6
```

### Floating Point with bc

```bash
#!/bin/bash

# Bash only does integer math
# Use bc for floating point

a=10
b=3

# Basic bc
result=$(echo "$a / $b" | bc)
echo "Division: $result"           # 3

# With decimal places
result=$(echo "scale=2; $a / $b" | bc)
echo "Division: $result"           # 3.33

# Complex calculation
result=$(echo "scale=4; sqrt($a)" | bc)
echo "Square root: $result"        # 3.1622

# Multiple operations
result=$(echo "scale=2; ($a + $b) * 2 / 3" | bc)
echo "Complex: $result"
```

## Comparison Operators

### Integer Comparison

| Operator | Description | Example |
|----------|-------------|---------|
| `-eq` | Equal | `[ $a -eq $b ]` |
| `-ne` | Not equal | `[ $a -ne $b ]` |
| `-gt` | Greater than | `[ $a -gt $b ]` |
| `-ge` | Greater or equal | `[ $a -ge $b ]` |
| `-lt` | Less than | `[ $a -lt $b ]` |
| `-le` | Less or equal | `[ $a -le $b ]` |

```bash
#!/bin/bash

a=10
b=20

if [ $a -eq $b ]; then
    echo "Equal"
fi

if [ $a -ne $b ]; then
    echo "Not equal"         # Prints
fi

if [ $a -lt $b ]; then
    echo "a is less than b"  # Prints
fi

if [ $a -gt $b ]; then
    echo "a is greater than b"
fi
```

### Integer Comparison with (( ))

| Operator | Description |
|----------|-------------|
| `==` | Equal |
| `!=` | Not equal |
| `>` | Greater than |
| `>=` | Greater or equal |
| `<` | Less than |
| `<=` | Less or equal |

```bash
#!/bin/bash

a=10
b=20

if (( a == b )); then
    echo "Equal"
fi

if (( a < b )); then
    echo "a is less than b"  # Prints
fi

if (( a >= 10 && b <= 20 )); then
    echo "Both conditions true"  # Prints
fi
```

### String Comparison

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equal | `[ "$a" = "$b" ]` |
| `==` | Equal (in [[ ]]) | `[[ "$a" == "$b" ]]` |
| `!=` | Not equal | `[ "$a" != "$b" ]` |
| `<` | Less than (ASCII) | `[[ "$a" < "$b" ]]` |
| `>` | Greater than (ASCII) | `[[ "$a" > "$b" ]]` |
| `-z` | String is empty | `[ -z "$a" ]` |
| `-n` | String is not empty | `[ -n "$a" ]` |

```bash
#!/bin/bash

str1="hello"
str2="world"
empty=""

# String equality
if [ "$str1" = "$str2" ]; then
    echo "Strings are equal"
fi

if [ "$str1" != "$str2" ]; then
    echo "Strings are not equal"  # Prints
fi

# Empty check
if [ -z "$empty" ]; then
    echo "String is empty"        # Prints
fi

if [ -n "$str1" ]; then
    echo "String is not empty"    # Prints
fi

# Pattern matching (only with [[ ]])
if [[ "$str1" == h* ]]; then
    echo "Starts with h"          # Prints
fi

if [[ "$str1" =~ ^hel ]]; then
    echo "Matches regex"          # Prints
fi
```

## Logical Operators

| Operator | Description | [ ] Syntax | [[ ]] Syntax |
|----------|-------------|------------|--------------|
| AND | Both true | `-a` | `&&` |
| OR | Either true | `-o` | `||` |
| NOT | Negation | `!` | `!` |

```bash
#!/bin/bash

a=10
b=20
c=10

# Using [ ] with -a and -o
if [ $a -eq $c -a $b -gt $a ]; then
    echo "Both conditions true"   # Prints
fi

if [ $a -gt $b -o $a -eq $c ]; then
    echo "At least one is true"   # Prints
fi

# Using [[ ]] with && and ||
if [[ $a -eq $c && $b -gt $a ]]; then
    echo "Both conditions true"   # Prints
fi

if [[ $a -gt $b || $a -eq $c ]]; then
    echo "At least one is true"   # Prints
fi

# NOT operator
if [ ! $a -gt $b ]; then
    echo "a is NOT greater than b"  # Prints
fi

# Combining with commands
if [ -f "file.txt" ] && [ -r "file.txt" ]; then
    echo "File exists and is readable"
fi
```

## File Test Operators

### File Existence and Type

| Operator | Description |
|----------|-------------|
| `-e` | File exists |
| `-f` | Regular file |
| `-d` | Directory |
| `-L` | Symbolic link |
| `-b` | Block device |
| `-c` | Character device |
| `-p` | Named pipe |
| `-S` | Socket |

```bash
#!/bin/bash

file="/etc/passwd"
dir="/tmp"
link="/usr/bin/python"

if [ -e "$file" ]; then
    echo "File exists"
fi

if [ -f "$file" ]; then
    echo "Is a regular file"
fi

if [ -d "$dir" ]; then
    echo "Is a directory"
fi

if [ -L "$link" ]; then
    echo "Is a symbolic link"
fi
```

### File Permissions

| Operator | Description |
|----------|-------------|
| `-r` | Readable |
| `-w` | Writable |
| `-x` | Executable |
| `-u` | SUID bit set |
| `-g` | SGID bit set |
| `-k` | Sticky bit set |

```bash
#!/bin/bash

file="script.sh"

if [ -r "$file" ]; then
    echo "File is readable"
fi

if [ -w "$file" ]; then
    echo "File is writable"
fi

if [ -x "$file" ]; then
    echo "File is executable"
fi

# Check all permissions
if [ -r "$file" ] && [ -w "$file" ] && [ -x "$file" ]; then
    echo "File has rwx permissions"
fi
```

### File Size and Properties

| Operator | Description |
|----------|-------------|
| `-s` | File size > 0 |
| `-N` | Modified since last read |
| `-O` | Owned by current user |
| `-G` | Owned by current group |

```bash
#!/bin/bash

file="data.txt"

if [ -s "$file" ]; then
    echo "File is not empty"
fi

if [ -O "$file" ]; then
    echo "You own this file"
fi
```

### File Comparison

| Operator | Description |
|----------|-------------|
| `-nt` | Newer than |
| `-ot` | Older than |
| `-ef` | Same file (hardlink) |

```bash
#!/bin/bash

file1="file1.txt"
file2="file2.txt"

if [ "$file1" -nt "$file2" ]; then
    echo "file1 is newer than file2"
fi

if [ "$file1" -ot "$file2" ]; then
    echo "file1 is older than file2"
fi

if [ "$file1" -ef "$file2" ]; then
    echo "Same file (hardlinks)"
fi
```

## Complete File Test Example

```bash
#!/bin/bash

check_file() {
    local file="$1"

    echo "Checking: $file"
    echo "========================"

    # Existence
    if [ -e "$file" ]; then
        echo "[✓] Exists"
    else
        echo "[✗] Does not exist"
        return 1
    fi

    # Type
    if [ -f "$file" ]; then
        echo "[✓] Regular file"
    elif [ -d "$file" ]; then
        echo "[✓] Directory"
    elif [ -L "$file" ]; then
        echo "[✓] Symbolic link"
    fi

    # Permissions
    [ -r "$file" ] && echo "[✓] Readable" || echo "[✗] Not readable"
    [ -w "$file" ] && echo "[✓] Writable" || echo "[✗] Not writable"
    [ -x "$file" ] && echo "[✓] Executable" || echo "[✗] Not executable"

    # Size
    if [ -s "$file" ]; then
        echo "[✓] Has content (size > 0)"
    else
        echo "[✗] Empty file"
    fi

    echo
}

# Usage
check_file "/etc/passwd"
check_file "/tmp"
check_file "$HOME/.bashrc"
```

## [ ] vs [[ ]]

| Feature | `[ ]` | `[[ ]]` |
|---------|-------|---------|
| POSIX compliant | Yes | No (Bash only) |
| Word splitting | Yes (quote variables) | No |
| Pattern matching | No | Yes (`==`, `!=`) |
| Regex matching | No | Yes (`=~`) |
| Logical operators | `-a`, `-o` | `&&`, `||` |
| `<`, `>` for strings | Need escaping | Works directly |

```bash
#!/bin/bash

name="John Doe"

# [ ] requires quoting
if [ "$name" = "John Doe" ]; then
    echo "Match with [ ]"
fi

# [[ ]] handles spaces without quotes
if [[ $name = "John Doe" ]]; then
    echo "Match with [[ ]]"
fi

# Pattern matching (only [[ ]])
if [[ $name == J* ]]; then
    echo "Starts with J"
fi

# Regex matching (only [[ ]])
if [[ $name =~ ^John ]]; then
    echo "Matches regex"
fi
```

## Quick Reference

| Category | Operator | Example |
|----------|----------|---------|
| Arithmetic | `+`, `-`, `*`, `/`, `%` | `$((a + b))` |
| Integer Compare | `-eq`, `-ne`, `-lt`, `-gt` | `[ $a -eq $b ]` |
| String Compare | `=`, `!=`, `-z`, `-n` | `[ "$a" = "$b" ]` |
| Logical | `-a`, `-o`, `!` | `[ $a -a $b ]` |
| Logical | `&&`, `||`, `!` | `[[ $a && $b ]]` |
| File Test | `-f`, `-d`, `-e`, `-r`, `-w`, `-x` | `[ -f file ]` |

---

**Previous:** [03-user-input.md](03-user-input.md) | **Next:** [05-conditionals.md](05-conditionals.md)
