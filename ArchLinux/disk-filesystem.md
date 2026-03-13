# Disk & Filesystem Management

Viewing, partitioning, formatting, and mounting disks. Working with `fstab`, BTRFS subvolumes and snapshots, LVM, swap, and SMART health monitoring.

## Viewing Disks and Partitions

These commands inspect what storage is available, how it's partitioned, and how much space remains.

### Block Devices

`lsblk` shows the hierarchy of disks, partitions, and their mount points. Add `-f` to include filesystem type and UUIDs.

```bash
# List all block devices (tree view)
lsblk

# Show with filesystem info (type, label, UUID, mountpoint)
lsblk -f

# Custom columns
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID

# Detailed partition info (MBR/GPT tables, sector ranges)
sudo fdisk -l

# Show specific disk
sudo fdisk -l /dev/sda
```

### Disk Space Usage

```bash
# Filesystem usage summary (human-readable)
df -h

# Inode usage (running out of inodes is a different problem than running out of space)
df -i

# Show filesystem type column
df -T

# Show specific mount
df -h /home
```

### Identifying Devices by UUID

UUIDs are stable identifiers that don't change when disks are reordered. Always use UUIDs in `fstab`.

```bash
# Show UUIDs for all partitions
blkid

# List disk symlinks
ls -l /dev/disk/by-uuid/
ls -l /dev/disk/by-id/
ls -l /dev/disk/by-label/
```

## Mounting

Mounting attaches a filesystem to a directory (mount point). Changes made via `mount` are temporary — use `fstab` for persistent mounts.

### Basic Mounting

```bash
# Mount a partition
sudo mount /dev/sdb1 /mnt

# Mount with explicit filesystem type
sudo mount -t ext4 /dev/sdb1 /mnt

# Mount read-only
sudo mount -o ro /dev/sdb1 /mnt

# Mount by UUID (more reliable than device path)
sudo mount UUID="xxxx-xxxx" /mnt

# Remount with different options (e.g., add write to a read-only mount)
sudo mount -o remount,rw /mnt
```

### Special Mounts

```bash
# Mount an ISO file
sudo mount -o loop image.iso /mnt/iso

# Mount a tmpfs (RAM disk)
sudo mount -t tmpfs -o size=2G tmpfs /mnt/ramdisk

# Mount a network share (NFS)
sudo mount -t nfs server:/share /mnt/nfs

# Mount a CIFS/SMB share
sudo mount -t cifs //server/share /mnt/smb -o username=user,password=pass
```

### Unmounting

```bash
# Unmount
sudo umount /mnt

# Lazy unmount (detach now, clean up when no longer busy)
sudo umount -l /mnt

# Force unmount (NFS)
sudo umount -f /mnt
```

### Inspecting Mounts

```bash
# Show all mounted filesystems
mount | column -t
findmnt
findmnt -t ext4,btrfs    # filter by type

# Show who's using a mount point (useful when umount says "busy")
fuser -mv /mnt
lsof +D /mnt
```

## fstab

`/etc/fstab` defines filesystems to mount at boot. Each line specifies a device, mount point, type, options, dump flag, and fsck pass order.

```conf
# <device>                                <mount>  <type>  <options>              <dump> <pass>
UUID=xxxx-xxxx-xxxx                        /        ext4    defaults               0      1
UUID=yyyy-yyyy-yyyy                        /home    ext4    defaults               0      2
UUID=zzzz-zzzz-zzzz                        /boot    vfat    defaults,umask=0077    0      2
UUID=aaaa-aaaa-aaaa                        none     swap    defaults               0      0
tmpfs                                      /tmp     tmpfs   defaults,noatime,mode=1777  0  0
//server/share                             /mnt/smb cifs    credentials=/etc/smbcreds,uid=1000  0  0
```

### Common Options

| Option | Description |
|--------|-------------|
| `defaults` | rw, suid, dev, exec, auto, nouser, async |
| `noatime` | Don't update access times (performance boost) |
| `nodiratime` | Don't update directory access times |
| `nofail` | Don't fail boot if device is missing |
| `x-systemd.automount` | Lazy mount via systemd (mount on first access) |
| `discard` | Enable TRIM for SSDs |
| `compress=zstd` | BTRFS compression |

### Testing fstab

```bash
# Mount all fstab entries (test without rebooting)
sudo mount -a

# Reload systemd's fstab-generated units
sudo systemctl daemon-reload

# Find a device's UUID for use in fstab
blkid /dev/sdb1
```

## Partitioning

Partitioning divides a disk into separate regions. Use **GPT** for modern UEFI systems and **MBR** for legacy BIOS.

### fdisk (MBR/GPT)

`fdisk` is an interactive partition editor. Changes are only written when you press `w`.

```bash
# Interactive partitioning
sudo fdisk /dev/sdb

# Common fdisk commands:
# p  — print partition table
# n  — new partition
# d  — delete partition
# t  — change type
# w  — write and exit
# q  — quit without saving
# g  — create new GPT table
# o  — create new MBR table
```

### gdisk (GPT only)

```bash
# GPT-specific partition editor
sudo gdisk /dev/sdb
```

### parted

`parted` supports both MBR and GPT, and can be used non-interactively for scripting.

```bash
# Interactive mode
sudo parted /dev/sdb

# Non-interactive examples
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 1MiB 100GiB
sudo parted /dev/sdb print

# Resize partition (grow only without data loss)
sudo parted /dev/sdb resizepart 2 200GiB
```

## Formatting Filesystems

After partitioning, format the partition with a filesystem before mounting.

```bash
# ext4 (most common Linux filesystem)
sudo mkfs.ext4 /dev/sdb1

# ext4 with label
sudo mkfs.ext4 -L "data" /dev/sdb1

# btrfs (copy-on-write, snapshots, compression)
sudo mkfs.btrfs /dev/sdb1
sudo mkfs.btrfs -L "data" /dev/sdb1

# FAT32 (required for EFI System Partition)
sudo mkfs.fat -F32 /dev/sdb1

# XFS (high-performance, can only grow — not shrink)
sudo mkfs.xfs /dev/sdb1

# Swap
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
```

## BTRFS

BTRFS is a modern copy-on-write filesystem with built-in support for subvolumes, snapshots, compression, and multi-device spanning. Popular on Arch for its snapshot-based rollback capabilities.

### Creating and Mounting

```bash
# Create BTRFS filesystem
sudo mkfs.btrfs -L "btrfs-data" /dev/sdb1

# Mount with compression and no access time updates
sudo mount -o compress=zstd,noatime /dev/sdb1 /mnt/data
```

### Subvolumes

Subvolumes are independently mountable directory trees within a BTRFS filesystem. They enable per-directory snapshots and different mount options.

```bash
# Create subvolumes
sudo btrfs subvolume create /mnt/data/@
sudo btrfs subvolume create /mnt/data/@home
sudo btrfs subvolume create /mnt/data/@snapshots

# List subvolumes
sudo btrfs subvolume list /mnt/data

# Mount a specific subvolume
sudo mount -o subvol=@home /dev/sdb1 /home

# Delete a subvolume
sudo btrfs subvolume delete /mnt/data/@snapshots/2025-01-01
```

### Snapshots

Snapshots are instant, space-efficient copies of a subvolume. Read-only snapshots are ideal for backups.

```bash
# Create a writable snapshot
sudo btrfs subvolume snapshot /mnt/data/@ /mnt/data/@snapshots/2025-01-01

# Create read-only snapshot (for backups)
sudo btrfs subvolume snapshot -r /mnt/data/@ /mnt/data/@snapshots/2025-01-01
```

### Maintenance

```bash
# Show filesystem usage
sudo btrfs filesystem usage /mnt/data
sudo btrfs filesystem df /mnt/data
sudo btrfs filesystem show

# Scrub (verify data integrity by reading all data and checking checksums)
sudo btrfs scrub start /mnt/data
sudo btrfs scrub status /mnt/data

# Balance (rebalance data across devices or rewrite data with new profiles)
sudo btrfs balance start /mnt/data
sudo btrfs balance status /mnt/data

# Defragment
sudo btrfs filesystem defragment -r /mnt/data
```

### Multi-Device (RAID)

```bash
# Add a device to the filesystem
sudo btrfs device add /dev/sdc1 /mnt/data

# Remove a device
sudo btrfs device remove /dev/sdb1 /mnt/data
```

## LVM (Logical Volume Manager)

LVM adds a layer of abstraction between physical disks and filesystems, allowing you to resize, snapshot, and span volumes across multiple disks. The hierarchy is: **Physical Volume (PV)** → **Volume Group (VG)** → **Logical Volume (LV)**.

### Creating the Stack

```bash
# Create physical volumes (mark disks/partitions for LVM use)
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdc1

# Show physical volumes
sudo pvs
sudo pvdisplay

# Create volume group (pool of storage from one or more PVs)
sudo vgcreate myvg /dev/sdb1 /dev/sdc1

# Show volume groups
sudo vgs
sudo vgdisplay

# Create logical volume (the thing you format and mount)
sudo lvcreate -L 50G -n mylv myvg
sudo lvcreate -l 100%FREE -n mylv myvg    # use all remaining space

# Show logical volumes
sudo lvs
sudo lvdisplay

# Format and mount
sudo mkfs.ext4 /dev/myvg/mylv
sudo mount /dev/myvg/mylv /mnt/lvm
```

### Resizing

One of LVM's main advantages — resize volumes without unmounting (for ext4 growth).

```bash
# Extend logical volume
sudo lvextend -L +10G /dev/myvg/mylv
sudo lvextend -l +100%FREE /dev/myvg/mylv

# Resize filesystem after extending
sudo resize2fs /dev/myvg/mylv         # ext4
sudo xfs_growfs /mnt/lvm              # xfs

# Extend LV and resize filesystem in one command
sudo lvextend -L +10G --resizefs /dev/myvg/mylv

# Reduce logical volume (ext4 — MUST unmount first, dangerous if miscalculated)
sudo umount /mnt/lvm
sudo e2fsck -f /dev/myvg/mylv
sudo resize2fs /dev/myvg/mylv 40G
sudo lvreduce -L 40G /dev/myvg/mylv
```

### Cleanup

```bash
# Rename
sudo lvrename myvg mylv newname

# Remove (in reverse order: LV → VG → PV)
sudo lvremove /dev/myvg/mylv
sudo vgremove myvg
sudo pvremove /dev/sdb1

# LVM snapshots (point-in-time copy, allocate space for changes)
sudo lvcreate -L 5G -s -n mysnap /dev/myvg/mylv
```

## Swap

Swap space is used when physical RAM is full. It can be a partition or a file. A swap partition is simpler; a swap file is more flexible (easy to resize).

```bash
# Show current swap
swapon --show
free -h
```

### Swap Partition

```bash
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
```

### Swap File

```bash
# Create a 4GB swap file
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Add to fstab for persistence
# /swapfile none swap defaults 0 0
```

### Managing Swap

```bash
# Disable swap
sudo swapoff /swapfile
sudo swapoff -a    # all swap

# Adjust swappiness (0-200, default 60 — lower means less eager to swap)
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=10

# Make persistent: add to /etc/sysctl.d/99-swappiness.conf
# vm.swappiness=10
```

## SMART Monitoring

SMART (Self-Monitoring, Analysis, and Reporting Technology) lets you check disk health and predict failures. Works on HDDs, SSDs, and NVMe drives.

**Install:** `pacman -S smartmontools`

```bash
# Check if SMART is available and enabled
sudo smartctl -i /dev/sda

# Enable SMART on the drive
sudo smartctl -s on /dev/sda

# Show all SMART data (attributes, thresholds, error log)
sudo smartctl -a /dev/sda

# Quick health check (PASSED/FAILED)
sudo smartctl -H /dev/sda

# Show error log
sudo smartctl -l error /dev/sda

# Show self-test log
sudo smartctl -l selftest /dev/sda
```

### Running Self-Tests

```bash
# Short test (1-2 minutes)
sudo smartctl -t short /dev/sda

# Long test (hours, thorough)
sudo smartctl -t long /dev/sda

# Conveyance test (post-shipping check)
sudo smartctl -t conveyance /dev/sda
```

### NVMe Drives

```bash
sudo smartctl -a /dev/nvme0
```

### Automatic Monitoring

```bash
# Enable the smartd daemon for continuous monitoring
sudo systemctl enable --now smartd
```

## Disk Usage Analysis

Tools for finding what's consuming disk space.

```bash
# Directory size
du -sh /home/jo

# Top-level directory sizes, sorted by size
du -h --max-depth=1 /home/jo | sort -hr

# Largest files on the system
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Interactive disk usage browser (install: pacman -S ncdu)
ncdu /home
```
