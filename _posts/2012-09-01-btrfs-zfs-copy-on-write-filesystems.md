---
layout: post
title: "Btrfs and ZFS on Linux: Copy-on-Write Filesystems Beyond ext4"
date: Sep-01-2012
categories: [Linux, SysAdmin]
tags: [linux, btrfs, zfs, filesystems, snapshots, storage]
image: /assets/img/btrfs-zfs-hard-drive.jpg
---

`ext4` and `xfs` are journaling filesystems: they protect metadata, not your data, and a snapshot means reaching for LVM. Btrfs and ZFS build copy-on-write and snapshots directly into the filesystem itself, which changes what "oops, let me undo that" actually means on a production server.

## Why Copy-on-Write Changes the Conversation

A copy-on-write filesystem never overwrites data in place. It writes new blocks and atomically updates pointers, which is what makes instant, consistent snapshots and cheap clones possible without a separate volume manager layer. The tradeoff is more complex free-space accounting and, on Btrfs, fragmentation patterns that traditional tools don't expect.

```bash
# Check which copy-on-write filesystems are available in your kernel
grep -E 'btrfs|zfs' /proc/filesystems

# Btrfs tools
sudo apt install btrfs-progs
```

*Tip:* Both filesystems assume they own the whole storage stack. Running Btrfs or ZFS on top of LVM or hardware RAID defeats most of the self-healing features that make them worth using in the first place.

## Creating and Mounting a Btrfs Filesystem

Btrfs can be created on a single device or span several from the start, and basic usage information uses its own commands instead of `df`.

```bash
# Format a device as Btrfs
sudo mkfs.btrfs /dev/sdb1

# Mount it
sudo mount /dev/sdb1 /mnt/data

# Btrfs-aware space usage, which accounts for metadata and RAID profile
sudo btrfs filesystem usage /mnt/data

# Standard df still works, but with Btrfs-specific caveats on free space
df -h /mnt/data
```

*Tip:* Don't trust `df` alone on Btrfs to answer "can I write more data?". `btrfs filesystem usage` shows allocated-but-unused space that `df` can't see, which is often the real answer.

## Subvolumes: Btrfs's Answer to Partitions

A subvolume is an independently mountable tree inside a Btrfs filesystem, and it's the unit that snapshots actually operate on.

```bash
# Create a subvolume
sudo btrfs subvolume create /mnt/data/home

# List subvolumes
sudo btrfs subvolume list /mnt/data

# Mount a specific subvolume directly
sudo mount -o subvol=home /dev/sdb1 /home
```

*Tip:* Most distributions that default to Btrfs (openSUSE, some Fedora spins) split `/`, `/home`, and `/var/log` into separate subvolumes specifically so each can be snapshotted and rolled back independently.

## Snapshots and Rollback on Btrfs

A Btrfs snapshot is a subvolume that shares blocks with its source, so creating one is nearly instantaneous regardless of how much data it contains.

```bash
# Create a read-write snapshot
sudo btrfs subvolume snapshot /mnt/data/home /mnt/data/home-snapshot-2026-01-01

# Create a read-only snapshot, the safer default for backup purposes
sudo btrfs subvolume snapshot -r /mnt/data/home /mnt/data/home-ro-2026-01-01

# "Rolling back" means swapping the subvolume mounted at a given path
sudo mv /mnt/data/home /mnt/data/home-broken
sudo mv /mnt/data/home-snapshot-2026-01-01 /mnt/data/home
```

*Tip:* Btrfs has no single `rollback` command the way ZFS does. Rolling back is a manual subvolume swap, which is exactly why distributions like openSUSE wrap it in a tool like `snapper` rather than expecting users to do it by hand.

## Multi-Device Pools, Balance, and Scrub

Btrfs can mirror or stripe across multiple devices natively, and it includes built-in checksum verification that catches silent corruption.

```bash
# Add a device to an existing filesystem
sudo btrfs device add /dev/sdc1 /mnt/data

# Rebalance data across all devices after adding one
sudo btrfs balance start /mnt/data

# Scrub: read every block and verify checksums, repairing from a mirror if possible
sudo btrfs scrub start /mnt/data
sudo btrfs scrub status /mnt/data
```

*Tip:* Btrfs's parity RAID5/6 modes have known, long-standing reliability problems with the "write hole." Stick to `raid1` or `raid10` profiles for anything you actually care about.

## Installing ZFS on Linux

ZFS isn't in the mainline kernel for licensing reasons, so it ships as a separate module, but it has been rock solid in production for over a decade via OpenZFS.

```bash
# Debian/Ubuntu
sudo apt install zfsutils-linux

# RHEL/CentOS/Fedora: enable the OpenZFS repo first, then
sudo dnf install zfs

# Confirm the kernel module loaded
sudo modprobe zfs
zfs version
```

*Tip:* On RHEL-family systems, the ZFS kernel module is built via DKMS against your running kernel. A kernel upgrade without a matching DKMS rebuild is the single most common cause of "zpool command not found" after a reboot.

## Pools and Datasets: ZFS's Core Abstractions

A ZFS pool (`zpool`) is the physical storage; datasets (`zfs`) are the filesystems and volumes you actually use, each with independently tunable properties.

```bash
# Create a pool mirroring two disks
sudo zpool create tank mirror /dev/sdb /dev/sdc

# Create a dataset inside the pool
sudo zfs create tank/data

# Enable transparent compression, essentially free on modern CPUs
sudo zfs set compression=lz4 tank/data

# Set a quota on a dataset
sudo zfs set quota=50G tank/data

# View pool and dataset status
zpool status tank
zfs list
```

*Tip:* Turn on `compression=lz4` by default. It is nearly free in CPU terms, usually reduces disk I/O rather than increasing it, and ZFS only compresses data that actually compresses well.

## ZFS Snapshots, Rollback, and Replication

Where Btrfs leaves rollback to external tooling, ZFS makes it a single first-class command, and `send`/`receive` turns snapshots into a replication mechanism.

```bash
# Take a snapshot
sudo zfs snapshot tank/data@before-migration

# List snapshots
zfs list -t snapshot

# Roll back to a snapshot, discarding everything written since
sudo zfs rollback tank/data@before-migration

# Replicate a snapshot to another pool or host
sudo zfs send tank/data@before-migration | sudo zfs receive backup/data
```

*Tip:* `zfs rollback` destroys every snapshot newer than the one you're rolling back to, unless you pass `-r`. Check `zfs list -t snapshot` first so you know exactly what you're about to lose.

## Checking Filesystem Health

Both filesystems checksum data by default, which means corruption is detected and, with redundancy, repaired automatically rather than silently propagating.

```bash
# Btrfs: read-only check without repair
sudo btrfs check /dev/sdb1

# ZFS: scrub the pool, verifying every checksum
sudo zpool scrub tank
zpool status tank

# Schedule a monthly scrub via cron or a systemd timer on either filesystem
```

*Tip:* Run a scrub on a schedule, not just when you suspect a problem. Silent bit rot on a disk that's never read back is only caught by a scrub that actively reads every block and checks it against the checksum.

## Bonus: Choosing Between Btrfs, ZFS, and ext4 + LVM

```bash
# Quick decision guide, not a command, but worth keeping nearby:
# - Need rock-solid RAID, snapshots, and compression today: ZFS
# - On a Linux-native distro that already defaults to it: Btrfs
# - Need the absolute simplest, most battle-tested option: ext4/xfs + LVM
```

*Tip:* There is no universally correct answer here. ZFS has the longer production track record and better RAID story; Btrfs integrates more tightly with the Linux kernel and tooling. ext4 with LVM underneath remains the safest default when you don't need either filesystem's advanced features.
