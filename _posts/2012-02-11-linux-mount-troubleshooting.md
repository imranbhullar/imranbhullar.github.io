---
layout: post
title: "Linux Mount Troubleshooting: Busy Targets, fstab, and Read-Only Filesystems"
date: Feb-11-2012
categories: [Linux, SysAdmin]
tags: [linux, mount, fstab, troubleshooting, storage]
image: /assets/img/Linux_unix_bsd.png
---

Mount problems are rarely about the `mount` command itself. They are about a process holding a file open, a stale entry in `/etc/fstab`, a filesystem that has been remounted read-only after an error, or a device that has not appeared yet. These are the checks that resolve almost every mount incident.

## Listing Mounts and Finding What Is Mounted Where

Start by confirming what is actually mounted and how.

```bash
# All mounts, human-readable
findmnt

# Mounts in a tree view
findmnt --tree

# Details for a specific mount point
findmnt /var

# Which device backs a given path
findmnt -T /var/log

# Legacy view
mount | column -t
```

*Tip:* `findmnt -T /path` walks up the tree and tells you which filesystem actually backs that path, which is invaluable when you are not sure whether a directory is on its own mount or part of `/`.

## "Target Is Busy" — Finding the Culprit

`umount` refuses to unmount a filesystem that has open files or a process with its working directory inside it. The error is terse; the fix is to find the holder.

```bash
# Which processes have files open under the mount point
sudo lsof +D /mnt/data

# Which processes have their cwd inside the mount point
sudo fuser -vm /mnt/data

# Kill everything holding the mount point
sudo fuser -km /mnt/data

# Lazy unmount, detaches now and cleans up when free
sudo umount -l /mnt/data
```

*Tip:* `fuser -vm` is the fastest way to see both open files and working directories. `umount -l` is a last resort — it hides the problem rather than solving it, and the mount stays alive until the last holder exits.

## Filesystem Remounted Read-Only

When the kernel detects an I/O error or filesystem corruption, it remounts the filesystem read-only to prevent further damage. Writes then fail with `Read-only file system`, which is often mistaken for a permissions problem.

```bash
# Check mount options for ro
findmnt -no OPTIONS /var

# Look for filesystem errors in the kernel log
dmesg -T | grep -i -E "remount|read-only|ext4-fs error|xfs"

# Check the filesystem (unmount first)
sudo umount /var
sudo fsck -f /dev/sdb1

# Remount read-write after fixing
sudo mount -o remount,rw /var
```

*Tip:* Never run `fsck` on a mounted filesystem. Unmount it first, or boot into rescue mode if it is the root filesystem.

## Debugging `/etc/fstab`

A bad `fstab` entry can prevent boot or silently skip a mount. Always test changes before rebooting.

```bash
# Validate fstab syntax without mounting anything
sudo findmnt --verify

# Mount everything in fstab that is not already mounted
sudo mount -a

# Show the UUIDs of all block devices
sudo blkid

# List block devices in a tree
lsblk -f
```

*Tip:* Use UUIDs or `LABEL=` in `fstab`, never `/dev/sdX`. Device names are not stable across reboots, and a reordered disk can mount the wrong filesystem at boot.

## Missing or Unavailable Devices

If a mount fails because the device does not exist, the problem is usually upstream: the disk is not attached, the LVM volume is not active, or the network share is unreachable.

```bash
# Are all expected block devices present?
lsblk

# Is the LVM volume active?
sudo lvs
sudo vgchange -ay

# Is the NFS server reachable?
showmount -e nfs-server

# Is the iSCSI target logged in?
sudo iscsiadm -m session
```

*Tip:* For network filesystems, add `_netdev` to the `fstab` options so systemd waits for the network before attempting the mount. Without it, boot can hang or fail.

## systemd Mount Units

systemd can manage mounts directly, which is useful when you need dependency ordering or automatic retry.

```ini
# /etc/systemd/system/mnt-data.mount
[Unit]
Description=Data mount
After=network-online.target

[Mount]
What=/dev/disk/by-uuid/xxxx-xxxx
Where=/mnt/data
Type=xfs
Options=defaults,noatime

[Install]
WantedBy=multi-user.target
```

```bash
# Enable the mount
sudo systemctl enable --now mnt-data.mount

# Check the status
systemctl status mnt-data.mount

# List mount units
systemctl list-units --type=mount
```

*Tip:* The unit name must match the mount point with slashes replaced by dashes. `/mnt/data` becomes `mnt-data.mount`, and a mismatch means the unit is ignored.

## systemd Automount

An automount unit mounts a filesystem on first access, which is ideal for network shares and removable media.

```ini
# /etc/systemd/system/mnt-data.automount
[Unit]
Description=Automount data

[Automount]
Where=/mnt/data

[Install]
WantedBy=multi-user.target
```

```bash
# Enable the automount
sudo systemctl enable --now mnt-data.automount

# Check the status
systemctl status mnt-data.automount
```

*Tip:* An automount unit needs a matching `.mount` unit. The automount triggers the mount on first access, and the mount unit does the actual work.

## autofs

`autofs` is the classic automounter, and it is still the right tool for large NFS environments.

```bash
# Install
sudo apt install autofs

# Configure a direct map
# /etc/auto.master.d/data.autofs
# /-  /etc/auto.data

# /etc/auto.data
# /mnt/data  -rw,soft,intr  nfs-server:/export/data

# Restart
sudo systemctl restart autofs
```

*Tip:* `autofs` is more flexible than systemd automount for large NFS environments, because it supports wildcards and indirect maps. Use systemd automount for a handful of mounts and `autofs` for hundreds.

## mount --bind

```bash
# Bind-mount a directory
sudo mount --bind /srv/data /mnt/data

# Bind-mount read-only
sudo mount --bind -o ro /srv/data /mnt/data

# Make it persistent in fstab
# /srv/data  /mnt/data  none  bind  0 0
```

*Tip:* `mount --bind` is the cleanest way to expose a directory at a second path without copying it. It is how most chroot and container setups work.

## Quick Reference

```bash
# What is mounted?
findmnt

# Who is holding it?
sudo fuser -vm /mnt/data

# Why is it read-only?
dmesg -T | grep -i remount

# Is fstab valid?
sudo findmnt --verify
```

*Tip:* Work in order: confirm the mount, find the holder, check for read-only remounts, then validate `fstab`. Most mount incidents are resolved by the second step.
