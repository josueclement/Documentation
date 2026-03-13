# Process Management

## ps (Process Status)

```bash
# List all processes (BSD style)
ps aux

# List all processes (POSIX style)
ps -ef

# Show process tree
ps auxf
ps -ejH

# Filter by user
ps -u jo

# Filter by process name
ps aux | grep nginx
ps -C nginx               # exact name match

# Show specific columns
ps -eo pid,ppid,user,%cpu,%mem,comm
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%mem

# Top memory consumers
ps aux --sort=-%mem | head -20

# Top CPU consumers
ps aux --sort=-%cpu | head -20

# Show threads
ps -eLf
ps -T -p 1234             # threads for PID 1234

# Show process by PID
ps -p 1234 -o pid,ppid,user,%cpu,%mem,stat,start,time,comm

# Show all processes for a command
ps -C nginx -o pid,ppid,%cpu,%mem,stat,start,cmd

# Process tree for a specific PID
ps --forest -p $(pgrep -P 1234) 1234
```

## Process Tree Viewers

```bash
# pstree — show process tree
pstree
pstree -p                  # include PIDs
pstree -u                  # show user transitions
pstree -a                  # show command-line arguments
pstree jo                  # tree for a specific user
pstree -p 1234             # tree rooted at PID
```

## Finding Processes

```bash
# Find PID by name
pgrep nginx
pgrep -f "python script.py"    # match full command line

# Find with details
pgrep -a nginx                  # show command line
pgrep -l nginx                  # show name
pgrep -u jo                     # processes owned by user
pgrep -c nginx                  # count matches

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

```bash
# Kill by PID
kill 1234                  # sends SIGTERM (graceful)
kill -9 1234               # sends SIGKILL (force)
kill -HUP 1234             # sends SIGHUP (reload)

# Kill by name
killall nginx
killall -9 nginx

# Kill by pattern (extended regex on full command line)
pkill -f "python script.py"

# Kill all processes of a user
pkill -u baduser
killall -u baduser

# Kill processes using a specific port
fuser -k 80/tcp

# Kill process group
kill -TERM -1234           # negative PID = process group

# Interactive kill (shows what would be killed)
pkill -i nginx
```

## Signals

| Signal | Number | Default Action | Use |
|--------|--------|---------------|-----|
| `SIGHUP` | 1 | Terminate | Reload config, hangup |
| `SIGINT` | 2 | Terminate | Ctrl+C |
| `SIGQUIT` | 3 | Core dump | Ctrl+\\ |
| `SIGKILL` | 9 | Terminate | Forced kill (cannot be caught) |
| `SIGTERM` | 15 | Terminate | Graceful shutdown (default) |
| `SIGSTOP` | 19 | Stop | Pause process (cannot be caught) |
| `SIGCONT` | 18 | Continue | Resume paused process |
| `SIGUSR1` | 10 | Terminate | User-defined |
| `SIGUSR2` | 12 | Terminate | User-defined |

```bash
# List all signals
kill -l

# Send signal by name or number
kill -SIGTERM 1234
kill -15 1234

# Trap signals in a script
trap 'echo "Caught SIGINT"; cleanup; exit' INT TERM
```

## nice & renice (Priority)

Nice values range from **-20** (highest priority) to **19** (lowest priority). Default is **0**.

```bash
# Start a process with lower priority
nice -n 10 tar czf backup.tar.gz /data

# Start with higher priority (requires root)
sudo nice -n -5 important-process

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

```bash
# Run in background
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

# Disown a job (detach from shell)
disown %1
disown -a                  # all jobs

# Run immune to hangup (persists after logout)
nohup long-command > output.log 2>&1 &

# Run detached
setsid long-command &
```

## cgroups (Control Groups)

### Systemd cgroup management

```bash
# View cgroup hierarchy
systemd-cgls
systemd-cgls /system.slice

# Show resource usage per cgroup
systemd-cgtop

# Set memory limit on a service
sudo systemctl set-property nginx.service MemoryMax=512M

# Set CPU quota (100% = 1 core)
sudo systemctl set-property myapp.service CPUQuota=50%

# Limit via unit file override
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
# Show cgroup controllers
cat /sys/fs/cgroup/cgroup.controllers

# Show current cgroup for a process
cat /proc/1234/cgroup

# Run a command in a scope with limits
sudo systemd-run --scope -p MemoryMax=256M -p CPUQuota=50% ./heavy-task
```

## ulimit (Resource Limits)

```bash
# Show all limits
ulimit -a

# Show specific limits
ulimit -n    # max open files
ulimit -u    # max user processes
ulimit -v    # max virtual memory
ulimit -s    # stack size

# Set soft limit (current session)
ulimit -n 65535

# Set hard limit (requires root)
ulimit -Hn 65535

# Persistent limits — edit /etc/security/limits.conf
# jo    soft    nofile    65535
# jo    hard    nofile    65535
# @developers  soft  nproc   4096

# Systemd service limits (in unit file)
# [Service]
# LimitNOFILE=65535
# LimitNPROC=4096

# Show limits for a running process
cat /proc/1234/limits
```

## Real-Time Monitoring

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
