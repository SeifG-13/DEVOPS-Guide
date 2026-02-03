# I/O Redirection

## Standard Streams

Every process has three standard streams:

| Stream | Name | File Descriptor | Default |
|--------|------|-----------------|---------|
| stdin | Standard Input | 0 | Keyboard |
| stdout | Standard Output | 1 | Terminal |
| stderr | Standard Error | 2 | Terminal |

```
            ┌─────────────────────┐
  stdin  →  │                     │  → stdout
   (0)      │      Process        │     (1)
            │                     │  → stderr
            └─────────────────────┘     (2)
```

## Output Redirection

### Redirect stdout (>)

```bash
# Redirect output to file (overwrite)
echo "Hello" > file.txt
ls -l > listing.txt

# Append output to file (>>)
echo "World" >> file.txt
ls -l >> listing.txt

# Create empty file
> emptyfile.txt
```

### Redirect stderr (2>)

```bash
# Redirect errors to file (overwrite)
ls nonexistent 2> errors.txt

# Append errors to file
ls nonexistent 2>> errors.txt

# Discard errors
ls nonexistent 2> /dev/null
```

### Redirect Both stdout and stderr

```bash
# Redirect both to different files
command > output.txt 2> errors.txt

# Redirect both to same file
command > all.txt 2>&1
command &> all.txt          # Bash shorthand

# Append both to same file
command >> all.txt 2>&1
command &>> all.txt         # Bash shorthand

# Discard all output
command > /dev/null 2>&1
command &> /dev/null        # Bash shorthand
```

### Order Matters!

```bash
# CORRECT: stderr goes where stdout goes (file)
command > file.txt 2>&1

# WRONG: stdout goes to file, stderr still goes to terminal
command 2>&1 > file.txt
```

## Input Redirection

### Redirect stdin (<)

```bash
# Read input from file
wc -l < file.txt
sort < unsorted.txt
cat < file.txt

# Combine input and output redirection
sort < unsorted.txt > sorted.txt
```

### Here Document (<<)

Send multiple lines to a command:

```bash
# Basic here document
cat << EOF
Line 1
Line 2
Line 3
EOF

# Write to file
cat << EOF > config.txt
Setting1=value1
Setting2=value2
EOF

# With variable expansion
NAME="John"
cat << EOF
Hello, $NAME
Today is $(date)
EOF

# Without variable expansion (quote delimiter)
cat << 'EOF'
Hello, $NAME
This will print $NAME literally
EOF

# Ignore leading tabs (<<-)
cat <<- EOF
	Indented content
	More indented content
EOF
```

### Here String (<<<)

Send single line to a command:

```bash
# Send string as input
grep "pattern" <<< "text to search in"

# Variable as input
VAR="Hello World"
wc -w <<< "$VAR"

# Command substitution
bc <<< "5 + 3"
```

## Pipes (|)

Connect stdout of one command to stdin of another:

```bash
# Basic pipe
ls -l | less
cat file.txt | grep "pattern"

# Chain multiple pipes
cat file.txt | grep "error" | wc -l
ps aux | grep nginx | awk '{print $2}'

# Find and process
find . -name "*.log" | xargs rm

# Common pipe patterns
ls -l | head -10           # First 10 lines
ls -l | tail -5            # Last 5 lines
history | grep "git"       # Search history
cat /etc/passwd | cut -d: -f1  # Extract usernames
```

### Pipe with stderr

```bash
# Pipe stderr (not stdout)
command 2>&1 >/dev/null | grep "error"

# Pipe both stdout and stderr
command 2>&1 | grep "pattern"
command |& grep "pattern"    # Bash shorthand
```

## tee Command

Write to file AND stdout simultaneously:

```bash
# Write to file and display
ls -l | tee listing.txt

# Append instead of overwrite
ls -l | tee -a listing.txt

# Write to multiple files
ls -l | tee file1.txt file2.txt

# In a pipeline
cat file.txt | tee backup.txt | grep "pattern"

# Capture intermediate results
ps aux | tee processes.txt | grep nginx | tee nginx.txt
```

## File Descriptors

### Custom File Descriptors

```bash
# Open file for reading (fd 3)
exec 3< input.txt
cat <&3
exec 3<&-           # Close fd 3

# Open file for writing (fd 4)
exec 4> output.txt
echo "Hello" >&4
exec 4>&-           # Close fd 4

# Open file for read/write (fd 5)
exec 5<> file.txt
```

### Duplicate File Descriptors

```bash
# Duplicate stdout to fd 3
exec 3>&1

# Duplicate stderr to fd 4
exec 4>&2

# Swap stdout and stderr
exec 3>&1 1>&2 2>&3 3>&-
```

## Process Substitution

Use output of command as a file:

```bash
# <(command) - output as input file
diff <(ls dir1) <(ls dir2)
comm <(sort file1) <(sort file2)

# >(command) - input file for command
tee >(grep error > errors.txt) >(grep warn > warnings.txt)
```

## Named Pipes (FIFO)

```bash
# Create named pipe
mkfifo mypipe

# In terminal 1: write to pipe
echo "Hello" > mypipe

# In terminal 2: read from pipe
cat < mypipe

# Remove pipe
rm mypipe
```

## /dev/null and /dev/zero

### /dev/null - Discard Output

```bash
# Discard stdout
command > /dev/null

# Discard stderr
command 2> /dev/null

# Discard all output
command &> /dev/null

# Silent execution
find / -name "file" 2>/dev/null
```

### /dev/zero - Generate Zeros

```bash
# Create file of specific size (100MB)
dd if=/dev/zero of=file.img bs=1M count=100

# Securely wipe disk (overwrite with zeros)
dd if=/dev/zero of=/dev/sdX bs=1M
```

### /dev/random and /dev/urandom

```bash
# Generate random data
head -c 32 /dev/urandom | base64

# Random password
head -c 16 /dev/urandom | base64 | tr -d '/+=' | head -c 16
```

## Practical Examples

### Logging

```bash
# Run command with logging
./script.sh > output.log 2>&1

# Log with timestamps
command 2>&1 | while read line; do
  echo "$(date '+%Y-%m-%d %H:%M:%S') $line"
done | tee logfile.txt
```

### Error Handling

```bash
# Check if command succeeded
if command > /dev/null 2>&1; then
    echo "Success"
else
    echo "Failed"
fi

# Capture stdout and stderr separately
{ stdout=$(command 2>&1 1>&3 3>&-); } 3>&1
stderr=$stdout
```

### Parallel Processing

```bash
# Process files in parallel
ls *.txt | xargs -P 4 -I {} gzip {}
```

## Quick Reference

| Operator | Description | Example |
|----------|-------------|---------|
| `>` | Redirect stdout (overwrite) | `ls > file.txt` |
| `>>` | Redirect stdout (append) | `ls >> file.txt` |
| `2>` | Redirect stderr | `cmd 2> error.txt` |
| `2>>` | Redirect stderr (append) | `cmd 2>> error.txt` |
| `&>` | Redirect both | `cmd &> all.txt` |
| `<` | Input from file | `sort < file.txt` |
| `<<` | Here document | `cat << EOF` |
| `<<<` | Here string | `grep x <<< "text"` |
| `\|` | Pipe | `ls \| grep txt` |
| `\|&` | Pipe with stderr | `cmd \|& grep err` |
| `tee` | Write to file and stdout | `ls \| tee file.txt` |
| `2>&1` | Redirect stderr to stdout | `cmd > f.txt 2>&1` |

---

**Previous:** [08-users-groups.md](08-users-groups.md) | **Next:** [10-text-processing.md](10-text-processing.md)
