# systemd

Managing services, writing unit files, creating timer-based scheduled tasks, switching targets, querying logs with `journalctl`, and analyzing boot performance.

## systemctl — Service Management

`systemctl` is the primary command for interacting with systemd — starting, stopping, enabling, and inspecting services and other units.

### Starting and Stopping

```bash
# Start / stop / restart a service
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# Reload service configuration (without restart — not all services support this)
sudo systemctl reload nginx
```

### Enabling at Boot

`enable` creates symlinks so the service starts automatically at boot. `--now` also starts it immediately.

```bash
# Enable service to start at boot
sudo systemctl enable nginx

# Enable and start immediately
sudo systemctl enable --now nginx

# Disable service from starting at boot
sudo systemctl disable nginx

# Disable and stop immediately
sudo systemctl disable --now nginx
```

### Checking Status

```bash
# Detailed status (running, PID, recent log lines)
systemctl status nginx

# Quick checks (return exit codes for scripting)
systemctl is-active nginx
systemctl is-enabled nginx
systemctl is-failed nginx
```

### Masking

Masking a service prevents it from being started by any means — even manually or as a dependency. Useful for disabling services completely.

```bash
# Mask a service (prevent starting entirely)
sudo systemctl mask nginx

# Unmask
sudo systemctl unmask nginx
```

### Reloading systemd Itself

After creating or editing unit files, reload the systemd manager so it picks up the changes.

```bash
# Reload systemd manager configuration (after editing unit files)
sudo systemctl daemon-reload

# Re-read all service files (heavier, rarely needed)
sudo systemctl daemon-reexec
```

## Listing and Inspecting Units

These commands help you understand what's running, what's failed, and how units are configured.

```bash
# List all running services
systemctl list-units --type=service

# List all services (including inactive)
systemctl list-units --type=service --all

# List failed units
systemctl --failed

# List enabled services
systemctl list-unit-files --type=service --state=enabled

# List all unit files (services, timers, sockets, etc.)
systemctl list-unit-files

# List all timers with next/last run times
systemctl list-timers --all
```

### Inspecting a Specific Unit

```bash
# Show full unit file contents
systemctl cat nginx.service

# Show all unit properties (machine-readable)
systemctl show nginx.service

# Show a specific property
systemctl show nginx.service -p MainPID

# Show unit dependencies (what it pulls in)
systemctl list-dependencies nginx.service

# Show reverse dependencies (what depends on this unit)
systemctl list-dependencies --reverse nginx.service
```

### Editing Units

`systemctl edit` creates a drop-in override file without modifying the original. `--full` opens a copy of the full unit file for editing.

```bash
# Edit a unit's override (drop-in — preferred for small changes)
sudo systemctl edit nginx.service

# Edit the full unit file (copy of original)
sudo systemctl edit --full nginx.service
```

## Writing Service Units

Unit files define how systemd manages a process. Package-provided files live in `/usr/lib/systemd/system/` (don't edit). Admin overrides and custom units go in `/etc/systemd/system/`.

### Example Service

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

The `Type=` directive tells systemd how to determine when the service is "ready".

| Type | Description |
|------|-------------|
| `simple` | Default. Process started by `ExecStart` is the main process |
| `exec` | Like `simple`, but service is "started" only after the binary executes successfully |
| `forking` | Process forks and the parent exits. Set `PIDFile=` so systemd can track the child |
| `oneshot` | Process exits after doing its work. Use `RemainAfterExit=yes` to keep the unit "active" |
| `notify` | Like `simple`, but service signals readiness with `sd_notify()` |
| `idle` | Like `simple`, but execution is delayed until all jobs are dispatched |

### Restart Policies

`Restart=` controls when systemd should automatically restart the service after it exits.

| Policy | Restart on... |
|--------|--------------|
| `no` | Never (default) |
| `always` | Any exit |
| `on-success` | Clean exit (code 0) |
| `on-failure` | Non-zero exit, signal, timeout, watchdog |
| `on-abnormal` | Signal, timeout, watchdog |
| `on-abort` | Signal only |

## Timer Units

Timer units are systemd's replacement for cron jobs. They trigger a corresponding `.service` unit on a schedule. Advantages over cron: journal integration, dependency handling, missed-run catch-up with `Persistent=true`, and per-timer randomized delay.

### Calendar-Based Timer (Realtime)

The timer and service must share the same name (minus the extension). The timer triggers the service.

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

### Managing Timers

```bash
# Enable and start the timer
sudo systemctl enable --now backup.timer

# List active timers with next/last fire times
systemctl list-timers

# Check when a timer will fire next
systemctl status backup.timer

# Manually trigger the associated service (for testing)
sudo systemctl start backup.service
```

### OnCalendar Syntax

`OnCalendar` uses a `DayOfWeek Year-Month-Day Hour:Minute:Second` format. `*` means "any". Use `systemd-analyze calendar` to test expressions.

```bash
# Test calendar expressions (shows next trigger time)
systemd-analyze calendar "Mon *-*-* 08:00:00"
systemd-analyze calendar "hourly"
systemd-analyze calendar "*-*-01 00:00:00"
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

Monotonic timers fire relative to an event (boot, unit activation) rather than wall clock time. Useful for periodic tasks that don't need to run at a specific time.

```ini
[Timer]
OnBootSec=5min          # 5 minutes after boot
OnUnitActiveSec=1h      # 1 hour after unit last activated
OnStartupSec=10min      # 10 minutes after systemd started
```

## Targets (Runlevels)

Targets group units together and represent system states. They replace SysV runlevels. The default target determines what the system boots into.

```bash
# Show current default target
systemctl get-default

# Set default target
sudo systemctl set-default multi-user.target    # no GUI
sudo systemctl set-default graphical.target      # with GUI

# Switch to a target immediately
sudo systemctl isolate multi-user.target
sudo systemctl isolate rescue.target
```

### Common Targets

| Target | Description |
|--------|-------------|
| `poweroff.target` | Shutdown |
| `rescue.target` | Single-user mode (root shell, minimal services) |
| `multi-user.target` | Multi-user, no GUI (like runlevel 3) |
| `graphical.target` | Multi-user with GUI (like runlevel 5) |
| `reboot.target` | Reboot |
| `emergency.target` | Emergency shell (even fewer services than rescue) |

## journalctl — Log Investigation

`journalctl` queries the systemd journal. Unlike traditional log files, the journal is structured and indexed — you can filter by unit, priority, time range, boot, PID, and more. See `system-monitoring.md` for system-wide log analysis and disk management.

### Following Logs

```bash
# View all logs (oldest first)
journalctl

# Follow live (like tail -f)
journalctl -f

# Logs for a specific unit
journalctl -u nginx.service

# Follow a specific unit live
journalctl -fu nginx.service
```

### Filtering by Boot

Each boot gets an index. `-b` is current boot, `-b -1` is previous boot.

```bash
# Logs since last boot
journalctl -b

# Logs from previous boot
journalctl -b -1

# List available boots with timestamps
journalctl --list-boots
```

### Filtering by Time

```bash
# Logs since a specific time
journalctl --since "2025-01-01 00:00:00"
journalctl --since "1 hour ago"
journalctl --since today

# Logs in a time range
journalctl --since "2025-01-01" --until "2025-01-02"
```

### Filtering by Priority

`-p` filters by syslog priority. Shows the specified level **and above** (more severe).

```bash
journalctl -p err                  # errors and above
journalctl -p warning              # warnings and above
```

Priority levels: `emerg` (0), `alert` (1), `crit` (2), `err` (3), `warning` (4), `notice` (5), `info` (6), `debug` (7).

### Other Filters

```bash
# Kernel messages only
journalctl -k

# Show logs for a specific PID
journalctl _PID=1234

# Show logs for a specific binary
journalctl /usr/bin/nginx

# Reverse order (newest first)
journalctl -r

# Limit number of lines
journalctl -n 50
```

### Output Formats

```bash
journalctl -o json-pretty          # JSON format (all fields)
journalctl -o short-iso            # ISO timestamps
journalctl -o verbose              # all fields, human-readable
```

### Journal Maintenance

```bash
# Disk usage of journal
journalctl --disk-usage

# Clean old logs by time
sudo journalctl --vacuum-time=7d

# Clean old logs by size
sudo journalctl --vacuum-size=500M

# Verify journal integrity
journalctl --verify
```

## Boot Analysis

systemd-analyze provides tools to understand and optimize boot times, verify unit files, and audit service security.

```bash
# Show total boot time breakdown (firmware → loader → kernel → userspace)
systemd-analyze

# Per-unit boot times sorted by duration
systemd-analyze blame

# Critical chain (boot dependency tree showing the critical path)
systemd-analyze critical-chain

# Critical chain for a specific unit
systemd-analyze critical-chain nginx.service

# Generate SVG boot visualization
systemd-analyze plot > boot-chart.svg

# Verify unit file syntax (catch errors before deploying)
systemd-analyze verify /etc/systemd/system/myapp.service

# Show all unit file search paths
systemd-analyze unit-paths

# Security audit — score a service's sandboxing (lower is more secure)
systemd-analyze security nginx.service
```

## Useful Patterns

Transient units, scopes, and environment management for one-off tasks.

```bash
# Run a oneshot command as a transient unit (managed by systemd)
sudo systemd-run --unit=my-task /usr/local/bin/task.sh

# Run as a specific user
sudo systemd-run --uid=myuser /usr/local/bin/task.sh

# Run with a transient timer (one-shot, 30 minutes from now)
sudo systemd-run --on-active=30min /usr/local/bin/task.sh

# Scope — wrap an existing process with cgroup resource limits
sudo systemd-run --scope -p MemoryMax=500M /usr/local/bin/heavy-process

# Show environment variables in the systemd manager
systemctl show-environment

# Set environment variable in the systemd manager
sudo systemctl set-environment MY_VAR=value

# List all sockets
systemctl list-sockets

# Reload everything after editing any unit file
sudo systemctl daemon-reload
```
