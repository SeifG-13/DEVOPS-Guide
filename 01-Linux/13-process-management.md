# Process Management

## What is a Process?

A process is a running instance of a program. Each process has:
- **PID** - Process ID (unique identifier)
- **PPID** - Parent Process ID
- **UID/GID** - User/Group ownership
- **State** - Running, sleeping, stopped, zombie
- **Priority** - Scheduling priority

## Process States

| State | Symbol | Description |
|-------|--------|-------------|
| Running | R | Actively executing or in run queue |
| Sleeping | S | Waiting for event (interruptible) |
| Disk Sleep | D | Waiting for I/O (uninterruptible) |
| Stopped | T | Stopped by signal (Ctrl+Z) |
| Zombie | Z | Terminated but not reaped by parent |

## Viewing Processes

### ps - Process Status

```bash
# Current shell processes
ps

# All processes (BSD syntax)
ps aux

# All processes (Unix syntax)
ps -ef

# With threads
ps -eLf

# Forest view (tree)
ps -ef --forest
ps auxf

# Specific user
ps -u username

# Specific process
ps -p 1234

# Custom output
ps -eo pid,ppid,user,%cpu,%mem,stat,cmd

# Sort by CPU
ps aux --sort=-%cpu | head

# Sort by memory
ps aux --sort=-%mem | head
```

### ps Output Columns

| Column | Description |
|--------|-------------|
| PID | Process ID |
| PPID | Parent PID |
| USER | Process owner |
| %CPU | CPU usage |
| %MEM | Memory usage |
| VSZ | Virtual memory (KB) |
| RSS | Resident memory (KB) |
| TTY | Terminal |
| STAT | Process state |
| START | Start time |
| TIME | CPU time used |
| COMMAND | Command |

### top - Real-Time Process Viewer

```bash
# Start top
top

# Interactive commands in top:
# q      - Quit
# h      - Help
# k      - Kill process (enter PID)
# r      - Renice process
# M      - Sort by memory
# P      - Sort by CPU
# T      - Sort by time
# c      - Show full command
# 1      - Show individual CPUs
# f      - Select fields
# Space  - Refresh
```

### htop - Enhanced Process Viewer

```bash
# Install
sudo apt install htop

# Start htop
htop

# Features:
# - Mouse support
# - Color-coded
# - Easier to use
# - F1-F10 function keys
# - Horizontal/vertical scrolling
```

### Other Process Viewers

```bash
# pgrep - Find processes by name
pgrep nginx
pgrep -u root
pgrep -l nginx        # With names
pgrep -a nginx        # With full command

# pstree - Process tree
pstree
pstree -p             # With PIDs
pstree -u username    # For specific user

# pidof - Get PID of program
pidof nginx
pidof sshd
```

## Managing Processes

### kill - Send Signals

```bash
# Kill by PID
kill 1234

# Common signals
kill -15 1234         # SIGTERM (graceful termination)
kill -9 1234          # SIGKILL (force kill)
kill -1 1234          # SIGHUP (reload config)
kill -19 1234         # SIGSTOP (pause)
kill -18 1234         # SIGCONT (continue)

# List all signals
kill -l

# Kill by name
pkill nginx
pkill -u username     # All processes by user
pkill -9 -f "python script.py"  # Match full command

# killall - Kill all by name
killall nginx
killall -9 nginx

# Kill with confirmation
kill -s SIGTERM $(pgrep nginx)
```

### Common Signals

| Signal | Number | Description |
|--------|--------|-------------|
| SIGHUP | 1 | Hangup / reload config |
| SIGINT | 2 | Interrupt (Ctrl+C) |
| SIGQUIT | 3 | Quit (core dump) |
| SIGKILL | 9 | Force kill (cannot be caught) |
| SIGTERM | 15 | Graceful termination (default) |
| SIGSTOP | 19 | Stop process (cannot be caught) |
| SIGCONT | 18 | Continue stopped process |
| SIGUSR1 | 10 | User-defined signal 1 |
| SIGUSR2 | 12 | User-defined signal 2 |

## Background and Foreground

### Running in Background

```bash
# Start command in background
command &

# Redirect output
command > output.log 2>&1 &

# Disown from shell
command &
disown

# nohup - Ignore hangup signal
nohup command &
nohup command > output.log 2>&1 &
```

### Job Control

```bash
# List background jobs
jobs
jobs -l        # With PIDs

# Send running process to background
# Press Ctrl+Z first (stops the process)
bg             # Resume in background
bg %1          # Resume job 1 in background

# Bring to foreground
fg             # Last job
fg %1          # Job 1

# Kill job
kill %1
```

### screen and tmux

```bash
# screen - Terminal multiplexer
screen                    # Start new session
screen -S name           # Named session
screen -ls               # List sessions
screen -r name           # Reattach
# Ctrl+a d               # Detach

# tmux - Modern alternative
tmux                     # Start session
tmux new -s name        # Named session
tmux ls                  # List sessions
tmux attach -t name     # Reattach
# Ctrl+b d               # Detach
```

## Process Priority

### nice - Set Priority

Priority ranges from -20 (highest) to 19 (lowest). Default is 0.

```bash
# Start with nice value
nice -n 10 command       # Lower priority
nice -n -10 command      # Higher priority (needs root)

# Default nice (10)
nice command
```

### renice - Change Priority

```bash
# Change by PID
renice 10 -p 1234

# Change by user
renice 10 -u username

# Change by group
renice 10 -g groupname

# Lower priority (higher number)
sudo renice 15 -p 1234

# Higher priority (lower number, needs root)
sudo renice -5 -p 1234
```

## System Load

### Understanding Load Average

```bash
# View load average
uptime
cat /proc/loadavg

# Output: 1.50 0.75 0.50 2/200 12345
# 1-minute average: 1.50
# 5-minute average: 0.75
# 15-minute average: 0.50
# Running/Total processes: 2/200
# Last PID: 12345
```

Load average represents the average number of processes waiting to run. Compare to number of CPUs:

- Load 1.0 on 1 CPU = 100% utilized
- Load 2.0 on 4 CPUs = 50% utilized

### Memory Usage

```bash
# Memory info
free -h

# Detailed
cat /proc/meminfo

# Per-process memory
ps aux --sort=-%mem | head
```

## /proc Filesystem

```bash
# Process info
ls /proc/1234/

# Useful files
cat /proc/1234/cmdline     # Command line
cat /proc/1234/environ     # Environment
cat /proc/1234/status      # Status info
cat /proc/1234/fd/         # File descriptors
cat /proc/1234/maps        # Memory maps

# System info
cat /proc/cpuinfo
cat /proc/meminfo
cat /proc/uptime
cat /proc/loadavg
```

## Practical Examples

### Find Resource-Heavy Processes

```bash
# Top 10 CPU consumers
ps aux --sort=-%cpu | head -11

# Top 10 memory consumers
ps aux --sort=-%mem | head -11

# Watch CPU usage
watch -n 1 'ps aux --sort=-%cpu | head -11'
```

### Kill Unresponsive Processes

```bash
# Try graceful termination first
kill 1234

# If still running, force kill
kill -9 1234

# Kill all instances of a program
pkill -9 firefox
killall -9 firefox
```

### Monitor Specific Process

```bash
# Watch specific process
watch -n 1 'ps -p 1234 -o pid,ppid,%cpu,%mem,stat,cmd'

# Monitor with top
top -p 1234
```

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `ps aux` | List all processes | `ps aux` |
| `ps -ef` | List all processes | `ps -ef` |
| `top` | Interactive process viewer | `top` |
| `htop` | Enhanced process viewer | `htop` |
| `pgrep` | Find process by name | `pgrep nginx` |
| `kill` | Send signal to process | `kill -9 1234` |
| `pkill` | Kill by name | `pkill nginx` |
| `killall` | Kill all by name | `killall nginx` |
| `nice` | Start with priority | `nice -n 10 cmd` |
| `renice` | Change priority | `renice 10 -p 1234` |
| `bg` | Background job | `bg %1` |
| `fg` | Foreground job | `fg %1` |
| `jobs` | List jobs | `jobs -l` |
| `nohup` | Ignore hangup | `nohup cmd &` |

---

**Previous:** [12-file-compression-archival.md](12-file-compression-archival.md) | **Next:** [14-service-management-systemd.md](14-service-management-systemd.md)
