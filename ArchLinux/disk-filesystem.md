# Disk & Filesystem Management

## Viewing Disks and Partitions

```bash
# List all block devices
lsblk

# Show with filesystem info
lsblk -f

# Include sizes and mount points
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID

# Detailed partition info
sudo fdisk -l

# Show specific disk
sudo fdisk -l /dev/sda

# Disk usage summary
df -h

# Disk usage by inode
df -i

# Show filesystem type
df -T

# Show specific mount
df -h /home

# Show disk UUIDs
blkid
ls -l /dev/disk/by-uuid/
ls -l /dev/disk/by-id/
ls -l /dev/disk/by-label/
```

## Mounting

```bash
# Mount a partition
sudo mount /dev/sdb1 /mnt

# Mount with filesystem type
sudo mount -t ext4 /dev/sdb1 /mnt

# Mount read-only
sudo mount -o ro /dev/sdb1 /mnt

# Mount by UUID
sudo mount UUID="xxxx-xxxx" /mnt

# Remount with different options (e.g., add write)
sudo mount -o remount,rw /mnt

# Mount an ISO file
sudo mount -o loop image.iso /mnt/iso

# Mount a tmpfs (RAM disk)
sudo mount -t tmpfs -o size=2G tmpfs /mnt/ramdisk

# Mount a network share (NFS)
sudo mount -t nfs server:/share /mnt/nfs

# Mount a CIFS/SMB share
sudo mount -t cifs //server/share /mnt/smb -o username=user,password=pass

# Unmount
sudo umount /mnt

# Force unmount
sudo umount -l /mnt    # lazy unmount
sudo umount -f /mnt    # force (NFS)

# Show all mounted filesystems
mount | column -t
findmnt
findmnt -t ext4,btrfs    # filter by type

# Show who's using a mount point
fuser -mv /mnt
lsof +D /mnt
```

## fstab

File: `/etc/fstab`

```conf
# <device>                                <mount>  <type>  <options>              <dump> <pass>
UUID=xxxx-xxxx-xxxx                        /        ext4    defaults               0      1
UUID=yyyy-yyyy-yyyy                        /home    ext4    defaults               0      2
UUID=zzzz-zzzz-zzzz                        /boot    vfat    defaults,umask=0077    0      2
UUID=aaaa-aaaa-aaaa                        none     swap    defaults               0      0
tmpfs                                      /tmp     tmpfs   defaults,noatime,mode=1777  0  0
//server/share                             /mnt/smb cifs    credentials=/etc/smbcreds,uid=1000  0  0
```

| Option | Description |
|--------|-------------|
| `defaults` | rw, suid, dev, exec, auto, nouser, async |
| `noatime` | Don't update access times (performance) |
| `nodiratime` | Don't update directory access times |
| `nofail` | Don't fail boot if device is missing |
| `x-systemd.automount` | Lazy mount via systemd |
| `discard` | Enable TRIM for SSDs |
| `compress=zstd` | BTRFS compression |

```bash
# Test fstab without rebooting (mount all entries)
sudo mount -a

# Reload systemd's fstab units
sudo systemctl daemon-reload

# Find a device's UUID
blkid /dev/sdb1
```

## Partitioning

### fdisk (MBR/GPT)

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
sudo gdisk /dev/sdb
# Similar to fdisk but GPT-specific
```

### parted

```bash
# Interactive mode
sudo parted /dev/sdb

# Non-interactive examples
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 1MiB 100GiB
sudo parted /dev/sdb print

# Resize partition
sudo parted /dev/sdb resizepart 2 200GiB
```

## Formatting Filesystems

```bash
# ext4
sudo mkfs.ext4 /dev/sdb1

# ext4 with label
sudo mkfs.ext4 -L "data" /dev/sdb1

# btrfs
sudo mkfs.btrfs /dev/sdb1
sudo mkfs.btrfs -L "data" /dev/sdb1

# FAT32 (for EFI/boot)
sudo mkfs.fat -F32 /dev/sdb1

# XFS
sudo mkfs.xfs /dev/sdb1

# Swap
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
```

## BTRFS

```bash
# Create BTRFS filesystem
sudo mkfs.btrfs -L "btrfs-data" /dev/sdb1

# Mount with options
sudo mount -o compress=zstd,noatime /dev/sdb1 /mnt/data

# Create subvolume
sudo btrfs subvolume create /mnt/data/@
sudo btrfs subvolume create /mnt/data/@home
sudo btrfs subvolume create /mnt/data/@snapshots

# List subvolumes
sudo btrfs subvolume list /mnt/data

# Mount a subvolume
sudo mount -o subvol=@home /dev/sdb1 /home

# Create a snapshot
sudo btrfs subvolume snapshot /mnt/data/@ /mnt/data/@snapshots/2025-01-01

# Create read-only snapshot
sudo btrfs subvolume snapshot -r /mnt/data/@ /mnt/data/@snapshots/2025-01-01

# Delete a subvolume/snapshot
sudo btrfs subvolume delete /mnt/data/@snapshots/2025-01-01

# Show filesystem usage
sudo btrfs filesystem usage /mnt/data
sudo btrfs filesystem df /mnt/data
sudo btrfs filesystem show

# Scrub (check data integrity)
sudo btrfs scrub start /mnt/data
sudo btrfs scrub status /mnt/data

# Balance (rebalance data across devices)
sudo btrfs balance start /mnt/data
sudo btrfs balance status /mnt/data

# Defragment
sudo btrfs filesystem defragment -r /mnt/data

# Add/remove devices (BTRFS RAID)
sudo btrfs device add /dev/sdc1 /mnt/data
sudo btrfs device remove /dev/sdb1 /mnt/data
```

## LVM (Logical Volume Manager)

```bash
# Create physical volume
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdc1

# Show physical volumes
sudo pvs
sudo pvdisplay

# Create volume group
sudo vgcreate myvg /dev/sdb1 /dev/sdc1

# Show volume groups
sudo vgs
sudo vgdisplay

# Create logical volume
sudo lvcreate -L 50G -n mylv myvg
sudo lvcreate -l 100%FREE -n mylv myvg    # use all remaining space

# Show logical volumes
sudo lvs
sudo lvdisplay

# Format and mount
sudo mkfs.ext4 /dev/myvg/mylv
sudo mount /dev/myvg/mylv /mnt/lvm

# Resize logical volume
sudo lvextend -L +10G /dev/myvg/mylv
sudo lvextend -l +100%FREE /dev/myvg/mylv

# Resize filesystem after extending
sudo resize2fs /dev/myvg/mylv         # ext4
sudo xfs_growfs /mnt/lvm              # xfs

# Extend LV and resize filesystem in one command
sudo lvextend -L +10G --resizefs /dev/myvg/mylv

# Reduce logical volume (ext4 — UNMOUNT FIRST)
sudo umount /mnt/lvm
sudo e2fsck -f /dev/myvg/mylv
sudo resize2fs /dev/myvg/mylv 40G
sudo lvreduce -L 40G /dev/myvg/mylv

# Rename
sudo lvrename myvg mylv newname

# Remove
sudo lvremove /dev/myvg/mylv
sudo vgremove myvg
sudo pvremove /dev/sdb1

# Snapshots (LVM)
sudo lvcreate -L 5G -s -n mysnap /dev/myvg/mylv
```

## Swap

```bash
# Show current swap
swapon --show
free -h

# Create swap partition
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2

# Create swap file
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Add to fstab
# /swapfile none swap defaults 0 0

# Disable swap
sudo swapoff /swapfile
sudo swapoff -a    # all swap

# Adjust swappiness (0-200, default 60)
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=10

# Make persistent
# Add to /etc/sysctl.d/99-swappiness.conf:
# vm.swappiness=10
```

## SMART Monitoring

**Install:** `pacman -S smartmontools`

```bash
# Check if SMART is available and enabled
sudo smartctl -i /dev/sda

# Enable SMART
sudo smartctl -s on /dev/sda

# Show all SMART data
sudo smartctl -a /dev/sda

# Quick health check
sudo smartctl -H /dev/sda

# Show error log
sudo smartctl -l error /dev/sda

# Show self-test log
sudo smartctl -l selftest /dev/sda

# Run self-test
sudo smartctl -t short /dev/sda      # 1-2 minutes
sudo smartctl -t long /dev/sda       # hours
sudo smartctl -t conveyance /dev/sda

# NVMe drives
sudo smartctl -a /dev/nvme0

# Enable automatic monitoring daemon
sudo systemctl enable --now smartd
```

## Disk Usage Analysis

```bash
# Directory size
du -sh /home/jo

# Top-level directory sizes
du -h --max-depth=1 /home/jo | sort -hr

# Largest files
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Interactive disk usage (install: pacman -S ncdu)
ncdu /home
```
