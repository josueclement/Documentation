# Configuration Files Reference

Centralized lookup of important configuration file paths — system, boot, pacman, network, authentication, systemd services, shell, logging, XDG directories, and hardware.

## System

Core system configuration files that define hostname, locale, timezone, kernel parameters, and environment.

| Path | Purpose |
|------|---------|
| `/etc/hostname` | System hostname |
| `/etc/hosts` | Static hostname-to-IP mappings |
| `/etc/locale.conf` | System locale (`LANG`, `LC_*`) |
| `/etc/locale.gen` | Available locales to generate (uncomment, then run `locale-gen`) |
| `/etc/vconsole.conf` | Virtual console keymap and font |
| `/etc/timezone` | Timezone symlink target |
| `/etc/localtime` | Symlink to timezone data in `/usr/share/zoneinfo/` |
| `/etc/machine-id` | Unique machine identifier (auto-generated) |
| `/etc/os-release` | Distribution identification |
| `/etc/sysctl.conf` | Kernel parameters (legacy, single file) |
| `/etc/sysctl.d/*.conf` | Kernel parameters (drop-in, preferred) |
| `/etc/environment` | System-wide environment variables (simple `KEY=VALUE`, no shell expansion) |
| `/etc/profile` | Login shell initialization (all users) |
| `/etc/profile.d/*.sh` | Drop-in login shell scripts |
| `/etc/bash.bashrc` | System-wide bash interactive config |
| `/etc/motd` | Message of the day (shown after login) |
| `/etc/issue` | Pre-login message (shown before login prompt) |

## Boot & Bootloader

Files controlling how the system boots — bootloader config, initramfs, kernel modules, and kernel parameters.

| Path | Purpose |
|------|---------|
| `/boot/loader/loader.conf` | systemd-boot main config |
| `/boot/loader/entries/*.conf` | systemd-boot entry files (one per kernel/option) |
| `/etc/default/grub` | GRUB configuration (source for `grub-mkconfig`) |
| `/boot/grub/grub.cfg` | GRUB generated config (**don't edit directly**) |
| `/etc/mkinitcpio.conf` | Initramfs generation config (hooks, modules, files) |
| `/etc/mkinitcpio.d/*.preset` | Initramfs presets (one per kernel) |
| `/etc/dracut.conf` | Dracut initramfs config (alternative to mkinitcpio) |
| `/etc/modprobe.d/*.conf` | Kernel module options and blacklists |
| `/etc/modules-load.d/*.conf` | Modules to load automatically at boot |
| `/proc/cmdline` | Current kernel command line (read-only) |

## Pacman & Packages

Package manager configuration, mirrors, keyring, and logs.

| Path | Purpose |
|------|---------|
| `/etc/pacman.conf` | Pacman configuration (repos, options, ignores) |
| `/etc/pacman.d/mirrorlist` | Repository mirror URLs (order = priority) |
| `/etc/pacman.d/gnupg/` | Pacman GPG keyring (package signature verification) |
| `/etc/makepkg.conf` | makepkg build configuration (CFLAGS, MAKEFLAGS, packager) |
| `/var/cache/pacman/pkg/` | Downloaded package cache |
| `/var/lib/pacman/` | Pacman database (installed packages, sync DBs) |
| `/var/log/pacman.log` | Pacman transaction log (install, upgrade, remove history) |

## Network

Network interface configuration, DNS, firewall, and connection profiles.

| Path | Purpose |
|------|---------|
| `/etc/resolv.conf` | DNS resolver (often a symlink managed by systemd-resolved or NM) |
| `/etc/nsswitch.conf` | Name service switch config (resolution order: files, dns, mdns) |
| `/etc/NetworkManager/` | NetworkManager main configuration |
| `/etc/NetworkManager/system-connections/` | Saved connection profiles (WiFi passwords, static IPs) |
| `/etc/systemd/network/*.network` | systemd-networkd network config |
| `/etc/systemd/network/*.netdev` | systemd-networkd virtual device definitions |
| `/etc/systemd/resolved.conf` | systemd-resolved DNS resolver config |
| `/etc/nftables.conf` | nftables firewall rules |
| `/etc/hosts.allow` | TCP wrapper allow rules (legacy) |
| `/etc/hosts.deny` | TCP wrapper deny rules (legacy) |
| `/etc/iproute2/rt_tables` | Routing table name-to-ID mappings |

## Authentication & Security

User accounts, passwords, sudo, PAM, SSH, and security policies.

| Path | Purpose |
|------|---------|
| `/etc/passwd` | User accounts (username, UID, GID, home, shell) |
| `/etc/shadow` | Encrypted passwords and expiry info |
| `/etc/group` | Group definitions (name, GID, members) |
| `/etc/gshadow` | Encrypted group passwords |
| `/etc/sudoers` | Sudo rules (**always edit with `visudo`**) |
| `/etc/sudoers.d/` | Drop-in sudo rules (preferred for custom rules) |
| `/etc/pam.d/` | PAM module configuration (per-service auth rules) |
| `/etc/security/limits.conf` | User resource limits (open files, processes) |
| `/etc/security/faillock.conf` | Failed login lockout policy |
| `/etc/security/pwquality.conf` | Password strength rules |
| `/etc/login.defs` | Login defaults (UID ranges, password aging, umask) |
| `/etc/skel/` | Template files copied to new user home directories |
| `/etc/ssh/sshd_config` | SSH server configuration |
| `/etc/ssh/sshd_config.d/` | SSH server drop-in configs |
| `/etc/polkit-1/` | Polkit authorization rules (GUI privilege escalation) |

## Services & Systemd

Unit files, systemd manager config, journal settings, and temporary file management.

| Path | Purpose |
|------|---------|
| `/usr/lib/systemd/system/` | Package-provided unit files (**don't edit**) |
| `/etc/systemd/system/` | Admin unit files and overrides (takes precedence) |
| `~/.config/systemd/user/` | User-session unit files |
| `/etc/systemd/system.conf` | System-wide systemd manager config |
| `/etc/systemd/user.conf` | User-session systemd config |
| `/etc/systemd/journald.conf` | Journal logging configuration (storage, size, retention) |
| `/etc/systemd/logind.conf` | Login manager config (lid close, power key, idle action) |
| `/etc/systemd/timesyncd.conf` | NTP time sync configuration |
| `/etc/systemd/resolved.conf` | DNS resolver config |
| `/etc/systemd/coredump.conf` | Core dump handling |
| `/etc/tmpfiles.d/*.conf` | Temporary file creation/cleanup rules (admin) |
| `/usr/lib/tmpfiles.d/*.conf` | Temporary file rules (package-provided) |

## Shell Configuration

Shell startup files for bash and zsh, readline config, and completion paths.

| Path | Purpose |
|------|---------|
| `~/.bashrc` | Bash interactive shell config (aliases, prompt, functions) |
| `~/.bash_profile` | Bash login shell config (environment, PATH) |
| `~/.bash_logout` | Bash logout actions |
| `~/.zshrc` | Zsh interactive shell config |
| `~/.zprofile` | Zsh login shell config |
| `~/.zshenv` | Zsh environment (always loaded, even for scripts) |
| `~/.inputrc` | Readline configuration (bash keybindings, completion behavior) |
| `~/.profile` | Generic login shell config (fallback if no `.bash_profile`) |
| `/etc/shells` | List of valid login shells (used by `chsh`) |
| `~/.local/share/bash-completion/` | User-installed bash completions |

## Logging

Log files and the systemd journal.

| Path | Purpose |
|------|---------|
| `/var/log/` | Log directory |
| `/var/log/pacman.log` | Package manager transaction log |
| `/var/log/Xorg.0.log` | X server log |
| `/var/log/journal/` | systemd journal persistent storage |
| `/var/log/wtmp` | Login records (query with `last`) |
| `/var/log/btmp` | Failed login records (query with `lastb`) |
| `/var/log/lastlog` | Last login per user (query with `lastlog`) |

## XDG Directories

The XDG Base Directory Specification standardizes where apps store config, data, cache, and runtime files.

| Variable | Default | Purpose |
|----------|---------|---------|
| `XDG_CONFIG_HOME` | `~/.config` | User configuration files |
| `XDG_DATA_HOME` | `~/.local/share` | User data files |
| `XDG_STATE_HOME` | `~/.local/state` | User state (logs, history) |
| `XDG_CACHE_HOME` | `~/.cache` | User cache (safe to delete) |
| `XDG_RUNTIME_DIR` | `/run/user/$UID` | Runtime files (sockets, PIDs — tmpfs, cleared on logout) |

## Hardware & Devices

Filesystem tables, device rules, and kernel/hardware interfaces.

| Path | Purpose |
|------|---------|
| `/etc/fstab` | Filesystem mount table (persistent mounts) |
| `/etc/crypttab` | Encrypted (LUKS) volume table |
| `/etc/udev/rules.d/` | Custom udev rules (device naming, permissions, triggers) |
| `/usr/lib/udev/rules.d/` | Package-provided udev rules |
| `/etc/X11/xorg.conf.d/` | X server configuration fragments |
| `/sys/` | Kernel device/driver interface (sysfs) |
| `/proc/` | Process and kernel info virtual filesystem (procfs) |

## Application-Specific

Common application configuration paths.

| Path | Purpose |
|------|---------|
| `~/.ssh/config` | SSH client configuration |
| `~/.ssh/authorized_keys` | Accepted public keys for SSH login |
| `~/.ssh/known_hosts` | Known remote host fingerprints |
| `~/.gitconfig` | Git user configuration |
| `~/.config/git/config` | Git configuration (XDG-compliant path) |
| `~/.gnupg/` | GPG keyring and config |
| `/etc/docker/daemon.json` | Docker daemon configuration |
