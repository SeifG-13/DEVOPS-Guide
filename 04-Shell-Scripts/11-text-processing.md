# Text Processing in Shell Scripts

## grep - Pattern Searching

### Basic Usage

```bash
#!/bin/bash

# Search for pattern in file
grep "pattern" file.txt

# Case insensitive
grep -i "pattern" file.txt

# Show line numbers
grep -n "pattern" file.txt

# Count matches
grep -c "pattern" file.txt

# Invert match (lines NOT matching)
grep -v "pattern" file.txt

# Whole word match
grep -w "word" file.txt

# Multiple patterns
grep -e "pattern1" -e "pattern2" file.txt
```

### Regular Expressions

```bash
#!/bin/bash

# Extended regex (-E)
grep -E "pattern1|pattern2" file.txt

# Match beginning of line
grep "^Start" file.txt

# Match end of line
grep "end$" file.txt

# Match any character
grep "h.t" file.txt      # hat, hit, hot

# Match zero or more
grep "go*d" file.txt     # gd, god, good, goood

# Character class
grep "[aeiou]" file.txt  # Any vowel
grep "[0-9]" file.txt    # Any digit
grep "[A-Za-z]" file.txt # Any letter
```

### Context Output

```bash
#!/bin/bash

# Lines before match
grep -B 3 "ERROR" log.txt

# Lines after match
grep -A 3 "ERROR" log.txt

# Lines before and after
grep -C 3 "ERROR" log.txt
```

### Recursive Search

```bash
#!/bin/bash

# Search recursively
grep -r "pattern" /path/to/dir

# Recursive with file pattern
grep -r --include="*.log" "ERROR" /var/log

# Exclude directories
grep -r --exclude-dir=".git" "TODO" .
```

### grep in Scripts

```bash
#!/bin/bash

# Check if pattern exists
if grep -q "pattern" file.txt; then
    echo "Pattern found"
fi

# Get matching lines into array
mapfile -t matches < <(grep "ERROR" log.txt)
echo "Found ${#matches[@]} errors"

# Process matches
grep "ERROR" log.txt | while read -r line; do
    echo "Processing: $line"
done
```

## sed - Stream Editor

### Basic Substitution

```bash
#!/bin/bash

# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace all occurrences per line
sed 's/old/new/g' file.txt

# Replace in place (modify file)
sed -i 's/old/new/g' file.txt

# Case insensitive replacement
sed 's/old/new/gi' file.txt

# Different delimiter
sed 's|/old/path|/new/path|g' file.txt
```

### Line Operations

```bash
#!/bin/bash

# Delete lines
sed '5d' file.txt           # Delete line 5
sed '1,5d' file.txt         # Delete lines 1-5
sed '/pattern/d' file.txt   # Delete matching lines

# Print specific lines
sed -n '5p' file.txt        # Print line 5
sed -n '1,5p' file.txt      # Print lines 1-5
sed -n '/pattern/p' file.txt # Print matching lines

# Insert/Append
sed '3i\New line' file.txt  # Insert before line 3
sed '3a\New line' file.txt  # Append after line 3

# Change line
sed '3c\Replacement line' file.txt
```

### Advanced sed

```bash
#!/bin/bash

# Multiple commands
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# Address range
sed '/start/,/end/d' file.txt          # Delete between patterns
sed '/start/,/end/s/old/new/g' file.txt # Replace between patterns

# Capture groups
sed 's/\(.*\):\(.*\)/\2:\1/' file.txt  # Swap around colon

# Extended regex
sed -E 's/([0-9]+)/[\1]/g' file.txt    # Bracket all numbers
```

### sed in Scripts

```bash
#!/bin/bash

# Update configuration
update_config() {
    local key="$1"
    local value="$2"
    local file="$3"

    if grep -q "^$key=" "$file"; then
        sed -i "s|^$key=.*|$key=$value|" "$file"
    else
        echo "$key=$value" >> "$file"
    fi
}

update_config "DATABASE_HOST" "localhost" "config.ini"
```

## awk - Pattern Processing

### Basic Usage

```bash
#!/bin/bash

# Print specific columns
awk '{print $1}' file.txt         # First column
awk '{print $1, $3}' file.txt     # First and third
awk '{print $NF}' file.txt        # Last column
awk '{print $(NF-1)}' file.txt    # Second to last

# Custom field separator
awk -F':' '{print $1}' /etc/passwd
awk -F',' '{print $1, $2}' data.csv

# Pattern matching
awk '/pattern/ {print}' file.txt
awk '/ERROR/ {print $0}' log.txt
```

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `$0` | Entire line |
| `$1, $2...` | Fields |
| `NF` | Number of fields |
| `NR` | Current line number |
| `FS` | Field separator |
| `OFS` | Output field separator |
| `RS` | Record separator |
| `ORS` | Output record separator |

```bash
#!/bin/bash

# Print with line numbers
awk '{print NR, $0}' file.txt

# Print lines with more than 3 fields
awk 'NF > 3 {print}' file.txt

# Sum a column
awk '{sum += $3} END {print sum}' file.txt

# Average
awk '{sum += $3; count++} END {print sum/count}' file.txt
```

### Conditions and Actions

```bash
#!/bin/bash

# If-else in awk
awk '{if ($3 > 100) print "High:", $0; else print "Low:", $0}' file.txt

# Multiple conditions
awk '$3 > 50 && $4 == "active" {print $1}' file.txt

# BEGIN and END
awk 'BEGIN {print "Start"} {print $0} END {print "End"}' file.txt

# Initialize variables
awk 'BEGIN {FS=":"; OFS="\t"} {print $1, $3}' /etc/passwd
```

### awk Scripts

```bash
#!/bin/bash

# Inline awk script
awk '
BEGIN {
    FS = ","
    print "Name\t\tAge\tCity"
    print "----\t\t---\t----"
}
NR > 1 {
    printf "%-15s\t%s\t%s\n", $1, $2, $3
}
END {
    print "Total records:", NR-1
}
' data.csv
```

## cut - Extract Columns

```bash
#!/bin/bash

# Extract by field
cut -f1 file.txt              # First field (tab-delimited)
cut -f1,3 file.txt            # Fields 1 and 3
cut -f2- file.txt             # Fields 2 to end
cut -d':' -f1 /etc/passwd     # Custom delimiter

# Extract by character position
cut -c1-5 file.txt            # Characters 1-5
cut -c5- file.txt             # From character 5 to end
cut -c-10 file.txt            # First 10 characters

# Extract by bytes
cut -b1-10 file.txt
```

## sort - Sorting

```bash
#!/bin/bash

# Basic sort
sort file.txt                 # Alphabetical
sort -n file.txt              # Numeric
sort -r file.txt              # Reverse

# Sort by field
sort -t',' -k2 file.csv       # Sort by second field, comma-delimited
sort -t':' -k3 -n /etc/passwd # Sort by UID

# Multiple keys
sort -t',' -k1,1 -k2,2n file.csv  # Primary then secondary

# Unique sort
sort -u file.txt

# Check if sorted
sort -c file.txt && echo "Sorted" || echo "Not sorted"
```

## uniq - Remove Duplicates

```bash
#!/bin/bash

# Remove consecutive duplicates (use with sort)
sort file.txt | uniq

# Count occurrences
sort file.txt | uniq -c

# Show only duplicates
sort file.txt | uniq -d

# Show only unique lines
sort file.txt | uniq -u

# Ignore case
sort file.txt | uniq -i

# Compare specific fields
uniq -f 1 file.txt    # Skip first field
```

## tr - Translate Characters

```bash
#!/bin/bash

# Replace characters
echo "hello" | tr 'a-z' 'A-Z'     # HELLO
echo "hello" | tr 'el' 'ip'       # hippo

# Delete characters
echo "hello 123" | tr -d '0-9'    # hello
echo "hello" | tr -d 'aeiou'      # hll

# Squeeze repeated characters
echo "heeello" | tr -s 'e'        # hello

# Replace newlines with spaces
tr '\n' ' ' < file.txt

# Remove non-printable characters
tr -cd '[:print:]' < file.txt
```

## wc - Word Count

```bash
#!/bin/bash

# Count lines, words, characters
wc file.txt

# Count only lines
wc -l file.txt

# Count only words
wc -w file.txt

# Count only characters
wc -c file.txt

# Multiple files
wc -l *.txt
```

## xargs - Build Commands

```bash
#!/bin/bash

# Basic usage
find . -name "*.txt" | xargs ls -la

# Handle spaces in filenames
find . -name "*.txt" -print0 | xargs -0 ls -la

# Limit arguments per command
echo "1 2 3 4 5" | xargs -n 2 echo

# Placeholder for argument
find . -name "*.bak" | xargs -I {} mv {} /backup/

# Parallel execution
find . -name "*.jpg" | xargs -P 4 -I {} convert {} -resize 50% {}.small.jpg

# Prompt before execution
echo "file.txt" | xargs -p rm
```

## Practical Examples

### Log Analysis

```bash
#!/bin/bash

analyze_logs() {
    local logfile="$1"

    echo "=== Log Analysis: $logfile ==="

    # Count error types
    echo -e "\nError counts:"
    grep -oE "(ERROR|WARN|INFO)" "$logfile" | sort | uniq -c | sort -rn

    # Top 10 IP addresses
    echo -e "\nTop 10 IPs:"
    awk '{print $1}' "$logfile" | sort | uniq -c | sort -rn | head -10

    # Requests per hour
    echo -e "\nRequests per hour:"
    awk '{print substr($4, 2, 14)}' "$logfile" | sort | uniq -c

    # Most requested URLs
    echo -e "\nTop 10 URLs:"
    awk '{print $7}' "$logfile" | sort | uniq -c | sort -rn | head -10
}

analyze_logs "access.log"
```

### CSV Processing

```bash
#!/bin/bash

process_csv() {
    local file="$1"

    # Skip header and process
    tail -n +2 "$file" | while IFS=',' read -r name age city salary; do
        # Calculate bonus (10% of salary)
        bonus=$(echo "$salary * 0.1" | bc)
        echo "$name: Salary=$salary, Bonus=$bonus"
    done

    # Calculate statistics
    echo -e "\nStatistics:"
    awk -F',' 'NR > 1 {
        sum += $4
        count++
        if ($4 > max) max = $4
        if (min == "" || $4 < min) min = $4
    }
    END {
        print "Total employees:", count
        print "Total salaries:", sum
        print "Average salary:", sum/count
        print "Max salary:", max
        print "Min salary:", min
    }' "$file"
}

process_csv "employees.csv"
```

### Text Report Generator

```bash
#!/bin/bash

generate_report() {
    local datafile="$1"
    local reportfile="$2"

    cat > "$reportfile" << EOF
=====================================
     DATA REPORT
     Generated: $(date)
=====================================

EOF

    # Add summary
    echo "Summary:" >> "$reportfile"
    echo "--------" >> "$reportfile"
    echo "Total lines: $(wc -l < "$datafile")" >> "$reportfile"
    echo "Unique entries: $(sort -u "$datafile" | wc -l)" >> "$reportfile"

    # Add top entries
    echo -e "\nTop 10 entries:" >> "$reportfile"
    echo "---------------" >> "$reportfile"
    sort "$datafile" | uniq -c | sort -rn | head -10 >> "$reportfile"

    echo "Report generated: $reportfile"
}

generate_report "data.txt" "report.txt"
```

### Data Transformation Pipeline

```bash
#!/bin/bash

# Transform data through multiple stages
cat raw_data.txt |
    grep -v '^#' |                # Remove comments
    grep -v '^$' |                # Remove empty lines
    tr '[:upper:]' '[:lower:]' |  # Lowercase
    sed 's/[[:space:]]\+/ /g' |   # Normalize spaces
    cut -d' ' -f1,3,5 |           # Extract fields
    sort -t' ' -k2 |              # Sort by second field
    uniq |                        # Remove duplicates
    awk '{print NR": "$0}' |      # Add line numbers
    tee output.txt                # Save and display
```

## Quick Reference

| Tool | Purpose | Example |
|------|---------|---------|
| grep | Search patterns | `grep -i "error" file` |
| sed | Stream editing | `sed 's/old/new/g' file` |
| awk | Column processing | `awk '{print $1}' file` |
| cut | Extract columns | `cut -d',' -f1 file` |
| sort | Sort lines | `sort -n file` |
| uniq | Remove duplicates | `sort file \| uniq` |
| tr | Translate chars | `tr 'a-z' 'A-Z'` |
| wc | Count lines/words | `wc -l file` |
| xargs | Build commands | `find . \| xargs rm` |

---

**Previous:** [10-file-operations.md](10-file-operations.md) | **Next:** [12-error-handling.md](12-error-handling.md)
