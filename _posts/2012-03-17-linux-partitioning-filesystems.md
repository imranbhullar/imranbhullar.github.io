---
layout: post
title: "Linux Partitioning and Filesystems: fdisk, parted, mkfs, and fsck"
date: Mar-17-2012
categories: [Linux, SysAdmin]
tags: [linux, partitioning, filesystems, fdisk, parted, mkfs, fsck]
image: /assets/img/Linux_unix_bsd.png
---

Partitioning is the layer most admins touch least often and understand least well, which is exactly why it causes the most damage when something goes wrong. This post covers the tools, the filesystem choices, and the recovery steps that matter.

## Inspecting Disks and Partitions

```bash
# Block devices in a tree with filesystem info
lsblk -f

# Detailed partition table
sudo fdisk -l

# GPT-specific view
sudo gdisk -l /dev/sdb

# Partition table with sizes in human units
sudo parted /dev/sdb print
```

*Tip:* `lsblk -f` is the one command to run first. It shows every disk, partition, filesystem, UUID, and mount point in a single tree.

## Choosing a Partition Table

* **MBR (msdos):** legacy, 4 primary partitions max, 2TB disk limit.
* **GPT:** modern, 128 partitions by default, no practical size limit, required for UEFI boot.

Use GPT unless you have a specific reason not to.

## Partitioning with fdisk

```bash
# Open the interactive editor
sudo fdisk /dev/sdb

# Inside fdisk:
#   n  new partition
#   p  primary
#   1  partition number
#   <enter> default first sector
#   +10G  size
#   w  write and exit
```

*Tip:* `fdisk` writes nothing until you press `w`. If you are unsure, press `q` to quit without saving and start over.

## Partitioning with parted

`parted` is scriptable, which makes it the better choice for automation.

```bash
# Create a GPT label
sudo parted /dev/sdb --script mklabel gpt

# Create a 10G partition
sudo parted /dev/sdb --script mkpart primary xfs 0% 10G

# Print the result
sudo parted /dev/sdb print
```

*Tip:* `parted` applies changes immediately, unlike `fdisk`. There is no "write" step, so double-check the device name before you press enter.

## Creating Filesystems

```bash
# XFS, the default on RHEL family
sudo mkfs.xfs /dev/sdb1

# ext4, the default on Debian family
sudo mkfs.ext4 /dev/sdb1

# btrfs
sudo mkfs.btrfs /dev/sdb1

# Swap
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
```

*Tip:* XFS cannot be shrunk. If you think you may need to shrink a filesystem later, choose ext4 or btrfs.

## Mounting and fstab

```bash
# Mount manually
sudo mount /dev/sdb1 /mnt/data

# Get the UUID for fstab
sudo blkid /dev/sdb1

# Add to /etc/fstab using UUID
# UUID=xxxx-xxxx  /mnt/data  xfs  defaults  0 0

# Test fstab without rebooting
sudo mount -a
```

*Tip:* Always use UUIDs in `fstab`, never `/dev/sdX`. Device names are not stable across reboots.

## Checking and Repairing Filesystems

```bash
# Check ext4 (must be unmounted)
sudo umount /dev/sdb1
sudo e2fsck -f /dev/sdb1

# Check XFS (must be mounted)
sudo xfs_repair -n /dev/sdb1     # dry run
sudo xfs_repair /dev/sdb1        # actual repair, unmounted

# Check btrfs
sudo btrfs check /dev/sdb1
```

*Tip:* `e2fsck` and `xfs_repair` must run on an unmounted filesystem. Running them on a mounted filesystem can corrupt data.

## Resizing Filesystems

```bash
# Grow ext4 (can be done online)
sudo resize2fs /dev/sdb1

# Grow XFS (must be mounted)
sudo xfs_growfs /mnt/data

# Shrink ext4 (must be unmounted)
sudo umount /mnt/data
sudo e2fsck -f /dev/sdb1
sudo resize2fs /dev/sdb1 50G
```

*Tip:* Growing is safe and usually online. Shrinking is risky and always offline. Back up before shrinking.

## Swap Files

```bash
# Create a 4G swap file
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make it persistent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

*Tip:* `fallocate` is fast but can create sparse files on some filesystems. If `swapon` complains, use `dd if=/dev/zero of=/swapfile bs=1M count=4096` instead.

## Quick Reference

```bash
# What do I have?
lsblk -f

# Create a partition
sudo fdisk /dev/sdb

# Create a filesystem
sudo mkfs.xfs /dev/sdb1

# Mount and persist
sudo mount /dev/sdb1 /mnt/data
sudo blkid /dev/sdb1

# Check a filesystem
sudo e2fsck -f /dev/sdb1
```

*Tip:* Work in order: inspect, partition, format, mount, persist. Most partitioning incidents are resolved by `lsblk -f`, and most data loss comes from formatting the wrong device.
