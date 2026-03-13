# Pacman & AUR

Package management on Arch Linux — installing, removing, updating, querying, and troubleshooting with `pacman`. AUR helpers (`yay`, `paru`), cache management, and mirror configuration.

## Installing Packages

`pacman -S` installs packages from the synced repositories. Use `--needed` to skip packages that are already up to date.

```bash
# Install a package
sudo pacman -S firefox

# Install multiple packages
sudo pacman -S firefox chromium vlc

# Install without confirmation
sudo pacman -S --noconfirm package

# Install from a specific repository
sudo pacman -S extra/package

# Install a local .pkg.tar.zst file
sudo pacman -U /path/to/package.pkg.tar.zst

# Install from a URL
sudo pacman -U https://example.com/package.pkg.tar.zst

# Reinstall only if not already up to date
sudo pacman -S --needed package
```

## Removing Packages

`-R` removes a package. `-s` also removes unneeded dependencies, `-n` removes backup config files. `-Rns` is the recommended way to cleanly remove a package and everything it pulled in.

```bash
# Remove a package only
sudo pacman -R package

# Remove package + unneeded dependencies
sudo pacman -Rs package

# Remove package + deps + config files (cleanest removal)
sudo pacman -Rns package

# Remove orphaned packages (installed as deps, no longer needed)
sudo pacman -Rns $(pacman -Qdtq)

# List orphans without removing
pacman -Qdt
```

## Updating the System

Always perform a full system upgrade with `-Syu`. Arch is a rolling release — partial upgrades (syncing the database without upgrading) break things.

```bash
# Sync databases and upgrade all packages
sudo pacman -Syu

# Force database refresh + upgrade
sudo pacman -Syyu

# Download packages without installing (e.g., for offline upgrade)
sudo pacman -Syuw
```

> **Warning:** Never run `pacman -Sy package` (partial upgrade). Always do a full `pacman -Syu` first.

## Searching & Querying

`-S` flags query remote repositories, `-Q` flags query locally installed packages. These are your main tools for finding what's installed and what's available.

### Searching Remote Repositories

```bash
# Search by keyword in package name or description
pacman -Ss keyword

# Search with regex
pacman -Ss '^vim-'

# Get detailed info about a remote package
pacman -Si package
```

### Querying Installed Packages

```bash
# Search installed packages
pacman -Qs keyword

# Get info about an installed package (version, size, depends)
pacman -Qi package

# List all files owned by an installed package
pacman -Ql package

# Find which package owns a file
pacman -Qo /usr/bin/vim

# Search for a file in remote packages (requires pkgfile)
pkgfile filename
```

### Listing and Filtering Installed Packages

```bash
# List all explicitly installed packages
pacman -Qe

# List explicitly installed, excluding base/base-devel
pacman -Qet

# List all foreign packages (AUR/manual installs)
pacman -Qm

# List all native packages (from official repos)
pacman -Qn

# List packages by install date (most recently installed last)
expac --timefmt='%Y-%m-%d %T' '%l\t%n' | sort | tail -20

# Check for broken dependencies
sudo pacman -Dk
```

## Cache Management

Pacman caches every package it downloads in `/var/cache/pacman/pkg/`. Over time this grows large. `paccache` (from `pacman-contrib`) manages cleanup by version count.

```bash
# Show cache location and size
du -sh /var/cache/pacman/pkg/

# List cached versions of a package
ls /var/cache/pacman/pkg/ | grep '^firefox-'

# Clean cache — keep only latest 3 versions (default)
sudo paccache -r

# Keep only the latest version
sudo paccache -rk1

# Remove all cached versions of uninstalled packages
sudo paccache -ruk0

# Remove ALL cached packages (use with caution)
sudo pacman -Scc

# Downgrade a package from cache
sudo pacman -U /var/cache/pacman/pkg/package-oldversion.pkg.tar.zst

# Enable automatic weekly cache cleanup via timer
sudo systemctl enable --now paccache.timer
```

## Mirrors

Mirrors determine download speed and freshness. `reflector` automates finding the fastest, most up-to-date mirrors for your location.

```bash
# Edit mirror list directly
sudo vim /etc/pacman.d/mirrorlist

# Install reflector
sudo pacman -S reflector

# Fastest 10 mirrors, HTTPS only, synced in last 12 hours
sudo reflector --country France,Germany --age 12 --protocol https \
    --sort rate --latest 10 --save /etc/pacman.d/mirrorlist

# Automate mirror updates with reflector timer
sudo systemctl enable --now reflector.timer

# Check mirror status
sudo reflector --list | head -20

# Force refresh after mirror change
sudo pacman -Syyu
```

## Configuration

The main config file is `/etc/pacman.conf`. It controls repositories, parallel downloads, and package-level ignore rules.

```ini
# Enable color output
Color

# Show Pac-Man progress bar
ILoveCandy

# Enable parallel downloads
ParallelDownloads = 5

# Enable multilib repository (for 32-bit packages on 64-bit systems)
[multilib]
Include = /etc/pacman.d/mirrorlist
```

```bash
# Ignore a package during upgrades (in pacman.conf)
# IgnorePkg = linux linux-headers

# Ignore a group
# IgnoreGroup = modified

# Skip dependency check during install (dangerous — almost never needed)
sudo pacman -Sd package
```

## AUR (Arch User Repository)

The AUR contains user-submitted PKGBUILDs for packages not in the official repos. Packages are built from source on your machine. Always review the PKGBUILD before building — AUR packages are not vetted.

### Manual AUR Installation

This is the hands-on approach — clone the repo, review the build script, then build.

```bash
# Clone the AUR package
git clone https://aur.archlinux.org/package-name.git
cd package-name

# Review the PKGBUILD before building (check for anything suspicious)
less PKGBUILD

# Build and install (-s installs dependencies, -i installs the result)
makepkg -si

# Build without installing
makepkg -s

# Rebuild, skipping checksums (for local modifications)
makepkg -sf --skipinteg
```

### yay (AUR Helper)

`yay` wraps `pacman` and adds AUR support. It searches both official repos and AUR, handles building, and resolves AUR dependencies.

**Install yay:**

```bash
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si
```

```bash
# Search AUR + repos
yay keyword

# Install from AUR
yay -S package-name

# Update everything (repos + AUR)
yay -Syu

# Update AUR packages only
yay -Sua

# Show AUR package info
yay -Si aur/package

# Remove build dependencies left over after install
yay -Yc

# Edit PKGBUILD before building
yay -S --editmenu package

# Generate dev packages database
yay -Y --gendb
```

### paru (AUR Helper)

`paru` is a feature-rich AUR helper written in Rust. It shows PKGBUILD diffs by default before building, making review easier.

**Install paru:**

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru && makepkg -si
```

```bash
# Search AUR
paru keyword

# Install (shows diff by default)
paru -S package-name

# Update all (repos + AUR)
paru -Syu

# AUR-only update
paru -Sua

# Clean unneeded build dependencies
paru -c

# Fetch AUR comments for a package
paru --comments package
```

## Troubleshooting

Common issues and their fixes when pacman misbehaves.

```bash
# Fix "unable to lock database" (only if no other pacman is running)
sudo rm /var/lib/pacman/db.lck

# Fix corrupted database (force re-sync)
sudo pacman -Syy

# Fix "invalid or corrupted package (PGP signature)"
sudo pacman -S archlinux-keyring
sudo pacman-key --init
sudo pacman-key --populate archlinux

# Reinstall all packages (nuclear option for broken shared libraries)
sudo pacman -S $(pacman -Qnq)

# Check for file conflicts across packages
sudo pacman -Qkk

# Force overwrite conflicting files (use with caution)
sudo pacman -S --overwrite '*' package

# Verify package integrity (check installed files against package database)
pacman -Qk package

# List recently updated packages from the log
grep -i "upgraded" /var/log/pacman.log | tail -20

# View all logged actions for a specific package
grep "package-name" /var/log/pacman.log
```

> **Note:** Downgrading packages is possible using the [Arch Linux Archive](https://archive.archlinux.org/packages/).

## Useful Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc
alias pacup='sudo pacman -Syu'
alias pacin='sudo pacman -S'
alias pacrm='sudo pacman -Rns'
alias pacss='pacman -Ss'
alias pacqs='pacman -Qs'
alias pacown='pacman -Qo'
alias pacorph='sudo pacman -Rns $(pacman -Qdtq)'
alias paclog='grep -i "installed\|upgraded\|removed" /var/log/pacman.log | tail -30'
```
