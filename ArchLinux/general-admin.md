# General Administration

## Date & Time

```bash
# Show current date and time
date
date +"%Y-%m-%d %H:%M:%S"
date +%F                       # 2025-01-15

# Show in UTC
date -u

# Set date manually (usually not needed with NTP)
sudo date -s "2025-01-15 12:00:00"

# timedatectl (systemd)
timedatectl
timedatectl list-timezones
timedatectl list-timezones | grep -i europe
sudo timedatectl set-timezone Europe/Paris
sudo timedatectl set-ntp true

# NTP status
timedatectl show-timesync
timedatectl timesync-status

# NTP config: /etc/systemd/timesyncd.conf
# [Time]
# NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org

# Hardware clock
sudo hwclock --show
sudo hwclock --systohc        # sync system → hardware

# Date arithmetic
date -d "+7 days"
date -d "last monday"
date -d "2 hours ago"
date -d "2025-01-01 + 90 days" +%F

# Epoch conversions
date +%s                       # current epoch
date -d @1700000000            # epoch to human-readable
```

## Locale

```bash
# Show current locale
locale

# List available locales
locale -a

# Generate locales
sudo vim /etc/locale.gen      # uncomment desired lines
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

Traditional cron format for reference. See `systemd.md` for timer units.

```
┌───────── minute (0-59)
│ ┌─────── hour (0-23)
│ │ ┌───── day of month (1-31)
│ │ │ ┌─── month (1-12)
│ │ │ │ ┌─ day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command
```

| Expression | Meaning |
|-----------|---------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 0 * * *` | Daily at midnight |
| `0 2 * * *` | Daily at 2:00 AM |
| `0 0 * * 0` | Weekly on Sunday |
| `0 0 1 * *` | First day of every month |
| `0 0 1 1 *` | January 1st |
| `30 8 * * 1-5` | 8:30 AM, Monday to Friday |
| `0 */6 * * *` | Every 6 hours |

> **Note:** On Arch, prefer **systemd timers** over cron. Timers integrate with the journal, support calendar expressions, and don't need a separate daemon. See `systemd.md` for timer unit examples.

```bash
# If using cronie (pacman -S cronie)
sudo systemctl enable --now cronie

# Edit crontab
crontab -e

# List crontab
crontab -l

# Remove crontab
crontab -r
```

## Kernel Modules

```bash
# List loaded modules
lsmod

# Show module info
modinfo nvidia
modinfo -p nvidia              # show parameters only

# Load a module
sudo modprobe nvidia

# Load with parameters
sudo modprobe snd_hda_intel power_save=1

# Unload a module
sudo modprobe -r nvidia

# Blacklist a module (prevent loading)
# /etc/modprobe.d/blacklist.conf
# blacklist nouveau

# Auto-load at boot
# /etc/modules-load.d/custom.conf
# nvidia
# vhost_net

# Set persistent module options
# /etc/modprobe.d/nvidia.conf
# options nvidia NVreg_UsePageAttributeTable=1

# Show module dependencies
modprobe --show-depends nvidia

# Force reload
sudo modprobe -r nvidia && sudo modprobe nvidia
```

## tmpfiles.d (Temporary File Management)

systemd-tmpfiles manages creation, deletion, and cleaning of temporary/volatile files.

```ini
# /etc/tmpfiles.d/custom.conf

# Create directory with permissions
d /run/myapp 0755 myuser mygroup -

# Create directory, clean files older than 7 days
D /tmp/myapp 0755 root root 7d

# Create a symlink
L /etc/myapp.conf - - - - /opt/myapp/config/myapp.conf

# Create file with content
f /run/myapp/pid 0644 myuser mygroup - ""

# Remove path on boot
R /tmp/old-data -

# Types:
# d  create directory
# D  create directory + clean contents
# f  create file
# F  create file + truncate
# L  create symlink
# R  remove recursively
# r  remove
# z  adjust permissions (existing path)
# Z  adjust permissions recursively
```

```bash
# Apply tmpfiles configuration
sudo systemd-tmpfiles --create

# Clean according to age rules
sudo systemd-tmpfiles --clean

# Remove entries marked with R/r
sudo systemd-tmpfiles --remove

# Dry run
sudo systemd-tmpfiles --create --dry-run
```

## Power Management

```bash
# Shutdown
sudo poweroff
sudo shutdown now
sudo shutdown -h +10           # shutdown in 10 minutes
sudo shutdown -h 22:00         # shutdown at 10 PM
sudo shutdown -c                # cancel scheduled shutdown

# Reboot
sudo reboot
sudo shutdown -r now
sudo systemctl reboot

# Suspend
sudo systemctl suspend

# Hibernate (requires swap >= RAM, configured resume)
sudo systemctl hibernate

# Hybrid sleep (suspend + hibernate)
sudo systemctl hybrid-sleep

# Suspend then hibernate (after timeout)
sudo systemctl suspend-then-hibernate

# Logind settings: /etc/systemd/logind.conf
# HandleLidSwitch=suspend
# HandlePowerKey=poweroff
# IdleAction=suspend
# IdleActionSec=30min

# Reload logind config
sudo systemctl restart systemd-logind

# Inhibit sleep (e.g., during a download)
systemd-inhibit --what=sleep --who="my-script" --why="downloading" wget https://example.com/big-file

# Check inhibitors
systemd-inhibit --list

# CPU frequency scaling
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# TLP for laptops (pacman -S tlp)
sudo systemctl enable --now tlp
sudo tlp-stat -b               # battery info
```

## Hostname

```bash
# Show hostname
hostname
hostnamectl

# Set hostname
sudo hostnamectl set-hostname myarch

# Set pretty hostname
sudo hostnamectl set-hostname "My Arch Workstation" --pretty

# Relevant files:
# /etc/hostname         — static hostname
# /etc/hosts            — should match hostname
```

## Scheduled Tasks Comparison

| Feature | cron | systemd timer |
|---------|------|---------------|
| Logging | syslog/mail | journalctl |
| Dependencies | None | Full systemd deps |
| Missed runs | Lost | `Persistent=true` |
| Random delay | No | `RandomizedDelaySec` |
| Resource limits | No | Full cgroup support |
| Calendar syntax | `5 fields` | `OnCalendar=` |
| Boot-relative | No | `OnBootSec=` |

## Miscellaneous Utilities

```bash
# Watch command output
watch -n 2 "df -h"            # refresh every 2 seconds
watch -d "free -h"            # highlight changes

# Yes/no to interactive commands
yes | pacman -S package
yes n | command

# Run command at low priority
nice -n 19 make -j$(nproc)
ionice -c3 rsync -a src/ dest/    # idle I/O class

# Timeout a command
timeout 30 wget https://example.com/file.tar.gz
timeout --signal=KILL 10 command

# Repeat command until success
until ping -c1 google.com; do sleep 1; done

# Run command on schedule (one-shot, not permanent)
# Using systemd-run (preferred)
sudo systemd-run --on-active=30min /usr/local/bin/task.sh

# Run at specific time
# Install: pacman -S at
echo "reboot" | sudo at 02:00

# Rename files in bulk
# Install: pacman -S perl-rename
rename 's/\.jpeg$/\.jpg/' *.jpeg
rename 'y/A-Z/a-z/' *              # lowercase all filenames

# Generate random data
head -c 32 /dev/urandom | base64   # random string
openssl rand -hex 16               # random hex

# Checksum files
sha256sum file.txt
sha256sum -c checksums.txt         # verify
md5sum file.txt

# Copy with progress
rsync -ah --progress src dest

# Create ISO from directory
mkisofs -o image.iso /path/to/dir

# Write ISO to USB
sudo dd if=archlinux.iso of=/dev/sdb bs=4M status=progress oflag=sync

# Screen session (for persistent terminals)
# Install: pacman -S tmux (or screen)
tmux                               # new session
tmux new -s name                   # named session
tmux attach -t name                # reattach
tmux ls                            # list sessions
# Ctrl+B D                         # detach

# System information summary
neofetch                           # pacman -S neofetch
fastfetch                          # pacman -S fastfetch
```

## Useful One-Liners

```bash
# Find largest files
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Kill all processes matching a pattern
pkill -f "pattern"

# Monitor file for changes
tail -f /var/log/pacman.log

# Quick HTTP server
python -m http.server 8000

# Port forwarding test
nc -l 8080                         # listen on port
nc host 8080                       # connect to port

# Base64 encode/decode
echo "hello" | base64
echo "aGVsbG8K" | base64 -d

# URL encode/decode
python -c "import urllib.parse; print(urllib.parse.quote('hello world'))"
python -c "import urllib.parse; print(urllib.parse.unquote('hello%20world'))"

# JSON pretty-print
cat data.json | python -m json.tool
cat data.json | jq .

# Compare directory contents
diff <(ls dir1) <(ls dir2)

# Count files by extension
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn
```
