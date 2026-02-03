# Introduction to Shell Scripting

## What is Shell Scripting?

Shell scripting is writing a series of commands in a file to be executed by the shell (command interpreter).

```
┌─────────────────────────────────────────────────────────────────┐
│                     Shell Script Execution                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Script File ──► Shell Interpreter ──► Commands ──► Output     │
│   (commands)        (bash/sh/zsh)        (execute)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Why Use Shell Scripts?

| Purpose | Description |
|---------|-------------|
| Automation | Automate repetitive tasks |
| System Admin | Manage users, backups, services |
| CI/CD | Build and deployment pipelines |
| Batch Processing | Process multiple files |
| Task Scheduling | Cron jobs and scheduled tasks |

## Types of Shells

| Shell | Path | Description |
|-------|------|-------------|
| Bash | `/bin/bash` | Bourne Again Shell (most common) |
| sh | `/bin/sh` | Original Bourne Shell |
| zsh | `/bin/zsh` | Z Shell (macOS default) |
| ksh | `/bin/ksh` | Korn Shell |
| dash | `/bin/dash` | Debian Almquist Shell |

```bash
# Check current shell
echo $SHELL

# List available shells
cat /etc/shells

# Check bash version
bash --version
```

## The Shebang (#!)

The shebang tells the system which interpreter to use.

```bash
#!/bin/bash
# This script will be executed with bash

#!/bin/sh
# This script will be executed with sh

#!/usr/bin/env bash
# Portable way - finds bash in PATH
```

### Why Use `#!/usr/bin/env bash`?

```bash
#!/usr/bin/env bash
# More portable - works across different systems
# bash might be in /bin/bash, /usr/bin/bash, or elsewhere
```

## Creating Your First Script

### Step 1: Create the Script File

```bash
# Using any text editor
vim myscript.sh
nano myscript.sh
```

### Step 2: Write the Script

```bash
#!/bin/bash
# My first shell script
# Author: Your Name
# Date: 2024-01-01

echo "Hello, World!"
echo "Today is $(date)"
echo "Current user: $USER"
```

### Step 3: Make it Executable

```bash
# Add execute permission
chmod +x myscript.sh

# Or use numeric mode
chmod 755 myscript.sh
```

### Step 4: Run the Script

```bash
# Method 1: Direct execution (requires execute permission)
./myscript.sh

# Method 2: Explicit interpreter (no execute permission needed)
bash myscript.sh

# Method 3: Source the script (runs in current shell)
source myscript.sh
. myscript.sh
```

## Script File Naming

| Convention | Example | Description |
|------------|---------|-------------|
| `.sh` extension | `backup.sh` | Common convention |
| No extension | `backup` | Also valid |
| Descriptive names | `daily-backup.sh` | Best practice |

```bash
# Good naming
backup-database.sh
deploy-app.sh
setup-environment.sh

# Avoid
script1.sh
test.sh
run.sh
```

## Script Structure

```bash
#!/bin/bash
#
# Script Name: example.sh
# Description: Brief description of what the script does
# Author: Your Name
# Date: 2024-01-01
# Version: 1.0
#

# Exit on error
set -e

# Variables
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_FILE="/var/log/myscript.log"

# Functions
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Main script
main() {
    log_message "Script started"

    # Your code here
    echo "Running main script..."

    log_message "Script completed"
}

# Run main function
main "$@"
```

## Comments

```bash
#!/bin/bash

# Single line comment

echo "Hello"  # Inline comment

: '
This is a
multi-line comment
using colon and quotes
'

<<'COMMENT'
Another way to do
multi-line comments
using heredoc
COMMENT
```

## Running Scripts in Different Ways

### Current Directory

```bash
# Must use ./ for scripts in current directory
./myscript.sh
```

### Absolute Path

```bash
/home/user/scripts/myscript.sh
```

### From PATH

```bash
# Add script directory to PATH
export PATH="$PATH:/home/user/scripts"

# Now run without path
myscript.sh
```

### Source vs Execute

```bash
# Execute - runs in subshell (new process)
./myscript.sh
bash myscript.sh

# Source - runs in current shell
source myscript.sh
. myscript.sh
```

| Method | Variables | Current Directory | Process |
|--------|-----------|-------------------|---------|
| Execute | Not inherited | Changes not kept | New subshell |
| Source | Inherited | Changes kept | Current shell |

## Script Exit Codes

```bash
#!/bin/bash

# Exit codes indicate success or failure
exit 0    # Success
exit 1    # General error
exit 2    # Misuse of command

# Check last command's exit code
echo $?
```

| Exit Code | Meaning |
|-----------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Misuse of shell command |
| 126 | Permission denied |
| 127 | Command not found |
| 128+n | Fatal error signal n |

## Basic Script Examples

### System Information Script

```bash
#!/bin/bash
# system-info.sh - Display system information

echo "=== System Information ==="
echo "Hostname: $(hostname)"
echo "OS: $(uname -s)"
echo "Kernel: $(uname -r)"
echo "Architecture: $(uname -m)"
echo "Uptime: $(uptime -p)"
echo "Date: $(date)"
echo "User: $USER"
echo "Home: $HOME"
echo "Shell: $SHELL"
```

### Simple Backup Script

```bash
#!/bin/bash
# backup.sh - Simple backup script

SOURCE="/home/$USER/documents"
DEST="/backup"
DATE=$(date +%Y%m%d)

echo "Starting backup..."
tar -czf "$DEST/backup-$DATE.tar.gz" "$SOURCE"
echo "Backup completed: $DEST/backup-$DATE.tar.gz"
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `#!/bin/bash` | Bash shebang |
| `chmod +x script.sh` | Make executable |
| `./script.sh` | Run script |
| `bash script.sh` | Run with bash |
| `source script.sh` | Source script |
| `echo $?` | Check exit code |
| `#` | Comment |

---

**Next:** [02-variables.md](02-variables.md)
