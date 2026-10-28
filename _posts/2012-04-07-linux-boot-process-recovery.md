---
layout: post
title: "Linux Boot Process and Recovery: GRUB2, initramfs, and Rescue Mode"
date: Apr-07-2012
categories: [Linux, SysAdmin]
tags: [linux, boot, grub, initramfs, recovery, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

A server that will not boot is the highest-stakes Linux problem there is. The good news is that the boot process is a short, well-defined sequence, and each stage has a known failure mode and a known recovery. This post walks the sequence and the fixes.

## The Boot Sequence

1. **Firmware** (BIOS or UEFI) selects a boot device.
2. **Bootloader** (GRUB2) loads the kernel and initramfs.
3. **Kernel** initialises hardware and mounts the initramfs.
4. **initramfs** loads the drivers needed to mount the real root filesystem.
5. **systemd** (PID 1) starts units and reaches the default target.

Each stage fails differently, and the error message usually tells you which one.

## Inspecting the Boot

```bash
# How long did boot take?
systemd-analyze

# Which units took the longest?
systemd-analyze blame

# The critical dependency chain
systemd-analyze critical-chain

# Kernel messages from this boot
dmesg -T

# Full boot log
journalctl -b
```

*Tip:* `systemd-analyze critical-chain` shows the exact dependency path that is slowing down boot. It is far more useful than `blame` alone.

## GRUB2 Basics

```bash
# Regenerate the GRUB config (Debian/Ubuntu)
sudo update-grub

# Regenerate the GRUB config (RHEL family)
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Show the current kernel command line
cat /proc/cmdline

# List installed kernels
ls /boot/vmlinuz-*
```

*Tip:* Never edit `/boot/grub/grub.cfg` directly. It is generated. Edit `/etc/default/grub` and regenerate.

## Editing the Kernel Command Line at Boot

When a system will not boot, the fastest fix is often a kernel parameter.

1. At the GRUB menu, press `e` to edit the selected entry.
2. Find the line starting with `linux` (or `linux16`).
3. Append the parameter you need.
4. Press `Ctrl+X` or `F10` to boot.

Useful parameters:

* `single` or `systemd.unit=rescue.target` — boot to a single-user shell.
* `systemd.unit=emergency.target` — minimal environment, root filesystem read-only.
* `rd.break` — drop to a shell inside the initramfs before the root filesystem is mounted.
* `rw init=/bin/bash` — boot straight to a root shell with no systemd.

*Tip:* `rd.break` is the standard way to reset a forgotten root password on RHEL family systems. It drops you into the initramfs, where you can remount the root filesystem read-write and change the password.

## Resetting a Forgotten Root Password

```bash
# At the GRUB menu, edit the kernel line and append:
rd.break

# Inside the initramfs shell:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
```

*Tip:* `touch /.autorelabel` is essential on SELinux systems. Without it, the new password file has the wrong context and the system will not boot.

## initramfs Problems

If the kernel cannot find the root filesystem, the initramfs is usually missing a driver.

```bash
# Rebuild the initramfs (Debian/Ubuntu)
sudo update-initramfs -u -k all

# Rebuild the initramfs (RHEL family)
sudo dracut -f

# List the contents of an initramfs
lsinitramfs /boot/initrd.img-$(uname -r)
```

*Tip:* After adding a new storage controller or moving a disk to new hardware, rebuild the initramfs. The old one may not contain the driver for the new controller.

## Filesystem Errors at Boot

If the root filesystem fails `fsck`, the system drops to emergency mode.

```bash
# Check the root filesystem from a rescue environment
sudo fsck -f /dev/sda1

# Remount read-write after repair
sudo mount -o remount,rw /

# Check fstab for errors
sudo findmnt --verify
```

*Tip:* A bad `fstab` entry is the most common cause of emergency mode. Boot with `rd.break`, comment out the offending line, and reboot.

## Rescue and Emergency Targets

```bash
# Boot to rescue mode (single-user, most services stopped)
sudo systemctl rescue

# Boot to emergency mode (minimal, root read-only)
sudo systemctl emergency

# Return to the default target
sudo systemctl default
```

*Tip:* Rescue mode mounts local filesystems and starts a shell. Emergency mode mounts only the root filesystem, read-only. Use emergency when rescue itself fails.

## Quick Reference

```bash
# Why is boot slow?
systemd-analyze critical-chain

# Reset root password
# At GRUB: append rd.break, then:
mount -o remount,rw /sysroot && chroot /sysroot && passwd root && touch /.autorelabel

# Rebuild initramfs
sudo dracut -f

# Check fstab
sudo findmnt --verify
```

*Tip:* Work in order: read the error, identify the stage, boot to the right target, fix, reboot. Most boot failures are a bad `fstab` entry, a missing initramfs driver, or a filesystem that needs `fsck`.
