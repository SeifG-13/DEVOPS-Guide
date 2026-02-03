# Basic Linux Commands

## Navigation Commands

### pwd - Print Working Directory

```bash
pwd
# /home/username
```

### cd - Change Directory

```bash
# Go to home directory
cd
cd ~
cd $HOME

# Go to specific directory
cd /var/log

# Go to parent directory
cd ..

# Go to previous directory
cd -

# Go up two levels
cd ../..
```

### ls - List Directory Contents

```bash
# Basic listing
ls

# Long format with details
ls -l

# Show hidden files (starting with .)
ls -a

# Combined: long format + hidden
ls -la

# Human-readable sizes
ls -lh

# Sort by modification time (newest first)
ls -lt

# Sort by size (largest first)
ls -lS

# Reverse order
ls -lr

# Recursive (include subdirectories)
ls -R

# Show only directories
ls -d */

# One file per line
ls -1

# Show inode numbers
ls -i

# Colorized output
ls --color=auto
```

## File Operations

### touch - Create Empty File / Update Timestamp

```bash
# Create new empty file
touch newfile.txt

# Create multiple files
touch file1.txt file2.txt file3.txt

# Update access/modification time
touch existingfile.txt
```

### mkdir - Create Directory

```bash
# Create single directory
mkdir mydir

# Create nested directories
mkdir -p parent/child/grandchild

# Create with specific permissions
mkdir -m 755 mydir

# Create multiple directories
mkdir dir1 dir2 dir3
```

### cp - Copy Files and Directories

```bash
# Copy file
cp source.txt destination.txt

# Copy to directory
cp file.txt /path/to/directory/

# Copy multiple files to directory
cp file1.txt file2.txt /path/to/directory/

# Copy directory recursively
cp -r sourcedir/ destdir/

# Preserve permissions and timestamps
cp -p source.txt destination.txt

# Interactive (prompt before overwrite)
cp -i source.txt destination.txt

# Verbose output
cp -v source.txt destination.txt

# Combined options
cp -rpv sourcedir/ destdir/
```

### mv - Move/Rename Files

```bash
# Rename file
mv oldname.txt newname.txt

# Move file to directory
mv file.txt /path/to/directory/

# Move multiple files
mv file1.txt file2.txt /path/to/directory/

# Move directory
mv sourcedir/ /path/to/destination/

# Interactive (prompt before overwrite)
mv -i source.txt destination.txt

# Verbose output
mv -v source.txt destination.txt
```

### rm - Remove Files and Directories

```bash
# Remove file
rm file.txt

# Remove multiple files
rm file1.txt file2.txt file3.txt

# Remove directory (empty)
rmdir emptydir

# Remove directory recursively
rm -r mydir/

# Force remove (no prompt)
rm -f file.txt

# Force remove directory
rm -rf mydir/

# Interactive (prompt for each file)
rm -i file.txt

# Verbose output
rm -v file.txt

# DANGER: Never run these!
# rm -rf /
# rm -rf /*
```

## Viewing File Contents

### cat - Concatenate and Display

```bash
# Display file content
cat file.txt

# Display with line numbers
cat -n file.txt

# Display multiple files
cat file1.txt file2.txt

# Display non-printing characters
cat -A file.txt
```

### less - Page Through File

```bash
# View file with pagination
less file.txt

# Navigation in less:
# Space/f    - Next page
# b          - Previous page
# g          - Go to beginning
# G          - Go to end
# /pattern   - Search forward
# ?pattern   - Search backward
# n          - Next search result
# N          - Previous search result
# q          - Quit
```

### more - Simple Pager

```bash
# View file
more file.txt

# Navigation:
# Space - Next page
# Enter - Next line
# q     - Quit
```

### head - Display First Lines

```bash
# First 10 lines (default)
head file.txt

# First N lines
head -n 20 file.txt
head -20 file.txt

# First N bytes
head -c 100 file.txt
```

### tail - Display Last Lines

```bash
# Last 10 lines (default)
tail file.txt

# Last N lines
tail -n 20 file.txt
tail -20 file.txt

# Follow file in real-time (great for logs!)
tail -f /var/log/syslog

# Follow and show last 50 lines
tail -n 50 -f /var/log/syslog

# Follow multiple files
tail -f file1.log file2.log
```

## File Information

### file - Determine File Type

```bash
file document.pdf
# document.pdf: PDF document, version 1.4

file script.sh
# script.sh: Bourne-Again shell script, ASCII text executable
```

### stat - Detailed File Information

```bash
stat file.txt
# File: file.txt
# Size: 1234       Blocks: 8          IO Block: 4096   regular file
# Device: 801h/2049d  Inode: 123456     Links: 1
# Access: (0644/-rw-r--r--)  Uid: ( 1000/   user)   Gid: ( 1000/   user)
# Access: 2024-01-15 10:30:00.000000000 +0000
# Modify: 2024-01-15 10:25:00.000000000 +0000
# Change: 2024-01-15 10:25:00.000000000 +0000
```

### wc - Word Count

```bash
# Lines, words, characters
wc file.txt
# 100  500  3000 file.txt

# Lines only
wc -l file.txt

# Words only
wc -w file.txt

# Characters only
wc -c file.txt

# Multiple files
wc -l *.txt
```

## Other Useful Commands

### echo - Display Text

```bash
# Print text
echo "Hello World"

# Print with newline interpretation
echo -e "Line1\nLine2"

# Print without trailing newline
echo -n "No newline"

# Print environment variable
echo $HOME
echo $PATH
```

### clear - Clear Terminal

```bash
clear
# Or press Ctrl + L
```

### history - Command History

```bash
# Show command history
history

# Show last N commands
history 20

# Execute previous command
!!

# Execute command by number
!123

# Search history
Ctrl + R  # then type search term
```

### alias - Create Shortcuts

```bash
# Create alias
alias ll='ls -la'
alias cls='clear'

# Remove alias
unalias ll

# List all aliases
alias

# Make permanent - add to ~/.bashrc
echo "alias ll='ls -la'" >> ~/.bashrc
```

## Command Help

```bash
# Built-in help
command --help
ls --help

# Manual pages
man ls
man cp

# Brief description
whatis ls

# Search man pages
apropos "copy files"
man -k "copy files"

# Info pages (detailed)
info ls
```

## Quick Reference Table

| Command | Purpose | Example |
|---------|---------|---------|
| `pwd` | Print working directory | `pwd` |
| `cd` | Change directory | `cd /var/log` |
| `ls` | List contents | `ls -la` |
| `touch` | Create file | `touch file.txt` |
| `mkdir` | Create directory | `mkdir -p a/b/c` |
| `cp` | Copy | `cp -r src/ dest/` |
| `mv` | Move/Rename | `mv old.txt new.txt` |
| `rm` | Remove | `rm -rf dir/` |
| `cat` | Display file | `cat file.txt` |
| `less` | Page through file | `less file.txt` |
| `head` | First lines | `head -20 file.txt` |
| `tail` | Last lines | `tail -f log.txt` |
| `wc` | Count lines/words | `wc -l file.txt` |

---

**Previous:** [04-file-types-hierarchy.md](04-file-types-hierarchy.md) | **Next:** [06-vi-editor.md](06-vi-editor.md)
