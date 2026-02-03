# Text Processing

## grep - Search Text Patterns

### Basic Usage

```bash
# Search for pattern in file
grep "pattern" file.txt

# Case insensitive search
grep -i "pattern" file.txt

# Search recursively in directory
grep -r "pattern" /path/to/dir/
grep -R "pattern" /path/to/dir/

# Show line numbers
grep -n "pattern" file.txt

# Show count of matches
grep -c "pattern" file.txt

# Show only matching part
grep -o "pattern" file.txt

# Invert match (lines NOT matching)
grep -v "pattern" file.txt

# Whole word match
grep -w "word" file.txt

# Show filenames only
grep -l "pattern" *.txt

# Show filenames with no match
grep -L "pattern" *.txt
```

### Context Options

```bash
# Lines after match
grep -A 3 "pattern" file.txt

# Lines before match
grep -B 3 "pattern" file.txt

# Lines before and after
grep -C 3 "pattern" file.txt
```

### Regular Expressions

```bash
# Extended regex
grep -E "pattern1|pattern2" file.txt
egrep "pattern1|pattern2" file.txt

# Basic patterns
grep "^start" file.txt     # Lines starting with "start"
grep "end$" file.txt       # Lines ending with "end"
grep "^$" file.txt         # Empty lines
grep "." file.txt          # Any single character
grep "a.*z" file.txt       # "a" followed by anything, then "z"

# Character classes
grep "[aeiou]" file.txt    # Any vowel
grep "[0-9]" file.txt      # Any digit
grep "[A-Za-z]" file.txt   # Any letter
grep "[^0-9]" file.txt     # Not a digit

# Extended regex patterns
grep -E "colou?r" file.txt       # color or colour
grep -E "go+gle" file.txt        # gogle, google, gooogle...
grep -E "[0-9]{3}" file.txt      # Exactly 3 digits
grep -E "[0-9]{2,4}" file.txt    # 2 to 4 digits
```

### Practical Examples

```bash
# Find errors in logs
grep -i "error" /var/log/syslog

# Find IP addresses
grep -E "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" file.txt

# Find email addresses
grep -E "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt

# Exclude patterns
grep -v "^#" config.txt | grep -v "^$"  # Remove comments and empty lines
```

## sed - Stream Editor

### Basic Substitution

```bash
# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace all occurrences per line
sed 's/old/new/g' file.txt

# Replace and save to file
sed -i 's/old/new/g' file.txt

# Create backup before editing
sed -i.bak 's/old/new/g' file.txt

# Case insensitive replace
sed 's/old/new/gi' file.txt
```

### Line Operations

```bash
# Delete lines containing pattern
sed '/pattern/d' file.txt

# Delete empty lines
sed '/^$/d' file.txt

# Delete lines 5-10
sed '5,10d' file.txt

# Print specific lines (suppress others with -n)
sed -n '5p' file.txt           # Line 5
sed -n '5,10p' file.txt        # Lines 5-10
sed -n '/pattern/p' file.txt   # Lines matching pattern

# Insert line before match
sed '/pattern/i\New line' file.txt

# Append line after match
sed '/pattern/a\New line' file.txt

# Replace entire line
sed '/pattern/c\Replacement line' file.txt
```

### Multiple Operations

```bash
# Multiple substitutions
sed -e 's/a/A/g' -e 's/b/B/g' file.txt

# Using semicolon
sed 's/a/A/g; s/b/B/g' file.txt

# Using script file
sed -f script.sed file.txt
```

### Practical Examples

```bash
# Remove leading whitespace
sed 's/^[ \t]*//' file.txt

# Remove trailing whitespace
sed 's/[ \t]*$//' file.txt

# Remove both
sed 's/^[ \t]*//; s/[ \t]*$//' file.txt

# Replace extension
echo "file.txt" | sed 's/\.txt$/.md/'

# Extract between patterns
sed -n '/START/,/END/p' file.txt

# Comment out lines
sed 's/^/#/' file.txt

# Uncomment lines
sed 's/^#//' file.txt
```

## awk - Pattern Processing

### Basic Syntax

```bash
# Print entire line
awk '{print}' file.txt

# Print specific columns
awk '{print $1}' file.txt          # First column
awk '{print $1, $3}' file.txt      # First and third
awk '{print $NF}' file.txt         # Last column
awk '{print $(NF-1)}' file.txt     # Second to last
```

### Field Separator

```bash
# Default separator is whitespace
awk '{print $1}' file.txt

# Custom separator
awk -F: '{print $1}' /etc/passwd
awk -F',' '{print $1}' file.csv
awk -F'\t' '{print $1}' file.tsv

# Multiple separators
awk -F'[,:]' '{print $1}' file.txt
```

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `$0` | Entire line |
| `$1, $2...` | Field 1, 2... |
| `NF` | Number of fields |
| `NR` | Record (line) number |
| `FS` | Field separator |
| `OFS` | Output field separator |
| `RS` | Record separator |
| `ORS` | Output record separator |

```bash
# Line numbers
awk '{print NR, $0}' file.txt

# Number of fields
awk '{print NF}' file.txt

# Custom output separator
awk -F: 'BEGIN{OFS=","} {print $1, $3}' /etc/passwd
```

### Conditions and Patterns

```bash
# Pattern matching
awk '/pattern/ {print}' file.txt

# Condition
awk '$3 > 100 {print}' file.txt
awk 'NR > 1 {print}' file.txt      # Skip header

# Multiple conditions
awk '$3 > 100 && $4 < 200 {print}' file.txt

# BEGIN and END blocks
awk 'BEGIN {print "Start"} {print} END {print "End"}' file.txt
```

### Arithmetic

```bash
# Sum column
awk '{sum += $1} END {print sum}' file.txt

# Average
awk '{sum += $1; count++} END {print sum/count}' file.txt

# Math operations
awk '{print $1 + $2}' file.txt
awk '{print $1 * 100}' file.txt
```

### Practical Examples

```bash
# Extract usernames from /etc/passwd
awk -F: '{print $1}' /etc/passwd

# Users with UID > 1000
awk -F: '$3 > 1000 {print $1}' /etc/passwd

# Print specific columns with formatting
awk -F: '{printf "%-20s %s\n", $1, $7}' /etc/passwd

# Sum file sizes
ls -l | awk '{sum += $5} END {print sum}'

# Count occurrences
awk '{count[$1]++} END {for (word in count) print word, count[word]}' file.txt
```

## cut - Extract Columns

```bash
# Extract by character position
cut -c1-5 file.txt           # Characters 1-5
cut -c1,3,5 file.txt         # Characters 1, 3, 5

# Extract by delimiter and field
cut -d: -f1 /etc/passwd      # First field
cut -d: -f1,3 /etc/passwd    # Fields 1 and 3
cut -d: -f1-3 /etc/passwd    # Fields 1 through 3
cut -d',' -f2 file.csv       # Second field, comma delimiter
```

## sort - Sort Lines

```bash
# Alphabetical sort
sort file.txt

# Reverse sort
sort -r file.txt

# Numeric sort
sort -n file.txt

# Sort by column (field)
sort -k2 file.txt            # Sort by 2nd field
sort -k2 -n file.txt         # Sort by 2nd field numerically
sort -t: -k3 -n /etc/passwd  # Sort by UID

# Unique sort
sort -u file.txt

# Case insensitive
sort -f file.txt

# Human readable numbers (1K, 2M, 3G)
sort -h file.txt
```

## uniq - Remove Duplicates

```bash
# Remove adjacent duplicates (must be sorted first)
sort file.txt | uniq

# Count occurrences
sort file.txt | uniq -c

# Show only duplicates
sort file.txt | uniq -d

# Show only unique (non-duplicated)
sort file.txt | uniq -u

# Ignore case
sort file.txt | uniq -i
```

## tr - Translate Characters

```bash
# Replace characters
tr 'a' 'b' < file.txt

# Replace character sets
tr 'a-z' 'A-Z' < file.txt    # Lowercase to uppercase
tr 'A-Z' 'a-z' < file.txt    # Uppercase to lowercase

# Delete characters
tr -d '0-9' < file.txt       # Remove digits
tr -d '\n' < file.txt        # Remove newlines

# Squeeze repeated characters
tr -s ' ' < file.txt         # Multiple spaces to single

# Replace non-matching
tr -c 'a-zA-Z\n' ' ' < file.txt  # Replace non-letters with space

# Common use cases
echo "Hello World" | tr ' ' '_'     # Spaces to underscores
echo "HELLO" | tr 'A-Z' 'a-z'       # To lowercase
cat file.txt | tr -d '\r'           # Remove Windows line endings
```

## diff - Compare Files

```bash
# Compare two files
diff file1.txt file2.txt

# Side by side
diff -y file1.txt file2.txt

# Unified format (for patches)
diff -u file1.txt file2.txt

# Brief output
diff -q file1.txt file2.txt

# Compare directories
diff -r dir1/ dir2/

# Ignore whitespace
diff -w file1.txt file2.txt

# Ignore case
diff -i file1.txt file2.txt
```

## wc - Word Count

```bash
# Lines, words, characters
wc file.txt

# Lines only
wc -l file.txt

# Words only
wc -w file.txt

# Characters only
wc -c file.txt

# Bytes
wc -m file.txt

# Multiple files
wc -l *.txt
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `grep` | Search patterns | `grep -i "error" log.txt` |
| `sed` | Stream editing | `sed 's/old/new/g' file` |
| `awk` | Field processing | `awk -F: '{print $1}' passwd` |
| `cut` | Extract columns | `cut -d: -f1 passwd` |
| `sort` | Sort lines | `sort -n file.txt` |
| `uniq` | Remove duplicates | `sort file \| uniq -c` |
| `tr` | Translate chars | `tr 'a-z' 'A-Z'` |
| `diff` | Compare files | `diff -u file1 file2` |
| `wc` | Count lines/words | `wc -l file.txt` |

---

**Previous:** [09-io-redirection.md](09-io-redirection.md) | **Next:** [11-search-files-patterns.md](11-search-files-patterns.md)
