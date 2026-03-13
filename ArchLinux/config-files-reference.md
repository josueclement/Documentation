# Configuration Files Reference

## System

| Path | Purpose |
|------|---------|
| `/etc/hostname` | System hostname |
| `/etc/hosts` | Static hostname-to-IP mappings |
| `/etc/locale.conf` | System locale (LANG, LC_*) |
| `/etc/locale.gen` | Available locales to generate |
| `/etc/vconsole.conf` | Virtual console keymap and font |
| `/etc/timezone` | Timezone symlink target |
| `/etc/localtime` | Symlink to timezone data |
| `/etc/machine-id` | Unique machine identifier |
| `/etc/os-release` | Distribution identification |
| `/etc/sysctl.conf` | Kernel parameters (legacy) |
| `/etc/sysctl.d/*.conf` | Kernel parameters (drop-in) |
| `/etc/environment` | System-wide environment variables |
| `/etc/profile` | Login shell initialization (all users) |
| `/etc/profile.d/*.sh` | Drop-in login shell scripts |
| `/etc/bash.bashrc` | System-wide bash config |
| `/etc/motd` | Message of the day |
| `/etc/issue` | Pre-login message |

## Boot & Bootloader

| Path | Purpose |
|------|---------|
| `/boot/loader/loader.conf` | systemd-boot main config |
| `/boot/loader/entries/*.conf` | systemd-boot entry files |
| `/etc/default/grub` | GRUB configuration |
| `/boot/grub/grub.cfg` | GRUB generated config (don't edit) |
| `/etc/mkinitcpio.conf` | Initramfs generation config |
| `/etc/mkinitcpio.d/*.preset` | Initramfs presets |
| `/etc/dracut.conf` | Dracut initramfs config (alternative) |
| `/etc/modprobe.d/*.conf` | Kernel module options |
| `/etc/modules-load.d/*.conf` | Modules to load at boot |
| `/proc/cmdline` | Current kernel command line (read-only) |

## Pacman & Packages

| Path | Purpose |
|------|---------|
| `/etc/pacman.conf` | Pacman configuration |
| `/etc/pacman.d/mirrorlist` | Repository mirrors |
| `/etc/pacman.d/gnupg/` | Pacman GPG keyring |
| `/etc/makepkg.conf` | makepkg build configuration |
| `/var/cache/pacman/pkg/` | Package cache |
| `/var/lib/pacman/` | Pacman database |
| `/var/log/pacman.log` | Pacman transaction log |

## Network

| Path | Purpose |
|------|---------|
| `/etc/resolv.conf` | DNS resolver (often managed) |
| `/etc/nsswitch.conf` | Name service switch config |
| `/etc/NetworkManager/` | NetworkManager configs |
| `/etc/NetworkManager/system-connections/` | Saved connection profiles |
| `/etc/systemd/network/*.network` | systemd-networkd config |
| `/etc/systemd/network/*.netdev` | Virtual network devices |
| `/etc/systemd/resolved.conf` | systemd-resolved config |
| `/etc/nftables.conf` | nftables firewall rules |
| `/etc/hosts.allow` | TCP wrapper allow (legacy) |
| `/etc/hosts.deny` | TCP wrapper deny (legacy) |
| `/etc/iproute2/rt_tables` | Routing table names |

## Authentication & Security

| Path | Purpose |
|------|---------|
| `/etc/passwd` | User accounts |
| `/etc/shadow` | Encrypted passwords |
| `/etc/group` | Group definitions |
| `/etc/gshadow` | Encrypted group passwords |
| `/etc/sudoers` | Sudo rules (use `visudo`) |
| `/etc/sudoers.d/` | Drop-in sudo rules |
| `/etc/pam.d/` | PAM module configuration |
| `/etc/security/limits.conf` | User resource limits |
| `/etc/security/faillock.conf` | Failed login lockout policy |
| `/etc/security/pwquality.conf` | Password strength rules |
| `/etc/login.defs` | Login defaults (UID ranges, aging) |
| `/etc/skel/` | Template files for new users |
| `/etc/ssh/sshd_config` | SSH server configuration |
| `/etc/ssh/sshd_config.d/` | SSH server drop-in configs |
| `/etc/polkit-1/` | Polkit authorization rules |

## Services & Systemd

| Path | Purpose |
|------|---------|
| `/usr/lib/systemd/system/` | Package-provided unit files |
| `/etc/systemd/system/` | Admin unit files and overrides |
| `~/.config/systemd/user/` | User unit files |
| `/etc/systemd/system.conf` | System-wide systemd manager config |
| `/etc/systemd/user.conf` | User-session systemd config |
| `/etc/systemd/journald.conf` | Journal logging configuration |
| `/etc/systemd/logind.conf` | Login manager config (lid close, idle) |
| `/etc/systemd/timesyncd.conf` | NTP time sync config |
| `/etc/systemd/resolved.conf` | DNS resolver config |
| `/etc/systemd/coredump.conf` | Core dump handling |
| `/etc/tmpfiles.d/*.conf` | Temporary file management |
| `/usr/lib/tmpfiles.d/*.conf` | Package-provided tmpfiles |

## Shell Configuration

| Path | Purpose |
|------|---------|
| `~/.bashrc` | Bash interactive shell config |
| `~/.bash_profile` | Bash login shell config |
| `~/.bash_logout` | Bash logout actions |
| `~/.zshrc` | Zsh interactive shell config |
| `~/.zprofile` | Zsh login shell config |
| `~/.zshenv` | Zsh environment (always loaded) |
| `~/.inputrc` | Readline configuration (bash) |
| `~/.profile` | Generic login shell config |
| `/etc/shells` | List of valid login shells |
| `~/.local/share/bash-completion/` | User bash completions |

## Logging

| Path | Purpose |
|------|---------|
| `/var/log/` | Log directory |
| `/var/log/pacman.log` | Package manager log |
| `/var/log/Xorg.0.log` | X server log |
| `/var/log/journal/` | systemd journal storage |
| `/var/log/wtmp` | Login records (`last` reads this) |
| `/var/log/btmp` | Failed login records (`lastb`) |
| `/var/log/lastlog` | Last login per user (`lastlog`) |

## XDG Directories

| Variable | Default | Purpose |
|----------|---------|---------|
| `XDG_CONFIG_HOME` | `~/.config` | User configuration |
| `XDG_DATA_HOME` | `~/.local/share` | User data files |
| `XDG_STATE_HOME` | `~/.local/state` | User state (logs, history) |
| `XDG_CACHE_HOME` | `~/.cache` | User cache |
| `XDG_RUNTIME_DIR` | `/run/user/$UID` | Runtime files (sockets, etc.) |

## Hardware & Devices

| Path | Purpose |
|------|---------|
| `/etc/fstab` | Filesystem mount table |
| `/etc/crypttab` | Encrypted volume table |
| `/etc/udev/rules.d/` | Custom udev rules |
| `/usr/lib/udev/rules.d/` | Package-provided udev rules |
| `/etc/X11/xorg.conf.d/` | X server configuration |
| `/sys/` | Kernel device/driver interface |
| `/proc/` | Process and kernel info filesystem |

## Application-Specific

| Path | Purpose |
|------|---------|
| `~/.ssh/config` | SSH client configuration |
| `~/.ssh/authorized_keys` | Accepted public keys for SSH login |
| `~/.ssh/known_hosts` | Known remote host keys |
| `~/.gitconfig` | Git user configuration |
| `~/.config/git/config` | Git configuration (XDG) |
| `~/.gnupg/` | GPG keyring and config |
| `/etc/docker/daemon.json` | Docker daemon config |
