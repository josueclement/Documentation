# General Administration

Date/time and locale settings, cron syntax reference, kernel module management, `tmpfiles.d`, power management, hostname config, and assorted one-liners.

## Date & Time

Linux tracks two clocks: the **system clock** (in-memory, managed by the kernel) and the **hardware clock** (RTC, persists across reboots). `timedatectl` manages both plus NTP synchronization.

### Viewing Date and Time

```bash
# Show current date and time
date
date +"%Y-%m-%d %H:%M:%S"
date +%F                       # ISO date: 2025-01-15

# Show in UTC
date -u

# Full status (timezone, NTP sync, RTC)
timedatectl
```

### Setting Timezone and NTP

```bash
# List available timezones
timedatectl list-timezones
timedatectl list-timezones | grep -i europe

# Set timezone
sudo timedatectl set-timezone Europe/Paris

# Enable NTP synchronization (automatic time sync)
sudo timedatectl set-ntp true

# NTP status
timedatectl show-timesync
timedatectl timesync-status
```

### NTP Configuration

```ini
# /etc/systemd/timesyncd.conf
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org
```

### Manual Clock Management

```bash
# Set date manually (usually not needed with NTP)
sudo date -s "2025-01-15 12:00:00"

# Hardware clock
sudo hwclock --show
sudo hwclock --systohc        # sync system clock → hardware clock
```

### Date Arithmetic

Useful for scripts that need relative dates.

```bash
date -d "+7 days"
date -d "last monday"
date -d "2 hours ago"
date -d "2025-01-01 + 90 days" +%F
```

### Epoch Conversions

Unix epoch is seconds since 1970-01-01 00:00:00 UTC.

```bash
# Current epoch timestamp
date +%s

# Convert epoch to human-readable
date -d @1700000000
```

## Locale

Locale settings control language, date/number formatting, sorting, and character encoding.

```bash
# Show current locale settings
locale

# List available locales
locale -a

# Generate locales (uncomment desired lines in /etc/locale.gen first)
sudo vim /etc/locale.gen
sudo locale-gen

# Set system locale
sudo localectl set-locale LANG=en_US.UTF-8
sudo localectl set-keymap us

# Show locale settings
localectl status

# Per-session override
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

## Cron Syntax (systemd Timers Preferred)

Traditional cron format for reference. On Arch, **systemd timers** are preferred — they integrate with the journal, support calendar expressions, and don't need a separate daemon. See `systemd.md` for timer unit examples.

### Cron Expression Format

```
┌───────── minute (0-59)
│ ┌─────── hour (0-23)
│ │ ┌───── day of month (1-31)
│ │ │ ┌─── month (1-12)
│ │ │ │ ┌─ day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command
```

### Common Expressions

| Expression | Meaning |
|-----------|---------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour (at minute 0) |
| `0 0 * * *` | Daily at midnight |
| `0 2 * * *` | Daily at 2:00 AM |
| `0 0 * * 0` | Weekly on Sunday at midnight |
| `0 0 1 * *` | First day of every month at midnight |
| `0 0 1 1 *` | January 1st at midnight |
| `30 8 * * 1-5` | 8:30 AM, Monday to Friday |
| `0 */6 * * *` | Every 6 hours |

### Using cronie

If you prefer cron over systemd timers:

```bash
# Install and enable cronie
# pacman -S cronie
sudo systemctl enable --now cronie

# Edit your crontab
crontab -e

# List your crontab
crontab -l

# Remove your crontab
crontab -r
```

## Kernel Modules

Kernel modules are loadable pieces of kernel code — drivers, filesystems, etc. They can be loaded and unloaded at runtime without rebooting.

### Viewing Modules

```bash
# List all currently loaded modules
lsmod

# Show info about a module (description, parameters, dependencies)
modinfo nvidia

# Show only parameters
modinfo -p nvidia
```

### Loading and Unloading

```bash
# Load a module (also loads dependencies)
sudo modprobe nvidia

# Load with parameters
sudo modprobe snd_hda_intel power_save=1

# Unload a module (fails if in use)
sudo modprobe -r nvidia

# Show module dependencies
modprobe --show-depends nvidia

# Force reload
sudo modprobe -r nvidia && sudo modprobe nvidia
```

### Persistent Configuration

```bash
# Blacklist a module (prevent it from loading — e.g., replace nouveau with nvidia)
# /etc/modprobe.d/blacklist.conf
# blacklist nouveau

# Auto-load modules at boot
# /etc/modules-load.d/custom.conf
# nvidia
# vhost_net

# Set persistent module options
# /etc/modprobe.d/nvidia.conf
# options nvidia NVreg_UsePageAttributeTable=1
```

## tmpfiles.d (Temporary File Management)

`systemd-tmpfiles` manages creation, deletion, and cleaning of temporary and volatile files and directories. It runs at boot and can be triggered manually.

### Configuration Format

```ini
# /etc/tmpfiles.d/custom.conf

# Type  Path                   Mode  Owner  Group  Age  Argument

# Create directory with specific permissions
d /run/myapp 0755 myuser mygroup -

# Create directory and clean files older than 7 days
D /tmp/myapp 0755 root root 7d

# Create a symlink
L /etc/myapp.conf - - - - /opt/myapp/config/myapp.conf

# Create file with content
f /run/myapp/pid 0644 myuser mygroup - ""

# Remove path recursively on boot
R /tmp/old-data -
```

### Common Types

| Type | Action |
|------|--------|
| `d` | Create directory |
| `D` | Create directory + clean contents by age |
| `f` | Create file (if it doesn't exist) |
| `F` | Create file + truncate if it exists |
| `L` | Create symlink |
| `R` | Remove recursively |
| `r` | Remove |
| `z` | Adjust permissions on existing path |
| `Z` | Adjust permissions recursively |

### Applying

```bash
# Apply tmpfiles configuration (create entries)
sudo systemd-tmpfiles --create

# Clean according to age rules
sudo systemd-tmpfiles --clean

# Remove entries marked with R/r
sudo systemd-tmpfiles --remove

# Dry run (preview without applying)
sudo systemd-tmpfiles --create --dry-run
```

## Power Management

Commands for shutdown, reboot, suspend, hibernate, and related power settings.

### Shutdown and Reboot

```bash
# Shutdown immediately
sudo poweroff
sudo shutdown now

# Scheduled shutdown
sudo shutdown -h +10           # shutdown in 10 minutes
sudo shutdown -h 22:00         # shutdown at 10 PM
sudo shutdown -c                # cancel scheduled shutdown

# Reboot
sudo reboot
sudo shutdown -r now
sudo systemctl reboot
```

### Suspend and Hibernate

Suspend saves state to RAM (fast resume, uses power). Hibernate saves state to swap (slow resume, no power). Hybrid combines both.

```bash
# Suspend (sleep)
sudo systemctl suspend

# Hibernate (requires swap >= RAM and configured resume= kernel param)
sudo systemctl hibernate

# Hybrid sleep (suspend + hibernate image for safety)
sudo systemctl hybrid-sleep

# Suspend then hibernate after timeout
sudo systemctl suspend-then-hibernate
```

### logind Configuration

`logind.conf` controls what happens when you close the laptop lid, press the power button, or the system goes idle.

```ini
# /etc/systemd/logind.conf
# HandleLidSwitch=suspend
# HandlePowerKey=poweroff
# IdleAction=suspend
# IdleActionSec=30min
```

```bash
# Reload logind config
sudo systemctl restart systemd-logind
```

### Inhibiting Sleep

Prevent the system from sleeping while a long-running task is active.

```bash
# Inhibit sleep during a download
systemd-inhibit --what=sleep --who="my-script" --why="downloading" wget https://example.com/big-file

# Check active inhibitors
systemd-inhibit --list
```

### CPU Frequency and Power Optimization

```bash
# Check current CPU frequency governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set to performance mode (all cores)
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# TLP for laptops (comprehensive power management)
# pacman -S tlp
sudo systemctl enable --now tlp
sudo tlp-stat -b               # battery info
```

## Hostname

The hostname identifies your machine on the network.

```bash
# Show hostname
hostname
hostnamectl

# Set hostname (persistent)
sudo hostnamectl set-hostname myarch

# Set pretty hostname (can include spaces, shown in UIs)
sudo hostnamectl set-hostname "My Arch Workstation" --pretty
```

Relevant files:
- `/etc/hostname` — static hostname
- `/etc/hosts` — should include `127.0.1.1 myarch` matching the hostname

## Scheduled Tasks Comparison

| Feature | cron | systemd timer |
|---------|------|---------------|
| Logging | syslog/mail | journalctl |
| Dependencies | None | Full systemd dependency system |
| Missed runs | Lost (not retried) | `Persistent=true` catches up |
| Random delay | No | `RandomizedDelaySec` (avoid thundering herd) |
| Resource limits | No | Full cgroup support |
| Calendar syntax | `5 fields (min hr dom mon dow)` | `OnCalendar=` (flexible) |
| Boot-relative | No | `OnBootSec=`, `OnStartupSec=` |

## Miscellaneous Utilities

Useful commands and patterns that don't fit neatly into other categories.

### Watching and Repeating

```bash
# Repeat a command every 2 seconds
watch -n 2 "df -h"

# Highlight changes between updates
watch -d "free -h"
```

### Interactive Prompts

```bash
# Answer yes to all prompts
yes | pacman -S package

# Answer no
yes n | command
```

### Priority and Throttling

```bash
# Run at low CPU priority
nice -n 19 make -j$(nproc)

# Run at low I/O priority (idle class — only runs when disk is free)
ionice -c3 rsync -a src/ dest/
```

### Timeouts and Retry

```bash
# Kill command if it takes longer than 30 seconds
timeout 30 wget https://example.com/file.tar.gz

# Send SIGKILL instead of SIGTERM after timeout
timeout --signal=KILL 10 command

# Repeat command until it succeeds
until ping -c1 google.com; do sleep 1; done
```

### One-Shot Scheduling

```bash
# Run command 30 minutes from now (via systemd)
sudo systemd-run --on-active=30min /usr/local/bin/task.sh

# Run at a specific time (requires: pacman -S at)
echo "reboot" | sudo at 02:00
```

### Bulk File Operations

```bash
# Rename files with regex (requires: pacman -S perl-rename)
rename 's/\.jpeg$/\.jpg/' *.jpeg
rename 'y/A-Z/a-z/' *              # lowercase all filenames
```

### Random Data and Checksums

```bash
# Generate random string
head -c 32 /dev/urandom | base64

# Generate random hex
openssl rand -hex 16

# Checksum files
sha256sum file.txt
sha256sum -c checksums.txt         # verify against checksum file
md5sum file.txt
```

### File Transfer and Serving

```bash
# Copy with progress bar
rsync -ah --progress src dest

# Quick HTTP server in current directory
python -m http.server 8000
```

### Encoding

```bash
# Base64 encode/decode
echo "hello" | base64
echo "aGVsbG8K" | base64 -d

# URL encode/decode
python -c "import urllib.parse; print(urllib.parse.quote('hello world'))"
python -c "import urllib.parse; print(urllib.parse.unquote('hello%20world'))"

# JSON pretty-print
cat data.json | python -m json.tool
cat data.json | jq .
```

### ISO and USB

```bash
# Create ISO from directory
mkisofs -o image.iso /path/to/dir

# Write ISO to USB drive (replace /dev/sdb with correct device!)
sudo dd if=archlinux.iso of=/dev/sdb bs=4M status=progress oflag=sync
```

### Terminal Multiplexing

`tmux` lets you run persistent terminal sessions that survive disconnection.

```bash
# pacman -S tmux
tmux                               # new session
tmux new -s name                   # named session
tmux attach -t name                # reattach to session
tmux ls                            # list sessions
# Ctrl+B D                         # detach from session
```

### System Info

```bash
# Pretty system info
neofetch                           # pacman -S neofetch
fastfetch                          # pacman -S fastfetch
```

## Useful One-Liners

Quick recipes for common tasks.

```bash
# Find largest files on the system
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Kill all processes matching a pattern
pkill -f "pattern"

# Monitor a file for changes in real time
tail -f /var/log/pacman.log

# Test if a port is open (listen / connect)
nc -l 8080                         # listen on port
nc host 8080                       # connect to port

# Compare directory contents
diff <(ls dir1) <(ls dir2)

# Count files by extension in current directory tree
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn
```
