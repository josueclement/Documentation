# System Monitoring

System-wide log analysis with `journalctl`, kernel messages with `dmesg`, resource stats with `vmstat`/`iostat`/`sar`, hardware info, and filesystem event watching with `inotifywait`.

## journalctl — System-Wide Analysis

This section covers system-wide log analysis and journal management. For per-unit log investigation (following a specific service, filtering by unit), see `systemd.md`.

### Filtering by Priority

`-p` filters by syslog priority and shows the specified level **and above** (more severe).

```bash
# Errors and above from current boot
journalctl -b -p err

# Errors from previous boot
journalctl -b -1 -p err

# Priority range (only errors through critical, not emergency)
journalctl -p err..crit

# Follow all new log entries in real time
journalctl -f
```

Priority levels (0 = most severe):

| Level | Name | Description |
|-------|------|-------------|
| 0 | `emerg` | System is unusable |
| 1 | `alert` | Immediate action needed |
| 2 | `crit` | Critical conditions |
| 3 | `err` | Error conditions |
| 4 | `warning` | Warning conditions |
| 5 | `notice` | Normal but significant |
| 6 | `info` | Informational |
| 7 | `debug` | Debug-level messages |

### Filtering by Time

```bash
journalctl --since "2025-01-01 00:00:00" --until "2025-01-01 23:59:59"
journalctl --since "1 hour ago"
journalctl --since today
journalctl --since yesterday --until today
```

### Advanced Filters

The journal is structured — every log entry has metadata fields you can filter on directly.

```bash
# Filter by syslog facility
journalctl SYSLOG_FACILITY=10         # authpriv
journalctl SYSLOG_FACILITY=0          # kern

# Filter by executable path
journalctl /usr/bin/sudo
journalctl /usr/lib/systemd/systemd

# Filter by UID
journalctl _UID=1000

# Filter by transport (how the message arrived at the journal)
journalctl _TRANSPORT=kernel          # kernel messages
journalctl _TRANSPORT=audit           # audit messages

# Combine filters (AND logic)
journalctl -b -p err _TRANSPORT=kernel
```

### Output Formats

Different formats are useful for different tasks — human reading, scripting, or sharing with others.

```bash
journalctl -o json-pretty             # full JSON (all fields)
journalctl -o short-iso               # ISO timestamps (default-like but precise)
journalctl -o cat                     # message only, no metadata
journalctl -o verbose                 # all fields, human-readable
journalctl -o export                  # binary export format (for import)
```

### Exporting Logs

```bash
# Export to text file for sharing
journalctl -b --no-pager > boot-log.txt
journalctl -b -p err -o json > errors.json
```

### Journal Disk Management

The journal can grow large. These commands manage its size.

```bash
# Check journal disk usage
journalctl --disk-usage

# Rotate current journal files (close current, start new)
sudo journalctl --rotate

# Clean by time (keep only last 7 days)
sudo journalctl --vacuum-time=7d

# Clean by size (keep at most 500MB)
sudo journalctl --vacuum-size=500M

# Clean by file count
sudo journalctl --vacuum-files=5

# Verify journal integrity
journalctl --verify
```

### Persistent Journal Configuration

By default, the journal may be volatile (in RAM only). Configure persistence and size limits in `journald.conf`.

```ini
# /etc/systemd/journald.conf
[Journal]
Storage=persistent           # auto, persistent, volatile, none
Compress=yes
SystemMaxUse=500M            # max total journal size
SystemMaxFileSize=50M        # max single file size
MaxRetentionSec=1month       # max age
MaxFileSec=1week             # rotate files this often
```

```bash
sudo systemctl restart systemd-journald
```

## dmesg (Kernel Messages)

`dmesg` displays the kernel ring buffer — messages from the kernel about hardware, drivers, and low-level events. Essential for diagnosing boot problems, hardware issues, and driver errors.

### Viewing Messages

```bash
# Show kernel messages
dmesg

# Human-readable timestamps (instead of seconds since boot)
dmesg -T

# Color output
dmesg --color=always

# Follow live (like tail -f for kernel messages)
dmesg -w
```

### Filtering

```bash
# Filter by facility (kern, daemon, etc.)
dmesg -f kern
dmesg -f daemon

# Filter by level (err, warn, info, etc.)
dmesg -l err
dmesg -l warn,err
```

### Common Searches

```bash
# USB events (device connected/disconnected)
dmesg | grep -i "usb"

# GPU-related messages
dmesg | grep -i "nvidia\|amd\|gpu"

# Out of memory killer events
dmesg | grep -i "oom"

# General errors and failures
dmesg | grep -i "error\|fail"

# Network link state changes
dmesg | grep -i "link up\|link down"
```

### Managing the Ring Buffer

```bash
# Clear the ring buffer (requires root)
sudo dmesg -C
```

## vmstat (Virtual Memory Statistics)

`vmstat` provides a snapshot of system-wide performance — processes, memory, swap, I/O, and CPU. The first output line shows averages since boot; subsequent lines show current activity.

```bash
# Single snapshot
vmstat

# Update every 2 seconds, 10 times
vmstat 2 10

# Show active/inactive memory
vmstat -a

# Show with timestamps
vmstat -t 1

# Disk statistics
vmstat -d

# Show values in MB
vmstat -S M
```

### Output Fields

| Section | Fields | Meaning |
|---------|--------|---------|
| procs | `r` | Runnable processes (waiting for CPU) |
| procs | `b` | Blocked processes (waiting for I/O) |
| memory | `swpd` | Swap used |
| memory | `free` | Free memory |
| memory | `buff` | Buffer memory |
| memory | `cache` | Cache memory |
| swap | `si` | Swap in (from disk) |
| swap | `so` | Swap out (to disk) — non-zero means memory pressure |
| io | `bi` | Blocks read from disk |
| io | `bo` | Blocks written to disk |
| system | `in` | Interrupts per second |
| system | `cs` | Context switches per second |
| cpu | `us` | User CPU time |
| cpu | `sy` | System CPU time |
| cpu | `id` | Idle time |
| cpu | `wa` | I/O wait — high means disk bottleneck |
| cpu | `st` | Stolen time (VM only) |

## iostat (I/O Statistics)

`iostat` shows CPU and disk I/O statistics. The extended mode (`-x`) reveals device utilization and latency — critical for diagnosing disk bottlenecks.

**Install:** `pacman -S sysstat`

```bash
# Basic I/O stats
iostat

# Extended stats with timestamps, every 2 seconds
iostat -x -t 2

# Show specific device
iostat -x /dev/nvme0n1

# Show in MB/s
iostat -m

# Show CPU stats only
iostat -c

# Show device stats only
iostat -d
```

### Key Fields in Extended Mode (-x)

| Field | Meaning |
|-------|---------|
| `r/s`, `w/s` | Reads/writes per second |
| `rMB/s`, `wMB/s` | Read/write throughput |
| `await` | Average wait time in ms (includes queue + service) |
| `%util` | Device utilization (100% = fully saturated) |

## sar (System Activity Reporter)

`sar` collects and reports historical system activity data. Unlike `vmstat`/`iostat` which show live data, `sar` can query saved historical data for post-mortem analysis.

**Install:** `pacman -S sysstat` and enable data collection.

```bash
# Enable data collection daemon
sudo systemctl enable --now sysstat
```

### Live Monitoring

```bash
# CPU usage (1-second interval, 5 samples)
sar -u 1 5

# Memory usage
sar -r 1 5

# Disk I/O
sar -d 1 5

# Network per-interface stats
sar -n DEV 1 5

# Socket stats
sar -n SOCK 1 5
```

### Historical Data

`sar` saves data in `/var/log/sa/`. Query past data by specifying the file and time range.

```bash
# CPU from the 13th of current month
sar -u -f /var/log/sa/sa13

# Memory between 8am and 5pm today
sar -r -s 08:00:00 -e 17:00:00
```

## Hardware Information

Commands for querying what hardware is installed — CPU, memory, PCI devices, USB, and sensors.

### CPU and Memory

```bash
# CPU info (model, cores, features, cache)
lscpu

# Detailed per-core info
cat /proc/cpuinfo

# Number of CPU cores
nproc

# Memory summary
free -h

# Detailed memory info (total, available, buffers, cache)
cat /proc/meminfo
```

### PCI and USB Devices

```bash
# PCI devices (GPUs, NICs, storage controllers)
lspci
lspci -v                      # verbose
lspci -k                      # show kernel drivers in use
lspci | grep -i vga           # find GPU

# USB devices
lsusb
lsusb -v                      # verbose
lsusb -t                      # tree view
```

### System and BIOS Info

`dmidecode` reads hardware info from the BIOS/UEFI firmware tables — useful for checking memory module details, serial numbers, and BIOS version.

```bash
# All DMI/SMBIOS data
sudo dmidecode

# Memory module details (size, speed, slots)
sudo dmidecode -t memory

# BIOS info (version, vendor, date)
sudo dmidecode -t bios

# System info (manufacturer, model, serial)
sudo dmidecode -t system
```

### Disk and Block Devices

```bash
# Block devices with filesystem info
lsblk -f

# Detailed disk info (model, firmware, features)
sudo hdparm -I /dev/sda
```

### Sensors

Hardware sensors report temperature, fan speed, and voltage. Requires a one-time detection step.

```bash
# Install sensor tools
# pacman -S lm_sensors

# Detect sensors (run once, answer yes to probes)
sudo sensors-detect

# Show sensor readings
sensors
```

### Battery (Laptops)

```bash
# Battery info via upower
upower -i /org/freedesktop/UPower/devices/battery_BAT0

# Quick battery percentage
cat /sys/class/power_supply/BAT0/capacity
```

### Kernel and OS Info

```bash
# Full kernel info
uname -a

# Kernel version only
uname -r

# Hostname, OS, kernel, architecture
hostnamectl

# Distribution info
cat /etc/os-release

# System uptime
uptime
```

## inotifywait (Filesystem Events)

`inotifywait` watches files and directories for changes in real time using the Linux inotify API. Useful for triggering rebuilds, syncs, or alerts when files change.

**Install:** `pacman -S inotify-tools`

### Basic Watching

```bash
# Watch a file for any change (blocks until event occurs)
inotifywait -m /path/to/file

# Watch a directory recursively
inotifywait -mr /path/to/dir

# Watch for specific events only
inotifywait -m -e modify,create,delete /path/to/dir
```

### Formatted Output

```bash
# Watch with timestamps and formatted output
inotifywait -mr --format '%T %w%f %e' --timefmt '%F %T' /path/to/dir
```

### Available Events

| Event | Triggered when... |
|-------|--------------------|
| `access` | File is read |
| `modify` | File content is changed |
| `attrib` | Metadata changes (permissions, timestamps) |
| `close_write` | File opened for writing is closed |
| `close_nowrite` | File opened read-only is closed |
| `open` | File is opened |
| `moved_to` | File is moved into watched directory |
| `moved_from` | File is moved out of watched directory |
| `create` | File or directory is created |
| `delete` | File or directory is deleted |
| `delete_self` | Watched file/directory is deleted |

### Practical Examples

```bash
# Auto-rebuild on source file change
inotifywait -mr -e modify --include '\.py$' ./src | while read -r path event file; do
    echo "Change detected: $file"
    make build
done

# Watch for new files in a directory
inotifywait -m -e create /var/log/ --format '%f' | while read -r file; do
    echo "New file: $file"
done

# One-shot (wait for single event then exit)
inotifywait -e modify /etc/hosts && echo "hosts file changed"
```

## Other Monitoring Tools

Additional tools for specific monitoring needs. All are available from the Arch repositories.

### Network Monitoring

```bash
# iftop — bandwidth per connection (pacman -S iftop)
sudo iftop -i enp0s3

# nethogs — bandwidth per process (pacman -S nethogs)
sudo nethogs enp0s3
```

### Disk I/O Monitoring

```bash
# iotop — disk I/O per process (pacman -S iotop)
sudo iotop
sudo iotop -o              # only show processes with active I/O
```

### All-in-One Monitors

```bash
# btop — modern TUI resource monitor (pacman -S btop)
btop

# glances — system overview with web mode (pacman -S glances)
glances

# dstat — versatile one-line resource stats (pacman -S dstat)
dstat
dstat -cdngy               # cpu, disk, net, page, sys
```

### Interactive Disk Usage

```bash
# ncdu — interactive disk usage browser (pacman -S ncdu)
ncdu /home
```

### Watching Commands

`watch` repeats a command at intervals, showing updated output. `-d` highlights what changed between updates.

```bash
# Socket stats every second
watch -n 1 "ss -s"

# Memory usage with change highlighting
watch -d free -h
```

## /proc and /sys Quick Reference

The `/proc` and `/sys` virtual filesystems expose kernel and hardware state as files. Reading them is instant and doesn't require any tools.

### System-Wide

```bash
cat /proc/cpuinfo           # CPU details
cat /proc/loadavg           # load averages (1, 5, 15 min)
cat /proc/meminfo           # detailed memory info
cat /proc/swaps             # swap devices
cat /proc/uptime            # seconds since boot
cat /proc/version           # kernel version
cat /proc/filesystems       # supported filesystem types
cat /proc/net/tcp           # TCP connections (raw)
cat /proc/net/dev           # network interface statistics
```

### Per-Process

Replace `1234` with the PID of interest.

```bash
cat /proc/1234/status       # process status (name, state, memory)
cat /proc/1234/cmdline      # command line that started the process
cat /proc/1234/environ      # environment variables
ls -la /proc/1234/fd        # open file descriptors
cat /proc/1234/maps         # memory mappings
cat /proc/1234/limits       # resource limits
```

### Kernel Tuning via /sys

```bash
# Disk I/O scheduler
cat /sys/block/sda/queue/scheduler

# CPU frequency governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```
