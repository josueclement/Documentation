# Boot & Bootloader

Configuring `systemd-boot` and GRUB, building initramfs with `mkinitcpio`, kernel parameters, microcode updates, Secure Boot, and rescue/recovery procedures.

## systemd-boot

systemd-boot is a simple UEFI boot manager (formerly gummiboot). It's minimal, fast, and integrates well with systemd. It only works on UEFI systems and only supports EFI executables (no BIOS/legacy).

### Installation

```bash
# Install to ESP (usually /boot or /efi)
sudo bootctl install

# Check status (current boot entry, ESP info)
bootctl status

# Update after systemd upgrade
sudo bootctl update
```

### Configuration

The main config controls default entry, timeout, and editor access. Setting `editor no` prevents anyone with physical access from editing kernel parameters at boot.

```ini
# /boot/loader/loader.conf
default  arch.conf
timeout  3
console-mode max
editor   no
```

### Boot Entries

Each entry is a separate `.conf` file in `/boot/loader/entries/`. The `options` line contains kernel parameters.

```ini
# /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options root=UUID=xxxx-xxxx rw quiet

# /boot/loader/entries/arch-fallback.conf
title   Arch Linux (fallback)
linux   /vmlinuz-linux
initrd  /initramfs-linux-fallback.img
options root=UUID=xxxx-xxxx rw

# /boot/loader/entries/arch-lts.conf
title   Arch Linux (LTS)
linux   /vmlinuz-linux-lts
initrd  /initramfs-linux-lts.img
options root=UUID=xxxx-xxxx rw quiet
```

### Managing Entries

```bash
# List available entries
bootctl list

# Set default entry
sudo bootctl set-default arch.conf

# Set one-time boot entry (for next boot only)
sudo bootctl set-oneshot arch-fallback.conf
```

## GRUB

GRUB is a more feature-rich bootloader that supports both UEFI and BIOS/legacy systems, plus advanced features like menus, theming, and OS detection.

### Installation

```bash
# UEFI installation
sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# BIOS/Legacy (install to MBR of disk, not partition)
sudo grub-install --target=i386-pc /dev/sda

# Regenerate config (must be run after any config or kernel change)
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Configuration

GRUB's config is generated from `/etc/default/grub`. Never edit `grub.cfg` directly — it gets overwritten.

```bash
# /etc/default/grub

# Timeout in seconds
GRUB_TIMEOUT=5

# Default entry (0 = first, "saved" = last booted)
GRUB_DEFAULT=0
# GRUB_DEFAULT=saved
# GRUB_SAVEDEFAULT=true

# Kernel parameters for all entries
GRUB_CMDLINE_LINUX_DEFAULT="quiet loglevel=3"
GRUB_CMDLINE_LINUX=""

# Disable submenu (show all entries in a flat list)
GRUB_DISABLE_SUBMENU=y

# Enable os-prober (detect Windows/other OS)
GRUB_DISABLE_OS_PROBER=false
```

```bash
# After any change to /etc/default/grub, regenerate:
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Detect other operating systems (requires os-prober)
sudo pacman -S os-prober
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## mkinitcpio

`mkinitcpio` creates the initial ramdisk (initramfs) — a temporary root filesystem loaded at boot that contains the modules and tools needed to mount the real root filesystem.

Config: `/etc/mkinitcpio.conf`

### Key Sections

```bash
# Modules to include unconditionally (bypass autodetect)
MODULES=(btrfs ext4 nvidia)

# Binaries to include
BINARIES=()

# Files to include
FILES=()

# Hooks — order matters! Each hook adds functionality to the initramfs
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)
```

### Common Hooks

| Hook | Purpose |
|------|---------|
| `base` | Essential base utilities |
| `udev` | Device manager (dynamic device node creation) |
| `autodetect` | Shrink initramfs to only include modules for detected hardware |
| `microcode` | CPU microcode updates (Intel/AMD) |
| `modconf` | Include module config from `/etc/modprobe.d/` |
| `kms` | Kernel Mode Setting (early GPU initialization) |
| `keyboard` | Keyboard drivers (needed for LUKS passphrase entry) |
| `keymap` | Keymap from `/etc/vconsole.conf` |
| `consolefont` | Console font |
| `block` | Block device modules (SATA, NVMe, USB storage) |
| `encrypt` | LUKS encryption support |
| `lvm2` | LVM support |
| `filesystems` | Filesystem modules (ext4, btrfs, xfs, etc.) |
| `fsck` | Filesystem check at boot |

### Generating Initramfs

```bash
# Regenerate initramfs for all presets (all installed kernels)
sudo mkinitcpio -P

# Regenerate for a specific preset only
sudo mkinitcpio -p linux
sudo mkinitcpio -p linux-lts

# Generate to a custom path
sudo mkinitcpio -g /boot/initramfs-custom.img

# List all available hooks
mkinitcpio -L

# Show help for a specific hook
mkinitcpio -H encrypt
```

## Kernel Parameters

Kernel parameters control boot behavior, hardware settings, and kernel features. They're passed via the bootloader's config.

### Setting Kernel Parameters

```bash
# Temporary (current boot only):
# At boot menu: press 'e' to edit, add params to the linux line

# Permanent — systemd-boot:
# Edit /boot/loader/entries/arch.conf, modify the 'options' line

# Permanent — GRUB:
# Edit /etc/default/grub, modify GRUB_CMDLINE_LINUX_DEFAULT
# Then: sudo grub-mkconfig -o /boot/grub/grub.cfg

# Runtime kernel parameters (sysctl — different from boot params):
sudo sysctl -w net.ipv4.ip_forward=1
# Persistent: create /etc/sysctl.d/99-custom.conf
```

### Common Kernel Parameters

| Parameter | Purpose |
|-----------|---------|
| `quiet` | Suppress most boot messages |
| `loglevel=3` | Show only errors during boot |
| `splash` | Show splash screen (if configured) |
| `nomodeset` | Disable KMS (troubleshoot GPU issues) |
| `nvidia-drm.modeset=1` | Enable NVIDIA DRM modesetting |
| `mem_sleep_default=deep` | Use S3 deep sleep instead of s2idle |
| `nowatchdog` | Disable watchdog timers (slightly faster shutdown) |
| `mitigations=off` | Disable CPU vulnerability mitigations (**insecure**) |
| `rootflags=subvol=@` | BTRFS root subvolume |
| `resume=UUID=xxxx` | Swap device for hibernation resume |
| `rd.luks.name=UUID=cryptroot` | LUKS encrypted root device |
| `init=/bin/bash` | Emergency: boot straight to bash (skip systemd) |
| `systemd.unit=rescue.target` | Boot to rescue mode |
| `systemd.unit=emergency.target` | Boot to emergency shell |

### Viewing Current Parameters

```bash
# View the kernel command line used for the current boot
cat /proc/cmdline

# View/set runtime kernel parameters
sysctl -a
sysctl net.ipv4.ip_forward
```

## Rescue & Recovery

When your system won't boot, you have two options: modify boot parameters from the bootloader menu, or boot from an Arch live USB and chroot into the installed system.

### From systemd-boot / GRUB Menu

Add these to the kernel parameters at the boot menu (press `e` to edit):

```bash
# Boot to rescue mode (root shell, basic services)
systemd.unit=rescue.target

# Boot to emergency mode (root shell, almost nothing mounted)
systemd.unit=emergency.target

# Drop to root shell bypassing init entirely
init=/bin/bash
```

### Using Arch Live USB

This is the standard recovery procedure for more serious problems (broken bootloader, missing kernel, corrupted filesystem).

```bash
# 1. Boot from Arch ISO

# 2. Identify partitions
lsblk

# 3. Mount the root partition
mount /dev/sda2 /mnt

# Mount boot partition
mount /dev/sda1 /mnt/boot

# Mount other partitions as needed
mount /dev/sda3 /mnt/home

# 4. chroot into the installed system
arch-chroot /mnt

# 5. Now you can fix things:

# Fix bootloader
bootctl install                    # systemd-boot
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

# Reinstall kernel
pacman -S linux linux-headers

# Regenerate initramfs
mkinitcpio -P

# Reset root password
passwd

# Fix fstab
vim /etc/fstab

# Fix broken packages
pacman -Syu

# 6. Exit and reboot
exit
umount -R /mnt
reboot
```

### Common Recovery Tasks

```bash
# Reinstall all packages (fix broken shared libraries)
pacman -S $(pacman -Qnq)

# Reinstall keyring (fix signature errors)
pacman -S archlinux-keyring
pacman-key --init
pacman-key --populate archlinux

# Check filesystem integrity (unmount first!)
fsck /dev/sda2
fsck.ext4 -f /dev/sda2
btrfs check /dev/sda2

# Fix broken fstab (system won't boot past initramfs)
# Boot to emergency mode, then:
mount -o remount,rw /
vim /etc/fstab

# View boot logs for debugging
journalctl -b -1 -p err               # errors from last boot
journalctl -b -1                       # full last boot log
```

## Microcode Updates

CPU microcode updates fix hardware bugs and security vulnerabilities. They're loaded very early in the boot process, before the kernel's own init.

```bash
# Install CPU microcode
sudo pacman -S intel-ucode    # Intel CPUs
sudo pacman -S amd-ucode      # AMD CPUs

# systemd-boot: add initrd line BEFORE the main initramfs
# initrd  /intel-ucode.img
# initrd  /initramfs-linux.img

# GRUB: automatically detected — just regenerate config
sudo grub-mkconfig -o /boot/grub/grub.cfg

# mkinitcpio: use the microcode hook (included by default in recent versions)
# HOOKS=(... microcode ...)
```

## Secure Boot

Secure Boot ensures only signed boot code runs. By default it only trusts Microsoft's keys. `sbctl` lets you create and manage your own keys for signing your kernel and bootloader.

```bash
# Check Secure Boot status
bootctl status
sbctl status

# Install sbctl (Secure Boot key manager)
sudo pacman -S sbctl

# Create your own signing keys
sudo sbctl create-keys

# Enroll keys (include Microsoft keys for firmware compatibility)
sudo sbctl enroll-keys -m

# Sign boot files
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi

# Verify all signed files
sudo sbctl verify

# List signed files
sudo sbctl list-files
```
