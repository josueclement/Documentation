# Process Management

Listing, inspecting, and controlling processes — `ps`, `kill`, signals, priority with `nice`/`renice`, job control, cgroups resource limits, and `ulimit`.

## ps (Process Status)

`ps` shows a snapshot of running processes. It supports two syntax styles: BSD (`ps aux`) and POSIX (`ps -ef`). Neither is better — they just format differently.

### Listing Processes

```bash
# List all processes (BSD style — shows USER, PID, %CPU, %MEM, etc.)
ps aux

# List all processes (POSIX style — shows UID, PID, PPID, etc.)
ps -ef

# Show process tree (parent-child hierarchy)
ps auxf
ps -ejH
```

### Filtering

```bash
# Filter by user
ps -u jo

# Filter by process name (exact match)
ps -C nginx

# Filter by PID
ps -p 1234 -o pid,ppid,user,%cpu,%mem,stat,start,time,comm
```

### Custom Output and Sorting

The `-o` flag selects which columns to display. `--sort` orders results by any column (prefix with `-` for descending).

```bash
# Show specific columns
ps -eo pid,ppid,user,%cpu,%mem,comm

# Top memory consumers
ps aux --sort=-%mem | head -20

# Top CPU consumers
ps aux --sort=-%cpu | head -20
```

### Threads

```bash
# Show all threads
ps -eLf

# Show threads for a specific process
ps -T -p 1234
```

## Process Tree Viewers

`pstree` displays a visual tree of process parent-child relationships, making it easy to see what spawned what.

```bash
# Show full process tree
pstree

# Include PIDs
pstree -p

# Show user transitions (who spawned what as whom)
pstree -u

# Show command-line arguments
pstree -a

# Tree for a specific user
pstree jo

# Tree rooted at a specific PID
pstree -p 1234
```

## Finding Processes

Various tools for locating processes by name, PID, port, or open file.

### By Name

`pgrep` finds PIDs by name pattern. `-f` matches against the full command line (not just the process name).

```bash
# Find PID by name
pgrep nginx

# Match full command line (useful for scripts with arguments)
pgrep -f "python script.py"

# Show command line with PID
pgrep -a nginx

# Show process name with PID
pgrep -l nginx

# Find processes owned by a user
pgrep -u jo

# Count matches
pgrep -c nginx
```

### By Port or File

```bash
# Find what's using a port
ss -tlnp sport = :80
fuser 80/tcp

# Find what's using a file
fuser /var/log/syslog
lsof /var/log/syslog

# Find all files opened by a process
lsof -p 1234

# Find processes with open network connections
lsof -i
lsof -i :80
lsof -i tcp
```

## Killing Processes

Terminating processes by PID, name, or pattern. Always try `SIGTERM` (graceful) before `SIGKILL` (forced).

### By PID

```bash
# Send SIGTERM (graceful shutdown — default)
kill 1234

# Send SIGKILL (force kill — cannot be caught or ignored)
kill -9 1234

# Send SIGHUP (often triggers config reload)
kill -HUP 1234
```

### By Name

```bash
# Kill all processes with a given name
killall nginx
killall -9 nginx

# Kill by pattern (extended regex on full command line)
pkill -f "python script.py"

# Kill all processes of a user
pkill -u baduser
killall -u baduser
```

### Other

```bash
# Kill processes using a specific port
fuser -k 80/tcp

# Kill a process group (use negative PID)
kill -TERM -1234
```

## Signals

Signals are messages sent to processes to trigger specific behavior. Some can be caught and handled; others (KILL, STOP) cannot.

| Signal | Number | Default Action | Common Use |
|--------|--------|---------------|-----|
| `SIGHUP` | 1 | Terminate | Reload config, terminal hangup |
| `SIGINT` | 2 | Terminate | Ctrl+C (interrupt) |
| `SIGQUIT` | 3 | Core dump | Ctrl+\\ (quit with dump) |
| `SIGKILL` | 9 | Terminate | Forced kill (cannot be caught) |
| `SIGTERM` | 15 | Terminate | Graceful shutdown (default `kill`) |
| `SIGSTOP` | 19 | Stop | Pause process (cannot be caught) |
| `SIGCONT` | 18 | Continue | Resume paused process |
| `SIGUSR1` | 10 | Terminate | User-defined (app-specific) |
| `SIGUSR2` | 12 | Terminate | User-defined (app-specific) |

```bash
# List all signals
kill -l

# Send signal by name or number (equivalent)
kill -SIGTERM 1234
kill -15 1234

# Trap signals in a shell script
trap 'echo "Caught SIGINT"; cleanup; exit' INT TERM
```

## nice & renice (Priority)

Nice values control CPU scheduling priority. Range is **-20** (highest priority) to **19** (lowest). Default is **0**. Only root can set negative (higher priority) values.

### Starting with a Priority

```bash
# Start a process with lower priority (less CPU time)
nice -n 10 tar czf backup.tar.gz /data

# Start with higher priority (requires root)
sudo nice -n -5 important-process
```

### Changing Priority of Running Processes

```bash
# Change priority of a running process
renice 10 -p 1234
sudo renice -5 -p 1234

# Change priority for all processes of a user
sudo renice 15 -u baduser

# Change priority for a process group
sudo renice 10 -g 5678

# Check current nice value
ps -o pid,ni,comm -p 1234
```

## Job Control

Job control lets you manage multiple processes in a single shell session — running commands in the background, suspending and resuming them.

```bash
# Run in background (append &)
long-command &

# List background jobs
jobs
jobs -l                    # include PID

# Suspend foreground process
# Ctrl+Z

# Resume in background
bg                         # most recent job
bg %2                      # job number 2

# Resume in foreground
fg
fg %1

# Wait for background jobs to finish
wait                       # wait for all
wait %1                    # wait for specific job
wait 1234                  # wait for PID

# Disown a job (detach from shell — survives shell exit)
disown %1
disown -a                  # all jobs

# Run immune to hangup (persists after logout, redirects output)
nohup long-command > output.log 2>&1 &

# Run detached (new session, fully independent)
setsid long-command &
```

## cgroups (Control Groups)

cgroups limit, account for, and isolate resource usage (CPU, memory, I/O) of process groups. On modern systems, systemd manages cgroups transparently — every service runs in its own cgroup.

### Systemd cgroup Management

```bash
# View cgroup hierarchy
systemd-cgls
systemd-cgls /system.slice

# Show resource usage per cgroup (like top for cgroups)
systemd-cgtop

# Set memory limit on a service
sudo systemctl set-property nginx.service MemoryMax=512M

# Set CPU quota (100% = 1 core, 200% = 2 cores)
sudo systemctl set-property myapp.service CPUQuota=50%
```

### Persistent Limits via Unit File Override

Use `systemctl edit` to create a drop-in override with resource limits.

```bash
sudo systemctl edit myapp.service
```

```ini
# Override file for cgroup limits
[Service]
MemoryMax=1G
MemoryHigh=800M
CPUQuota=200%
TasksMax=100
IOWeight=50
```

### Manual cgroup v2

```bash
# Show available cgroup controllers
cat /sys/fs/cgroup/cgroup.controllers

# Show current cgroup for a process
cat /proc/1234/cgroup

# Run a command in a scope with limits (transient, not a full service)
sudo systemd-run --scope -p MemoryMax=256M -p CPUQuota=50% ./heavy-task
```

## ulimit (Resource Limits)

`ulimit` controls per-process resource limits (open files, max processes, stack size, etc.) for the current shell session. Persistent limits are set in `/etc/security/limits.conf` or systemd unit files.

### Viewing Limits

```bash
# Show all limits (soft)
ulimit -a

# Show specific limits
ulimit -n    # max open files
ulimit -u    # max user processes
ulimit -v    # max virtual memory
ulimit -s    # stack size

# Show limits for a running process
cat /proc/1234/limits
```

### Setting Limits

```bash
# Set soft limit (current session only)
ulimit -n 65535

# Set hard limit (requires root, cannot exceed)
ulimit -Hn 65535
```

### Persistent Limits

```bash
# /etc/security/limits.conf
# jo    soft    nofile    65535
# jo    hard    nofile    65535
# @developers  soft  nproc   4096

# Systemd service limits (in unit file)
# [Service]
# LimitNOFILE=65535
# LimitNPROC=4096
```

## Real-Time Monitoring

Interactive process monitors for watching CPU, memory, and I/O usage in real time.

```bash
# top — classic process monitor
top
# Key commands in top:
# M  — sort by memory
# P  — sort by CPU
# k  — kill a process
# r  — renice
# 1  — show per-CPU stats
# c  — show full command
# q  — quit

# htop — improved top (install: pacman -S htop)
htop
# F5 — tree view
# F6 — sort by column
# F9 — kill
# F2 — setup

# btop — modern resource monitor (install: pacman -S btop)
btop
```
