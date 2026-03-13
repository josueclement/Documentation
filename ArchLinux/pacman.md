# Pacman & AUR

## Installing Packages

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

# Reinstall a package
sudo pacman -S --needed package    # skip if already up to date
```

## Removing Packages

```bash
# Remove a package
sudo pacman -R package

# Remove package + unneeded dependencies
sudo pacman -Rs package

# Remove package + deps + config files
sudo pacman -Rns package

# Remove orphaned packages (installed as deps, no longer needed)
sudo pacman -Rns $(pacman -Qdtq)

# List orphans
pacman -Qdt
```

## Updating the System

```bash
# Sync databases and upgrade all packages
sudo pacman -Syu

# Force database refresh + upgrade
sudo pacman -Syyu

# Download packages without installing
sudo pacman -Syuw
```

> **Warning:** Never run `pacman -Sy package` (partial upgrade). Always do a full `pacman -Syu` first.

## Searching & Querying

```bash
# Search remote repositories
pacman -Ss keyword

# Search with regex
pacman -Ss '^vim-'

# Get info about a remote package
pacman -Si package

# Search installed packages
pacman -Qs keyword

# Get info about an installed package
pacman -Qi package

# List files owned by an installed package
pacman -Ql package

# Find which package owns a file
pacman -Qo /usr/bin/vim

# Search for a file in remote packages (requires pkgfile)
pkgfile filename

# List all explicitly installed packages
pacman -Qe

# List all explicitly installed packages (not in base/base-devel)
pacman -Qet

# List all foreign packages (AUR/manual installs)
pacman -Qm

# List all native packages (from repos)
pacman -Qn

# List packages by install date
expac --timefmt='%Y-%m-%d %T' '%l\t%n' | sort | tail -20

# Check for broken dependencies
sudo pacman -Dk
```

## Cache Management

```bash
# Show cache location and size
du -sh /var/cache/pacman/pkg/

# List cached versions of a package
ls /var/cache/pacman/pkg/ | grep '^firefox-'

# Clean cache — keep only latest 3 versions
sudo paccache -r

# Keep only the latest version
sudo paccache -rk1

# Remove all cached versions of uninstalled packages
sudo paccache -ruk0

# Remove all cached packages
sudo pacman -Scc

# Downgrade a package from cache
sudo pacman -U /var/cache/pacman/pkg/package-oldversion.pkg.tar.zst

# Automatic cache cleanup (timer)
sudo systemctl enable --now paccache.timer
```

## Mirrors

```bash
# Edit mirror list
sudo vim /etc/pacman.d/mirrorlist

# Generate a mirror list by speed (using reflector)
sudo pacman -S reflector

# Fastest 10 mirrors, HTTPS only, synced in last 12 hours
sudo reflector --country France,Germany --age 12 --protocol https \
    --sort rate --latest 10 --save /etc/pacman.d/mirrorlist

# Automate with reflector timer
sudo systemctl enable --now reflector.timer

# Check mirror status
sudo reflector --list | head -20

# Force refresh after mirror change
sudo pacman -Syyu
```

## Configuration

Key file: `/etc/pacman.conf`

```ini
# Enable color output
Color

# Show Pac-Man progress bar
ILoveCandy

# Enable parallel downloads
ParallelDownloads = 5

# Enable multilib repository (for 32-bit packages)
[multilib]
Include = /etc/pacman.d/mirrorlist
```

```bash
# Ignore a package during upgrades (in pacman.conf)
# IgnorePkg = linux linux-headers

# Ignore a group
# IgnoreGroup = modified

# Skip dependency check during install (dangerous)
sudo pacman -Sd package
```

## AUR (Arch User Repository)

### Manual AUR Installation

```bash
# Clone the AUR package
git clone https://aur.archlinux.org/package-name.git
cd package-name

# Review the PKGBUILD before building
less PKGBUILD

# Build and install
makepkg -si

# Build without installing
makepkg -s

# Rebuild, skipping checksums (for local modifications)
makepkg -sf --skipinteg
```

### yay (AUR Helper)

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

# Remove build dependencies after install
yay -Yc

# Edit PKGBUILD before building
yay -S --editmenu package

# Generate dev packages database
yay -Y --gendb
```

### paru (AUR Helper)

**Install paru:**

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru && makepkg -si
```

```bash
# Search AUR
paru keyword

# Install
paru -S package-name

# Update all (repos + AUR)
paru -Syu

# AUR-only update
paru -Sua

# Review PKGBUILD + diffs
paru -S package    # shows diff by default

# Clean unneeded deps
paru -c

# Fetch AUR comments
paru --comments package
```

## Troubleshooting

```bash
# Fix "unable to lock database"
sudo rm /var/lib/pacman/db.lck

# Fix corrupted database
sudo pacman -Syy

# Fix "invalid or corrupted package (PGP signature)"
sudo pacman -S archlinux-keyring
sudo pacman-key --init
sudo pacman-key --populate archlinux

# Reinstall all packages (nuclear option for broken libs)
sudo pacman -S $(pacman -Qnq)

# Check for file conflicts
sudo pacman -Qkk

# Force overwrite conflicting files (use with caution)
sudo pacman -S --overwrite '*' package

# Verify package integrity
pacman -Qk package

# Downgrade a package using the Arch Linux Archive
# URL: https://archive.archlinux.org/packages/

# List recently updated packages
grep -i "upgraded" /var/log/pacman.log | tail -20

# View all actions for a specific package
grep "package-name" /var/log/pacman.log
```

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
