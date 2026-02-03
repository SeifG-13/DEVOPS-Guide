# Process Management in Shell Scripts

## Background Processes

### Running in Background

```bash
#!/bin/bash

# Run command in background with &
sleep 100 &
echo "Background PID: $!"

# Run multiple commands in background
command1 &
command2 &
command3 &

# Wait for all background jobs
wait

echo "All jobs completed"
```

### Disown Process

```bash
#!/bin/bash

# Start background process
long_running_command &
pid=$!

# Remove from shell's job table (survives shell exit)
disown $pid

# Or disown all background jobs
disown -a

# Disown and suppress HUP signal
disown -h $pid
```

### nohup - No Hangup

```bash
#!/bin/bash

# Run command immune to hangups
nohup long_running_command &

# With output redirection
nohup command > output.log 2>&1 &

# In script
nohup ./backup.sh > /var/log/backup.log 2>&1 &
echo "Backup started with PID $!"
```

## Job Control

### jobs Command

```bash
#!/bin/bash

# Start some background jobs
sleep 100 &
sleep 200 &
sleep 300 &

# List jobs
jobs

# Output:
# [1]   Running  sleep 100 &
# [2]-  Running  sleep 200 &
# [3]+  Running  sleep 300 &

# Job states
# + : Current job
# - : Previous job
```

### fg and bg Commands

```bash
#!/bin/bash

# Move job to foreground
fg %1        # Job number 1
fg %%        # Current job
fg %+        # Current job
fg %-        # Previous job

# Move stopped job to background
bg %1
bg %%
```

### Stopping and Resuming

```bash
# In interactive shell:
# Ctrl+Z    Stop foreground process
# Ctrl+C    Terminate foreground process

# In scripts, send signals
kill -STOP $pid    # Stop process
kill -CONT $pid    # Continue process
```

## wait Command

### Basic wait

```bash
#!/bin/bash

# Start background processes
process1 &
pid1=$!

process2 &
pid2=$!

# Wait for specific process
wait $pid1
echo "Process 1 completed with status $?"

# Wait for all background processes
wait
echo "All processes completed"
```

### wait with Exit Status

```bash
#!/bin/bash

# Run commands and capture exit status
command1 &
pid1=$!

command2 &
pid2=$!

wait $pid1
status1=$?

wait $pid2
status2=$?

if [ $status1 -eq 0 ] && [ $status2 -eq 0 ]; then
    echo "All commands succeeded"
else
    echo "Some commands failed"
fi
```

### wait -n (Bash 4.3+)

```bash
#!/bin/bash

# Wait for any background job to complete
command1 &
command2 &
command3 &

wait -n
echo "First job completed with status $?"

wait -n
echo "Second job completed"

wait
echo "All jobs completed"
```

## Subshells

### What is a Subshell?

```bash
#!/bin/bash

var="original"

# Parentheses create subshell
(
    var="modified"
    echo "Inside subshell: $var"
)

echo "Outside subshell: $var"    # Still "original"
```

### Subshell Uses

```bash
#!/bin/bash

# Change directory temporarily
(cd /tmp && ls)
# Still in original directory

# Group commands
(command1; command2; command3)

# Isolate variable changes
(
    export PATH="/custom/path:$PATH"
    run_with_custom_path
)
# PATH unchanged here

# Capture grouped output
output=$(
    echo "Line 1"
    echo "Line 2"
)
```

### Command Grouping: () vs {}

```bash
#!/bin/bash

var="outside"

# Parentheses - subshell (new process)
(
    var="subshell"
    echo $var
)
echo "After (): $var"    # outside

# Braces - current shell
{
    var="braces"
    echo $var
}
echo "After {}: $var"    # braces
```

## Process Substitution

### Reading from Process

```bash
#!/bin/bash

# <(command) - process output as file
diff <(ls dir1) <(ls dir2)

# Compare sorted files
diff <(sort file1) <(sort file2)

# Read from command output
while read -r line; do
    echo "Line: $line"
done < <(find . -name "*.txt")
```

### Writing to Process

```bash
#!/bin/bash

# >(command) - process input as file
# Tee to multiple processes
cat input.txt | tee >(grep "error" > errors.txt) >(grep "warn" > warnings.txt)

# Log while processing
command | tee >(logger -t myapp)
```

## Signals

### Common Signals

| Signal | Number | Description | Default Action |
|--------|--------|-------------|----------------|
| SIGHUP | 1 | Hangup | Terminate |
| SIGINT | 2 | Interrupt (Ctrl+C) | Terminate |
| SIGQUIT | 3 | Quit (Ctrl+\) | Core dump |
| SIGKILL | 9 | Kill (cannot catch) | Terminate |
| SIGTERM | 15 | Termination | Terminate |
| SIGSTOP | 19 | Stop (cannot catch) | Stop |
| SIGCONT | 18 | Continue | Continue |
| SIGUSR1 | 10 | User-defined 1 | Terminate |
| SIGUSR2 | 12 | User-defined 2 | Terminate |

### Sending Signals

```bash
#!/bin/bash

# Using kill
kill $pid              # Send SIGTERM (15)
kill -15 $pid          # Same as above
kill -TERM $pid        # Same as above
kill -SIGTERM $pid     # Same as above

kill -9 $pid           # Send SIGKILL (cannot be caught)
kill -KILL $pid        # Same as above

kill -0 $pid           # Check if process exists
```

### Handling Signals

```bash
#!/bin/bash

# Handle SIGINT (Ctrl+C)
handle_sigint() {
    echo "Caught SIGINT, cleaning up..."
    cleanup
    exit 1
}

trap handle_sigint SIGINT

# Handle SIGTERM
trap 'echo "Terminated"; exit 0' SIGTERM

# Handle multiple signals
trap 'cleanup; exit' SIGINT SIGTERM SIGHUP

# Ignore signal
trap '' SIGINT    # Ignore Ctrl+C

# Reset to default
trap - SIGINT     # Reset to default behavior
```

### Using SIGUSR1/SIGUSR2

```bash
#!/bin/bash

# Script that responds to user signals
status="running"

handle_usr1() {
    echo "Status: $status"
    echo "PID: $$"
    echo "Uptime: $SECONDS seconds"
}

handle_usr2() {
    status="reloading"
    echo "Reloading configuration..."
    # Reload config here
    status="running"
}

trap handle_usr1 SIGUSR1
trap handle_usr2 SIGUSR2

echo "Script running with PID $$"
echo "Send USR1 for status, USR2 to reload"

while true; do
    sleep 1
done
```

## Practical Examples

### Parallel Execution with Limit

```bash
#!/bin/bash

# Limit concurrent jobs
MAX_JOBS=4
job_count=0

# Files to process
files=(file1 file2 file3 file4 file5 file6 file7 file8)

for file in "${files[@]}"; do
    # Wait if max jobs reached
    while [ $job_count -ge $MAX_JOBS ]; do
        wait -n 2>/dev/null
        ((job_count--))
    done

    # Start job in background
    (
        echo "Processing $file..."
        sleep 2    # Simulate work
        echo "Done: $file"
    ) &

    ((job_count++))
done

# Wait for remaining jobs
wait
echo "All files processed"
```

### Process Pool

```bash
#!/bin/bash

# Create a simple process pool
POOL_SIZE=3
QUEUE_FILE="/tmp/job_queue"

# Initialize queue
> "$QUEUE_FILE"
for i in {1..10}; do
    echo "job_$i" >> "$QUEUE_FILE"
done

# Worker function
worker() {
    local worker_id=$1
    while true; do
        # Get next job (atomic)
        job=$(sed -n '1p' "$QUEUE_FILE" 2>/dev/null)
        [ -z "$job" ] && break

        sed -i '1d' "$QUEUE_FILE" 2>/dev/null

        echo "Worker $worker_id processing: $job"
        sleep 1    # Simulate work
    done
    echo "Worker $worker_id finished"
}

# Start workers
for i in $(seq 1 $POOL_SIZE); do
    worker $i &
done

wait
echo "All workers finished"
```

### Daemon Script

```bash
#!/bin/bash

PIDFILE="/var/run/mydaemon.pid"
LOGFILE="/var/log/mydaemon.log"

start_daemon() {
    if [ -f "$PIDFILE" ]; then
        if kill -0 $(cat "$PIDFILE") 2>/dev/null; then
            echo "Daemon already running"
            return 1
        fi
    fi

    # Start daemon
    nohup "$0" daemon >> "$LOGFILE" 2>&1 &
    echo $! > "$PIDFILE"
    echo "Daemon started with PID $!"
}

stop_daemon() {
    if [ -f "$PIDFILE" ]; then
        pid=$(cat "$PIDFILE")
        if kill -0 $pid 2>/dev/null; then
            kill $pid
            rm "$PIDFILE"
            echo "Daemon stopped"
        else
            echo "Daemon not running"
            rm "$PIDFILE"
        fi
    else
        echo "No PID file found"
    fi
}

daemon_loop() {
    echo "Daemon started at $(date)"
    while true; do
        # Daemon work here
        echo "Working... $(date)"
        sleep 10
    done
}

case "${1:-}" in
    start)   start_daemon ;;
    stop)    stop_daemon ;;
    restart) stop_daemon; sleep 1; start_daemon ;;
    status)
        if [ -f "$PIDFILE" ] && kill -0 $(cat "$PIDFILE") 2>/dev/null; then
            echo "Running (PID: $(cat "$PIDFILE"))"
        else
            echo "Not running"
        fi
        ;;
    daemon)  daemon_loop ;;
    *)       echo "Usage: $0 {start|stop|restart|status}" ;;
esac
```

### Watchdog Script

```bash
#!/bin/bash

WATCH_PID=""
WATCH_CMD=""
CHECK_INTERVAL=5
RESTART_DELAY=3

start_process() {
    $WATCH_CMD &
    WATCH_PID=$!
    echo "Started process with PID $WATCH_PID"
}

watchdog() {
    WATCH_CMD="$1"

    trap 'echo "Watchdog stopped"; kill $WATCH_PID 2>/dev/null; exit' INT TERM

    start_process

    while true; do
        sleep $CHECK_INTERVAL

        if ! kill -0 $WATCH_PID 2>/dev/null; then
            echo "Process died, restarting in ${RESTART_DELAY}s..."
            sleep $RESTART_DELAY
            start_process
        fi
    done
}

# Usage
watchdog "./my_service.sh"
```

### Graceful Shutdown

```bash
#!/bin/bash

# Track child processes
declare -a CHILD_PIDS

cleanup() {
    echo "Shutting down gracefully..."

    # Send SIGTERM to all children
    for pid in "${CHILD_PIDS[@]}"; do
        if kill -0 $pid 2>/dev/null; then
            echo "Stopping process $pid..."
            kill -TERM $pid
        fi
    done

    # Wait briefly for graceful shutdown
    sleep 2

    # Force kill remaining
    for pid in "${CHILD_PIDS[@]}"; do
        if kill -0 $pid 2>/dev/null; then
            echo "Force killing $pid..."
            kill -9 $pid
        fi
    done

    echo "Shutdown complete"
    exit 0
}

trap cleanup SIGINT SIGTERM

# Start child processes
./worker1.sh &
CHILD_PIDS+=($!)

./worker2.sh &
CHILD_PIDS+=($!)

./worker3.sh &
CHILD_PIDS+=($!)

echo "Started ${#CHILD_PIDS[@]} workers"
echo "PIDs: ${CHILD_PIDS[*]}"

# Wait for all children
wait
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `command &` | Run in background |
| `$!` | Last background PID |
| `jobs` | List jobs |
| `fg %n` | Foreground job n |
| `bg %n` | Background job n |
| `wait` | Wait for all jobs |
| `wait $pid` | Wait for specific PID |
| `nohup cmd &` | Run immune to hangups |
| `disown` | Remove from job table |
| `kill $pid` | Send SIGTERM |
| `kill -9 $pid` | Send SIGKILL |
| `trap 'cmd' SIG` | Handle signal |
| `(commands)` | Run in subshell |
| `<(cmd)` | Process substitution |

---

**Previous:** [12-error-handling.md](12-error-handling.md) | **Next:** [14-practical-scripts.md](14-practical-scripts.md)
