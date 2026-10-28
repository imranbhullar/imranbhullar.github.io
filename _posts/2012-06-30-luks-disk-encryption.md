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

## LUKS2: The Modern Format

LUKS2 is the default on modern distributions. It supports Argon2 key derivation, TPM2, FIDO2, and online re-encryption.

```bash
# Format with LUKS2 explicitly
sudo cryptsetup luksFormat --type luks2 /dev/sdb1

# Show the LUKS2 metadata
sudo cryptsetup luksDump /dev/sdb1

# Convert LUKS1 to LUKS2
sudo cryptsetup convert --type luks2 /dev/sdb1
```

*Tip:* LUKS2 is the default on any distribution released after 2018. If `luksDump` shows `Version: 1`, the volume predates LUKS2 and should be converted.

## TPM2 Unlock with systemd-cryptenroll

Binding a LUKS volume to the TPM2 chip lets the system unlock it automatically at boot, without a passphrase.

```bash
# Enroll the TPM2 device
sudo systemd-cryptenroll --tpm2-device=auto /dev/sda3

# Enroll with a PCR policy
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/sda3

# List enrolled keys
sudo systemd-cryptenroll /dev/sda3

# Remove a key slot
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/sda3
```

*Tip:* Always keep a passphrase key slot alongside the TPM2 slot. If the firmware is updated and the PCR values change, the TPM2 unlock fails and the passphrase is your only way in.

## FIDO2 Unlock

A FIDO2 security key can unlock a LUKS volume, which is a strong second factor for disk encryption.

```bash
# Enroll a FIDO2 key
sudo systemd-cryptenroll --fido2-device=auto /dev/sda3

# Enroll with a PIN
sudo systemd-cryptenroll --fido2-device=auto --fido2-with-client-pin=yes /dev/sda3

# List enrolled keys
sudo systemd-cryptenroll /dev/sda3
```

*Tip:* FIDO2 unlock requires the key to be present at boot. It is a good fit for laptops, not for servers that reboot unattended.

## Detached Headers

A detached header stores the LUKS metadata on a separate device, which lets you keep the header on a USB stick or a separate partition.

```bash
# Format with a detached header
sudo cryptsetup luksFormat --header /boot/luks-header.img /dev/sdb1

# Open with a detached header
sudo cryptsetup luksOpen --header /boot/luks-header.img /dev/sdb1 data

# Back up the detached header
sudo cp /boot/luks-header.img /root/luks-header.img.bak
```

*Tip:* A detached header is a strong defence against offline attacks, because the attacker has no header to attack. It also means losing the header means losing the data, so back it up.

## Online Re-encryption

`cryptsetup reencrypt` changes the encryption key or cipher without taking the volume offline.

```bash
# Re-encrypt with a new key
sudo cryptsetup reencrypt /dev/sdb1

# Re-encrypt with a new cipher
sudo cryptsetup reencrypt --cipher aes-xts-plain64 --key-size 512 /dev/sdb1

# Check progress
sudo cryptsetup status /dev/mapper/data
```

*Tip:* Online re-encryption is safe but slow. Run it in `tmux` and do not interrupt it, because an interrupted re-encryption leaves the volume in a state that requires the header backup to recover.

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
