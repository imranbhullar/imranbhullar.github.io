---
layout: post
title: "LUKS Disk Encryption: cryptsetup from Setup to Recovery"
date: Jun-30-2012
categories: [Linux, Security]
tags: [linux, encryption, luks, cryptsetup, security]
image: /assets/img/Linux_unix_bsd.png
---

Full-disk encryption is the only thing standing between a stolen laptop and your data. LUKS (Linux Unified Key Setup) is the standard on Linux, and `cryptsetup` is the tool that drives it. This post covers creating, opening, and recovering encrypted volumes.

## Checking for LUKS Support

```bash
# Is cryptsetup installed?
cryptsetup --version

# Is the device already LUKS?
sudo cryptsetup isLuks /dev/sdb1 && echo "LUKS" || echo "not LUKS"

# Show LUKS header information
sudo cryptsetup luksDump /dev/sdb1
```

*Tip:* `luksDump` shows the LUKS version, the cipher, and the key slots. It is the first command to run on any encrypted volume you inherit.

## Creating an Encrypted Volume

```bash
# Format a partition as LUKS
sudo cryptsetup luksFormat /dev/sdb1

# Open it and map it to /dev/mapper/data
sudo cryptsetup luksOpen /dev/sdb1 data

# Create a filesystem on the mapped device
sudo mkfs.ext4 /dev/mapper/data

# Mount it
sudo mkdir -p /mnt/data
sudo mount /dev/mapper/data /mnt/data
```

*Tip:* `luksFormat` destroys the existing filesystem without confirmation beyond a `YES` prompt. Double-check the device name before you press enter.

## Managing Keys

```bash
# Add a new key slot
sudo cryptsetup luksAddKey /dev/sdb1

# Remove a key slot
sudo cryptsetup luksRemoveKey /dev/sdb1

# List key slots
sudo cryptsetup luksDump /dev/sdb1 | grep -A1 "Key Slot"

# Change a passphrase
sudo cryptsetup luksChangeKey /dev/sdb1
```

*Tip:* Always add a second key before removing the first. If you remove the only key, the data is gone, and there is no recovery.

## Closing and Reopening

```bash
# Unmount first
sudo umount /mnt/data

# Close the mapping
sudo cryptsetup luksClose data

# Reopen
sudo cryptsetup luksOpen /dev/sdb1 data
```

*Tip:* `luksClose` fails if the volume is still mounted. Unmount it first, or use `luksClose` after `umount` and check with `findmnt`.

## Persistent Mounting with crypttab

```bash
# /etc/crypttab
# name  device      keyfile  options
data    /dev/sdb1   none     luks

# /etc/fstab
# /dev/mapper/data  /mnt/data  ext4  defaults  0 2
```

*Tip:* `none` in the keyfile column means "prompt for a passphrase at boot". For unattended servers, use a keyfile on a separate encrypted volume, never on the same disk.

## Encrypting the Root Filesystem

Root encryption is set up at install time by most distributions. If you need to add it later, the process is:

1. Boot from a live environment.
2. Shrink the root filesystem.
3. Create a LUKS container in the freed space.
4. Move the root filesystem into it.
5. Update GRUB and initramfs.

*Tip:* Root encryption is not a task to attempt on a production host without a tested backup. The failure mode is an unbootable system.

## Recovery: What to Do When the Header Is Damaged

```bash
# Back up the LUKS header (do this immediately after creating the volume)
sudo cryptsetup luksHeaderBackup /dev/sdb1 --header-backup-file /root/luks-header.img

# Restore the header
sudo cryptsetup luksHeaderRestore /dev/sdb1 --header-backup-file /root/luks-header.img
```

*Tip:* The LUKS header is the single point of failure. Back it up to a separate, secure location immediately after creating the volume. Without it, a damaged header means permanent data loss.

## Quick Reference

```bash
# Create and open
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup luksOpen /dev/sdb1 data

# Add a key
sudo cryptsetup luksAddKey /dev/sdb1

# Back up the header
sudo cryptsetup luksHeaderBackup /dev/sdb1 --header-backup-file /root/luks-header.img

# Close
sudo cryptsetup luksClose data
```

*Tip:* Work in order: format, open, format the filesystem, mount, then back up the header. Most LUKS incidents are a missing header backup or a removed key slot.
