# systemd

## systemctl — Service Management

```bash
# Start / stop / restart a service
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# Reload service configuration (without restart)
sudo systemctl reload nginx

# Enable service to start at boot
sudo systemctl enable nginx

# Enable and start immediately
sudo systemctl enable --now nginx

# Disable service from starting at boot
sudo systemctl disable nginx

# Disable and stop immediately
sudo systemctl disable --now nginx

# Check service status
systemctl status nginx

# Check if service is active / enabled / failed
systemctl is-active nginx
systemctl is-enabled nginx
systemctl is-failed nginx

# Mask a service (prevent starting entirely)
sudo systemctl mask nginx

# Unmask
sudo systemctl unmask nginx

# Reload systemd manager configuration (after editing unit files)
sudo systemctl daemon-reload

# Re-read all service files
sudo systemctl daemon-reexec
```

## Listing and Inspecting Units

```bash
# List all running services
systemctl list-units --type=service

# List all services (including inactive)
systemctl list-units --type=service --all

# List failed units
systemctl --failed

# List enabled services
systemctl list-unit-files --type=service --state=enabled

# List all unit files
systemctl list-unit-files

# List all timers
systemctl list-timers --all

# Show full unit file contents
systemctl cat nginx.service

# Show unit properties
systemctl show nginx.service

# Show specific property
systemctl show nginx.service -p MainPID

# Show unit dependencies
systemctl list-dependencies nginx.service

# Show reverse dependencies (what depends on this unit)
systemctl list-dependencies --reverse nginx.service

# Edit a unit's override (drop-in)
sudo systemctl edit nginx.service

# Edit the full unit file
sudo systemctl edit --full nginx.service
```

## Writing Service Units

Unit files live in:
- `/usr/lib/systemd/system/` — package defaults (don't edit)
- `/etc/systemd/system/` — admin overrides (use this)

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target
Wants=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/opt/myapp/data

[Install]
WantedBy=multi-user.target
```

### Service Types

| Type | Description |
|------|-------------|
| `simple` | Default. Process started by `ExecStart` is the main process |
| `exec` | Like `simple`, but service is "started" only after binary executes |
| `forking` | Process forks and parent exits. Set `PIDFile=` |
| `oneshot` | Process exits after doing its work. Use `RemainAfterExit=yes` |
| `notify` | Like `simple`, but service signals readiness with `sd_notify()` |
| `idle` | Like `simple`, but execution is delayed until all jobs are dispatched |

### Restart Policies

| Policy | Restart on... |
|--------|--------------|
| `no` | Never (default) |
| `always` | Any exit |
| `on-success` | Clean exit (code 0) |
| `on-failure` | Non-zero exit, signal, timeout, watchdog |
| `on-abnormal` | Signal, timeout, watchdog |
| `on-abort` | Signal only |

## Timer Units

Timer units replace cron jobs in systemd.

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup every day at 2am

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```bash
# Enable and start the timer
sudo systemctl enable --now backup.timer

# List active timers
systemctl list-timers

# Check when a timer will fire next
systemctl status backup.timer

# Manually trigger the associated service
sudo systemctl start backup.service
```

### OnCalendar Syntax

```bash
# Test calendar expressions
systemd-analyze calendar "Mon *-*-* 08:00:00"
systemd-analyze calendar "hourly"
systemd-analyze calendar "*-*-01 00:00:00"    # first of every month
```

| Expression | Meaning |
|-----------|---------|
| `minutely` | Every minute |
| `hourly` | Every hour |
| `daily` | Every day at midnight |
| `weekly` | Every Monday at midnight |
| `monthly` | First day of every month |
| `*-*-* *:00:00` | Every hour |
| `*-*-* *:*:00` | Every minute |
| `Mon,Fri *-*-* 18:00:00` | Mon and Fri at 6pm |
| `*-*-01 02:00:00` | First of every month at 2am |
| `*-01,07-01 00:00:00` | Jan 1st and Jul 1st |

### Monotonic Timers (Relative)

```ini
[Timer]
OnBootSec=5min          # 5 minutes after boot
OnUnitActiveSec=1h      # 1 hour after unit last activated
OnStartupSec=10min      # 10 minutes after systemd started
```

## Targets (Runlevels)

```bash
# Show current default target
systemctl get-default

# Set default target
sudo systemctl set-default multi-user.target    # no GUI
sudo systemctl set-default graphical.target      # with GUI

# Switch to a target immediately
sudo systemctl isolate multi-user.target
sudo systemctl isolate rescue.target

# Common targets
# poweroff.target     — shutdown
# rescue.target       — single-user mode
# multi-user.target   — multi-user, no GUI (runlevel 3)
# graphical.target    — with GUI (runlevel 5)
# reboot.target       — reboot
# emergency.target    — emergency shell
```

## journalctl — Log Investigation

```bash
# View all logs
journalctl

# Follow live (like tail -f)
journalctl -f

# Logs for a specific unit
journalctl -u nginx.service

# Follow a specific unit
journalctl -fu nginx.service

# Logs since last boot
journalctl -b

# Logs from previous boot
journalctl -b -1

# List available boots
journalctl --list-boots

# Logs since a time
journalctl --since "2025-01-01 00:00:00"
journalctl --since "1 hour ago"
journalctl --since today

# Logs in a time range
journalctl --since "2025-01-01" --until "2025-01-02"

# Filter by priority
journalctl -p err                  # errors and above
journalctl -p warning              # warnings and above
# Priorities: emerg, alert, crit, err, warning, notice, info, debug

# Kernel messages only
journalctl -k

# Show output in different formats
journalctl -o json-pretty          # JSON format
journalctl -o short-iso            # ISO timestamps
journalctl -o verbose              # all fields

# Show logs for a specific PID
journalctl _PID=1234

# Show logs for a specific binary
journalctl /usr/bin/nginx

# Reverse order (newest first)
journalctl -r

# Limit number of lines
journalctl -n 50                   # last 50 lines

# Disk usage of journal
journalctl --disk-usage

# Clean old logs
sudo journalctl --vacuum-time=7d   # keep 7 days
sudo journalctl --vacuum-size=500M # keep 500MB max

# Verify journal integrity
journalctl --verify
```

## Boot Analysis

```bash
# Show boot time breakdown
systemd-analyze

# Blame — show per-unit boot times
systemd-analyze blame

# Critical chain (boot dependency tree)
systemd-analyze critical-chain

# Critical chain for a specific unit
systemd-analyze critical-chain nginx.service

# Generate SVG boot chart
systemd-analyze plot > boot-chart.svg

# Verify unit file syntax
systemd-analyze verify /etc/systemd/system/myapp.service

# Show all unit file paths systemd searches
systemd-analyze unit-paths

# Security audit of a service
systemd-analyze security nginx.service
```

## Useful Patterns

```bash
# Run a oneshot command as a transient unit
sudo systemd-run --unit=my-task /usr/local/bin/task.sh

# Run as a specific user
sudo systemd-run --uid=myuser /usr/local/bin/task.sh

# Run with a timer (transient)
sudo systemd-run --on-active=30min /usr/local/bin/task.sh

# Scope (manage existing process with systemd)
sudo systemd-run --scope -p MemoryMax=500M /usr/local/bin/heavy-process

# Show environment variables for a unit
systemctl show-environment

# Set environment variable for systemd
sudo systemctl set-environment MY_VAR=value

# List all sockets
systemctl list-sockets

# Reload everything after editing any unit file
sudo systemctl daemon-reload
```
