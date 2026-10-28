---
layout: post
title: "LVM: Logical Volume Manager from PV to Snapshot"
date: Mar-10-2012
categories: [Linux, SysAdmin]
tags: [linux, lvm, storage, partitioning, snapshots]
image: /assets/img/Linux_unix_bsd.png
---

LVM sits between your disks and your filesystems and gives you something raw partitions cannot: the ability to grow, shrink, move, and snapshot storage without downtime. Once you understand the three layers — physical volumes, volume groups, and logical volumes — the commands become obvious.

## The Three Layers

* **PV (Physical Volume):** a disk or partition initialised for LVM.
* **VG (Volume Group):** a pool of one or more PVs.
* **LV (Logical Volume):** a slice of a VG that you format and mount.

Think of it as: disks become PVs, PVs pool into a VG, and LVs are carved out of the VG.

## Inspecting the Current Layout

```bash
# Physical volumes
sudo pvs
sudo pvdisplay

# Volume groups
sudo vgs
sudo vgdisplay

# Logical volumes
sudo lvs
sudo lvdisplay

# Tree view of the whole stack
sudo lsblk
```

*Tip:* `lsblk` shows the full hierarchy — disk, partition, PV, VG, LV, filesystem — in one tree. It is the fastest way to understand an unfamiliar LVM layout.

## Creating a PV, VG, and LV

```bash
# Initialise a disk as a physical volume
sudo pvcreate /dev/sdb

# Create a volume group from it
sudo vgcreate data_vg /dev/sdb

# Create a 100G logical volume
sudo lvcreate -L 100G -n data_lv data_vg

# Create an LV using all remaining space
sudo lvcreate -l 100%FREE -n data_lv data_vg

# Format and mount
sudo mkfs.xfs /dev/data_vg/data_lv
sudo mkdir -p /data
sudo mount /dev/data_vg/data_lv /data
```

*Tip:* `-L` takes an absolute size (`100G`), `-l` takes extents or a percentage (`100%FREE`). Mixing them up is the most common LVM syntax error.

## Extending a Logical Volume

This is the operation you will run most often, and it is safe to do online for both XFS and ext4.

```bash
# Grow the LV by 20G
sudo lvextend -L +20G /dev/data_vg/data_lv

# Grow the LV and resize the filesystem in one step
sudo lvextend -r -L +20G /dev/data_vg/data_lv

# Or resize the filesystem manually
sudo xfs_growfs /data          # XFS
sudo resize2fs /dev/data_vg/data_lv   # ext4
```

*Tip:* `lvextend -r` calls the right filesystem resize tool for you. It is the safest option and works for both XFS and ext4.

## Extending a Volume Group

If the VG is out of free space, add another disk to it.

```bash
# Initialise the new disk
sudo pvcreate /dev/sdc

# Add it to the existing VG
sudo vgextend data_vg /dev/sdc

# Confirm the new free space
sudo vgs
```

*Tip:* `vgextend` is online and non-disruptive. You can add a disk to a VG while filesystems on it are mounted and in use.

## Shrinking a Logical Volume

Shrinking is riskier than growing and is **not supported on XFS**. For ext4, unmount first.

```bash
# Unmount
sudo umount /data

# Check the filesystem
sudo e2fsck -f /dev/data_vg/data_lv

# Shrink the filesystem first
sudo resize2fs /dev/data_vg/data_lv 50G

# Then shrink the LV
sudo lvreduce -L 50G /dev/data_vg/data_lv

# Remount
sudo mount /dev/data_vg/data_lv /data
```

*Tip:* Always shrink the filesystem before the LV, and grow the LV before the filesystem. Doing it in the wrong order destroys data.

## Snapshots

LVM snapshots are copy-on-write point-in-time views. They are excellent for short-lived backups before an upgrade, and terrible as long-term backups.

```bash
# Create a 10G snapshot of data_lv
sudo lvcreate -L 10G -s -n data_snap /dev/data_vg/data_lv

# Mount the snapshot read-only
sudo mkdir -p /mnt/snap
sudo mount -o ro /dev/data_vg/data_snap /mnt/snap

# Remove the snapshot when done
sudo umount /mnt/snap
sudo lvremove /dev/data_vg/data_snap
```

*Tip:* A snapshot that fills up becomes invalid. Size it for the amount of change you expect during the snapshot's lifetime, not for the size of the original volume.

## Moving Data Between PVs

When you need to remove a disk from a VG, `pvmove` migrates the extents off it without downtime.

```bash
# Move all extents off /dev/sdb
sudo pvmove /dev/sdb

# Remove the now-empty PV from the VG
sudo vgreduce data_vg /dev/sdb

# Wipe the LVM label
sudo pvremove /dev/sdb
```

*Tip:* `pvmove` can take hours on large volumes. Run it in `tmux` so a dropped SSH session does not interrupt it.

## Thin Provisioning

Thin pools let you over-provision storage: you create logical volumes larger than the physical space, and space is allocated on write.

```bash
# Create a thin pool
sudo lvcreate --type thin-pool -L 100G -n pool data_vg

# Create a thin volume from the pool
sudo lvcreate --type thin -V 500G -n thin_lv --thinpool pool data_vg

# Show thin pool usage
sudo lvs -o+segtype,data_percent,metadata_percent

# Extend the thin pool
sudo lvextend -L +50G data_vg/pool
```

*Tip:* Thin pools need monitoring. If the pool fills up, every thin volume on it becomes read-only, and recovery requires extending the pool or deleting data. Set an alert at 80% usage.

## LVM Cache

LVM cache uses a fast device (SSD or NVMe) as a cache for a slow device (HDD), which is a cheap way to accelerate read-heavy workloads.

```bash
# Create a cache pool
sudo lvcreate --type cache-pool -L 20G -n cachepool data_vg /dev/nvme0n1

# Attach the cache to an existing LV
sudo lvconvert --type cache --cachepool data_vg/cachepool data_vg/data_lv

# Show cache statistics
sudo lvs -o+cache_total_blocks,cache_used_blocks,cache_dirty_blocks

# Detach the cache
sudo lvconvert --uncache data_vg/data_lv
```

*Tip:* LVM cache is a good fit for read-heavy workloads and a poor fit for write-heavy ones. A cache device failure can lose data, so use a mirrored cache pool for anything important.

## lvconvert --type thin: Converting an Existing LV

```bash
# Convert a linear LV to a thin volume
sudo lvconvert --type thin --thinpool data_vg/pool data_vg/data_lv

# Convert a thin volume back to linear
sudo lvconvert --type linear data_vg/thin_lv
```

*Tip:* Converting to thin is online and non-disruptive. Converting back to linear requires the thin volume to be fully allocated, which is rarely the case.

## Quick Reference

```bash
# What do I have?
sudo pvs; sudo vgs; sudo lvs

# Grow an LV and its filesystem
sudo lvextend -r -L +20G /dev/vg/lv

# Add a disk to a VG
sudo pvcreate /dev/sdc && sudo vgextend vg /dev/sdc

# Snapshot before an upgrade
sudo lvcreate -L 10G -s -n snap /dev/vg/lv
```

*Tip:* Work in order: inspect, extend, then snapshot. Most LVM incidents are resolved by `lvextend -r`, and most data loss comes from shrinking in the wrong order.
