# Documentation

## Arch Linux

| File | Topics |
|------|--------|
| [File Searching](ArchLinux/file-searching.md) | find, fd, locate, grep, ripgrep |
| [Text Processing](ArchLinux/text-processing.md) | sed, awk, cut, sort, uniq, tr, xargs, tee |
| [Pacman & AUR](ArchLinux/pacman.md) | pacman, yay/paru, AUR, cache, mirrors |
| [systemd](ArchLinux/systemd.md) | systemctl, units, timers, targets, journalctl |
| [Networking](ArchLinux/networking.md) | ip, ss, DNS, NetworkManager, systemd-networkd, nftables |
| [Disk & Filesystem](ArchLinux/disk-filesystem.md) | lsblk, mount, fstab, BTRFS, LVM, swap, SMART |
| [Users & Permissions](ArchLinux/users-permissions.md) | useradd, chmod, chown, ACLs, sudoers, PAM |
| [Process Management](ArchLinux/process-management.md) | ps, kill, signals, nice, jobs, cgroups, ulimit |
| [SSH](ArchLinux/ssh.md) | keys, config, tunnels, agent, sshd hardening, rsync |
| [Boot & Bootloader](ArchLinux/boot-bootloader.md) | systemd-boot, GRUB, mkinitcpio, kernel params, rescue |
| [Compression & Archives](ArchLinux/compression-archives.md) | tar, gzip, xz, zstd, zip, 7z, comparison |
| [Shell Productivity](ArchLinux/shell-productivity.md) | shortcuts, history, aliases, globbing, parameter expansion |
| [System Monitoring](ArchLinux/system-monitoring.md) | journalctl deep-dive, dmesg, vmstat, iostat, hardware, inotifywait |
| [Config Files Reference](ArchLinux/config-files-reference.md) | Centralized path lookup tables |
| [Environment & Shell Config](ArchLinux/environment-shell-config.md) | Startup order, env vars, PATH, XDG, bash/zsh config |
| [Git Quick Reference](ArchLinux/git-quickref.md) | Workflow, branching, remotes, undoing, cherry-pick, worktrees |
| [General Admin](ArchLinux/general-admin.md) | Date/time, locale, cron, kernel modules, tmpfiles, power |

## .NET

| File | Topics |
|------|--------|
| [CLI Commands](DotNet/dotnet-cmd.md) | dotnet new, sln, add, build, run |
| [IConfiguration](DotNet/IConfiguration.md) | Configuration sources, options pattern, custom providers |
| [Templates](DotNet/Templates.md) | Creating and publishing custom dotnet new templates |
| [WPF AppHost](DotNet/WPF-AppHost.md) | WPF with Generic Host integration |

## Docker

| File | Topics |
|------|--------|
| [Overview](Docker/overview.md) | Concepts: images vs containers vs layers, registries, client/daemon model, lifecycle states |
| [Installation](Docker/installation.md) | Install on Arch + Debian/Ubuntu, docker group, enable service, verify, uninstall |
| [Images](Docker/images.md) | pull, images, inspect, tag, rmi, search, save/load, image naming |
| [Containers](Docker/containers.md) | run, ps, start/stop/restart, exec, logs, rm, flags, restart policies, practical examples |
| [Dockerfile](Docker/dockerfile.md) | Instructions, .dockerignore, layer caching, ARG vs ENV, multi-stage builds (incl. .NET example) |
| [Volumes](Docker/volumes.md) | Named volumes, bind mounts, tmpfs, backup pattern, practical examples |
| [Networking](Docker/networking.md) | Default bridge, user-defined networks, host/none modes, port publishing, container DNS |
| [Compose](Docker/compose.md) | docker compose v2, compose.yaml, multi-service example, .env, overrides, profiles |
| [Registries](Docker/registries.md) | Docker Hub, GHCR, private registries, login, push, image naming, self-hosted registry |
| [Cleanup](Docker/cleanup.md) | system df, system prune, targeted prunes, log size control, scheduled cleanup |
| [Security & Rootless](Docker/security-rootless.md) | docker-group caveat, rootless setup, unprivileged containers, capabilities, image trust |

## OPC UA

| File | Topics |
|------|--------|
| [Overview](OpcUa/overview.md) | Goals, address space (node model), read/write, subscriptions, method calls, limitations |
| [.NET — OPC Foundation SDK](OpcUa/dotnet-opc-foundation.md) | NuGet packages, client connection, read/write, browse, subscriptions, method calls, server basics |
