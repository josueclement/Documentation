# Boot & Bootloader

## systemd-boot

systemd-boot is a simple UEFI boot manager (formerly gummiboot).

### Installation

```bash
# Install to ESP (usually /boot or /efi)
sudo bootctl install

# Check status
bootctl status

# Update after systemd upgrade
sudo bootctl update
```

### Configuration

```ini
# /boot/loader/loader.conf
default  arch.conf
timeout  3
console-mode max
editor   no
```

### Boot Entries

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

```bash
# List available entries
bootctl list

# Set default entry
sudo bootctl set-default arch.conf

# Set one-time boot entry
sudo bootctl set-oneshot arch-fallback.conf
```

## GRUB

### Installation

```bash
# UEFI
sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# BIOS/Legacy (MBR)
sudo grub-install --target=i386-pc /dev/sda

# Regenerate config
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Configuration

```bash
# /etc/default/grub

# Timeout
GRUB_TIMEOUT=5

# Default entry (0 = first, "saved" = last booted)
GRUB_DEFAULT=0
# GRUB_DEFAULT=saved
# GRUB_SAVEDEFAULT=true

# Kernel parameters
GRUB_CMDLINE_LINUX_DEFAULT="quiet loglevel=3"
GRUB_CMDLINE_LINUX=""

# Disable submenu
GRUB_DISABLE_SUBMENU=y

# Enable os-prober (detect other OS)
GRUB_DISABLE_OS_PROBER=false
```

```bash
# After any change to /etc/default/grub
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Detect other operating systems
sudo pacman -S os-prober
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## mkinitcpio

mkinitcpio creates the initial ramdisk (initramfs).

Config: `/etc/mkinitcpio.conf`

```bash
# Key sections in mkinitcpio.conf:

# Modules to load early
MODULES=(btrfs ext4 nvidia)

# Binaries to include
BINARIES=()

# Files to include
FILES=()

# Hooks (order matters!)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)
```

### Common Hook Order

| Hook | Purpose |
|------|---------|
| `base` | Base utilities |
| `udev` | Device manager |
| `autodetect` | Shrink initramfs to needed modules |
| `microcode` | CPU microcode updates |
| `modconf` | Module config from `/etc/modprobe.d/` |
| `kms` | Kernel Mode Setting (GPU early start) |
| `keyboard` | Keyboard drivers |
| `keymap` | Keymap from `/etc/vconsole.conf` |
| `consolefont` | Console font |
| `block` | Block device modules |
| `encrypt` | LUKS encryption support |
| `lvm2` | LVM support |
| `filesystems` | Filesystem modules |
| `fsck` | Filesystem check |

```bash
# Regenerate initramfs for all presets
sudo mkinitcpio -P

# Regenerate for specific preset
sudo mkinitcpio -p linux
sudo mkinitcpio -p linux-lts

# Generate to a specific file
sudo mkinitcpio -g /boot/initramfs-custom.img

# List available hooks
mkinitcpio -L

# Show hook help
mkinitcpio -H encrypt
```

## Kernel Parameters

### Setting Kernel Parameters

```bash
# Temporary (current boot only)
# At boot menu: press 'e' to edit, add params to the linux line

# Permanent — systemd-boot
# Edit /boot/loader/entries/arch.conf, modify the 'options' line

# Permanent — GRUB
# Edit /etc/default/grub, modify GRUB_CMDLINE_LINUX_DEFAULT
# Then: sudo grub-mkconfig -o /boot/grub/grub.cfg

# Permanent — sysctl (runtime kernel parameters)
sudo sysctl -w net.ipv4.ip_forward=1
# Persistent: /etc/sysctl.d/99-custom.conf
```

### Common Kernel Parameters

| Parameter | Purpose |
|-----------|---------|
| `quiet` | Suppress most boot messages |
| `loglevel=3` | Show only errors during boot |
| `splash` | Show splash screen |
| `nomodeset` | Disable KMS (troubleshoot GPU issues) |
| `nvidia-drm.modeset=1` | Enable NVIDIA DRM modesetting |
| `mem_sleep_default=deep` | Use S3 deep sleep |
| `nowatchdog` | Disable watchdog timers (faster shutdown) |
| `mitigations=off` | Disable CPU vulnerability mitigations (insecure) |
| `rootflags=subvol=@` | BTRFS root subvolume |
| `resume=UUID=xxxx` | Swap device for hibernation |
| `rd.luks.name=UUID=cryptroot` | LUKS encrypted root |
| `init=/bin/bash` | Emergency: boot to bash (skip init) |
| `systemd.unit=rescue.target` | Boot to rescue mode |
| `systemd.unit=emergency.target` | Boot to emergency shell |

```bash
# View current kernel command line
cat /proc/cmdline

# View current sysctl values
sysctl -a
sysctl net.ipv4.ip_forward
```

## Rescue & Recovery

### From systemd-boot / GRUB Menu

```bash
# Boot to rescue mode — add to kernel parameters:
systemd.unit=rescue.target

# Boot to emergency mode (minimal, no mounts):
systemd.unit=emergency.target

# Drop to root shell:
init=/bin/bash
```

### Using Arch Live USB

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

# 4. chroot into the system
arch-chroot /mnt

# 5. Now you can:
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
# Reinstall all packages (fix broken libraries)
pacman -S $(pacman -Qnq)

# Reinstall keyring (fix signature errors)
pacman -S archlinux-keyring
pacman-key --init
pacman-key --populate archlinux

# Check filesystem
fsck /dev/sda2                         # unmount first!
fsck.ext4 -f /dev/sda2
btrfs check /dev/sda2

# Recover GRUB
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg

# Fix broken fstab (system won't boot)
# Boot to emergency mode, then:
mount -o remount,rw /
vim /etc/fstab

# View boot logs for debugging
journalctl -b -1 -p err               # errors from last boot
journalctl -b -1                       # full last boot log
```

## Microcode Updates

```bash
# Install CPU microcode
sudo pacman -S intel-ucode    # Intel
sudo pacman -S amd-ucode      # AMD

# systemd-boot: add to entry (before initramfs)
# initrd  /intel-ucode.img
# initrd  /initramfs-linux.img

# GRUB: regenerate config (auto-detected)
sudo grub-mkconfig -o /boot/grub/grub.cfg

# mkinitcpio: use the microcode hook (auto in recent versions)
# HOOKS=(... microcode ...)
```

## Secure Boot

```bash
# Check Secure Boot status
bootctl status
sbctl status

# Install sbctl (Secure Boot key manager)
sudo pacman -S sbctl

# Create custom keys
sudo sbctl create-keys

# Enroll keys (with Microsoft keys for compatibility)
sudo sbctl enroll-keys -m

# Sign boot files
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi

# Verify signatures
sudo sbctl verify

# List signed files
sudo sbctl list-files
```
