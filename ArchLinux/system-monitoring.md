# System Monitoring

## journalctl — System-Wide Analysis

```bash
# Show all logs from current boot
journalctl -b

# Errors and above from current boot
journalctl -b -p err

# Errors from previous boot
journalctl -b -1 -p err

# Follow all new log entries
journalctl -f

# Filter by priority
journalctl -p emerg      # 0 — system is unusable
journalctl -p alert      # 1 — immediate action needed
journalctl -p crit       # 2 — critical
journalctl -p err        # 3 — errors
journalctl -p warning    # 4 — warnings
journalctl -p notice     # 5 — normal but significant
journalctl -p info       # 6 — informational
journalctl -p debug      # 7 — debug

# Priority range
journalctl -p err..crit

# Filter by time
journalctl --since "2025-01-01 00:00:00" --until "2025-01-01 23:59:59"
journalctl --since "1 hour ago"
journalctl --since today
journalctl --since yesterday --until today

# Filter by syslog facility
journalctl SYSLOG_FACILITY=10         # authpriv
journalctl SYSLOG_FACILITY=0          # kern

# Filter by executable path
journalctl /usr/bin/sudo
journalctl /usr/lib/systemd/systemd

# Filter by UID
journalctl _UID=1000

# Filter by transport
journalctl _TRANSPORT=kernel          # kernel messages
journalctl _TRANSPORT=audit           # audit messages

# Combine filters (AND logic)
journalctl -b -p err _TRANSPORT=kernel

# Output formats
journalctl -o json-pretty             # full JSON
journalctl -o short-iso               # ISO timestamps
journalctl -o cat                     # message only, no metadata
journalctl -o verbose                 # all fields
journalctl -o export                  # binary export format

# Export logs for sharing
journalctl -b --no-pager > boot-log.txt
journalctl -b -p err -o json > errors.json

# Journal disk usage
journalctl --disk-usage

# Rotate and clean journal
sudo journalctl --rotate
sudo journalctl --vacuum-time=7d      # keep 7 days
sudo journalctl --vacuum-size=500M    # keep 500MB max
sudo journalctl --vacuum-files=5      # keep 5 files max

# Verify journal integrity
journalctl --verify
```

### Persistent Journal Configuration

```ini
# /etc/systemd/journald.conf
[Journal]
Storage=persistent           # auto, persistent, volatile, none
Compress=yes
SystemMaxUse=500M
SystemMaxFileSize=50M
MaxRetentionSec=1month
MaxFileSec=1week
```

```bash
sudo systemctl restart systemd-journald
```

## dmesg (Kernel Messages)

```bash
# Show kernel messages
dmesg

# Human-readable timestamps
dmesg -T

# Color output
dmesg --color=always

# Filter by facility
dmesg -f kern
dmesg -f daemon

# Filter by level
dmesg -l err
dmesg -l warn,err

# Follow (live)
dmesg -w

# Clear the ring buffer
sudo dmesg -C

# Show only since last boot
dmesg -T | grep -i "error\|fail\|warn"

# Common things to look for:
dmesg | grep -i "usb"          # USB events
dmesg | grep -i "nvidia\|amd\|gpu"
dmesg | grep -i "oom"          # out of memory killer
dmesg | grep -i "error\|fail"
dmesg | grep -i "link up\|link down"   # network link changes
```

## vmstat (Virtual Memory Statistics)

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

# Show in MB
vmstat -S M

# Output fields:
# procs: r=running, b=blocked
# memory: swpd=swap used, free, buff, cache
# swap: si=swap in, so=swap out
# io: bi=blocks in, bo=blocks out
# system: in=interrupts, cs=context switches
# cpu: us=user, sy=system, id=idle, wa=iowait, st=stolen
```

## iostat (I/O Statistics)

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

# Key fields in extended mode (-x):
# r/s, w/s    — reads/writes per second
# rMB/s, wMB/s — read/write throughput
# await       — average wait time (ms)
# %util       — device utilization (100% = saturated)
```

## sar (System Activity Reporter)

**Install:** `pacman -S sysstat` and enable `sysstat.service`

```bash
# Enable data collection
sudo systemctl enable --now sysstat

# CPU usage
sar -u 1 5                    # every 1 second, 5 samples

# Memory usage
sar -r 1 5

# Disk I/O
sar -d 1 5

# Network
sar -n DEV 1 5                # per-interface stats
sar -n SOCK 1 5               # socket stats

# Historical data (from saved logs)
sar -u -f /var/log/sa/sa13    # CPU from 13th of current month
sar -r -s 08:00:00 -e 17:00:00    # memory, 8am to 5pm today
```

## Hardware Information

```bash
# CPU info
lscpu
cat /proc/cpuinfo
nproc                          # number of CPU cores

# Memory info
free -h
cat /proc/meminfo

# PCI devices (GPUs, NICs, controllers)
lspci
lspci -v                      # verbose
lspci -k                      # show kernel drivers
lspci | grep -i vga           # GPU

# USB devices
lsusb
lsusb -v                      # verbose
lsusb -t                      # tree view

# Block devices
lsblk -f
sudo hdparm -I /dev/sda       # detailed disk info

# DMI/BIOS data
sudo dmidecode
sudo dmidecode -t memory       # memory module details
sudo dmidecode -t bios         # BIOS info
sudo dmidecode -t system       # system info

# Sensors (install: pacman -S lm_sensors)
sudo sensors-detect            # first-time setup
sensors

# Battery (laptops)
upower -i /org/freedesktop/UPower/devices/battery_BAT0
cat /sys/class/power_supply/BAT0/capacity

# Kernel and OS info
uname -a                      # full kernel info
uname -r                      # kernel version
hostnamectl                   # hostname, OS, kernel, arch
cat /etc/os-release           # distribution info

# System uptime
uptime
```

## inotifywait (Filesystem Events)

**Install:** `pacman -S inotify-tools`

```bash
# Watch a file for changes
inotifywait -m /path/to/file

# Watch a directory recursively
inotifywait -mr /path/to/dir

# Watch for specific events
inotifywait -m -e modify,create,delete /path/to/dir

# Watch recursively with format string
inotifywait -mr --format '%T %w%f %e' --timefmt '%F %T' /path/to/dir

# Available events:
# access, modify, attrib, close_write, close_nowrite, close,
# open, moved_to, moved_from, move, create, delete, delete_self,
# unmount

# Auto-rebuild on file change
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

```bash
# iftop — network bandwidth per connection (pacman -S iftop)
sudo iftop -i enp0s3

# nethogs — bandwidth per process (pacman -S nethogs)
sudo nethogs enp0s3

# iotop — disk I/O per process (pacman -S iotop)
sudo iotop
sudo iotop -o              # only show active I/O

# ncdu — interactive disk usage (pacman -S ncdu)
ncdu /home

# btop — all-in-one (pacman -S btop)
btop

# glances — system overview (pacman -S glances)
glances

# dstat — versatile resource stats (pacman -S dstat)
dstat
dstat -cdngy               # cpu, disk, net, page, sys

# watch — repeat a command
watch -n 1 "ss -s"         # socket stats every second
watch -d free -h           # highlight changes in memory
```

## /proc and /sys Quick Reference

```bash
# CPU
cat /proc/cpuinfo
cat /proc/loadavg

# Memory
cat /proc/meminfo
cat /proc/swaps

# Per-process info
cat /proc/1234/status       # process status
cat /proc/1234/cmdline      # command line
cat /proc/1234/environ      # environment variables
cat /proc/1234/fd/          # open file descriptors
cat /proc/1234/maps         # memory mappings
ls -la /proc/1234/fd        # list open files

# Network
cat /proc/net/tcp
cat /proc/net/dev

# System
cat /proc/uptime
cat /proc/version
cat /proc/filesystems

# Kernel tuning via /sys
cat /sys/block/sda/queue/scheduler
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```
