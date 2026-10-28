---
layout: post
title: "udev Rules: Persistent Device Naming and Custom Rules"
date: Aug-18-2012
categories: [Linux, SysAdmin]
tags: [linux, udev, devices, hardware, naming]
image: /assets/img/Linux_unix_bsd.png
---

Device names like `/dev/sdb` are not stable. Add a disk, remove a USB drive, or reboot in a different order, and the name changes. `udev` is the userspace device manager that solves this, and its rules are how you give a device a name that never changes.

## Inspecting Devices with udevadm

```bash
# Show all attributes of a device
udevadm info --query=all --name=/dev/sdb

# Show the attributes of the parent device
udevadm info --attribute-walk --name=/dev/sdb

# Show the device path
udevadm info --query=path --name=/dev/sdb

# Monitor udev events in real time
udevadm monitor
```

*Tip:* `udevadm info --attribute-walk` is the command you will use most. It prints every attribute you can match on, in the order udev evaluates them.

## Finding Stable Identifiers

```bash
# UUID and label of a filesystem
sudo blkid /dev/sdb1

# Serial number of a disk
sudo udevadm info --query=property --name=/dev/sdb | grep -E "ID_SERIAL|ID_WWN"

# PCI address of a NIC
sudo udevadm info --query=property --name=eth0 | grep ID_NET_NAME
```

*Tip:* `ID_SERIAL` and `ID_WWN` are the most reliable identifiers for disks. They survive reordering, re-cabling, and moving the disk to another port.

## Writing a udev Rule

```bash
# /etc/udev/rules.d/99-my-disk.rules

# Give a disk a stable symlink based on its serial
SUBSYSTEM=="block", ENV{ID_SERIAL}=="ABC123", SYMLINK+="disk/backup"

# Give a NIC a stable name based on its MAC address
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="00:11:22:33:44:55", NAME="eth-backup"

# Run a script when a USB device is plugged in
SUBSYSTEM=="usb", ACTION=="add", RUN+="/usr/local/bin/usb-inserted.sh"
```

```bash
# Reload the rules
sudo udevadm control --reload-rules

# Trigger the rules for existing devices
sudo udevadm trigger
```

*Tip:* `SYMLINK+=` is safer than `NAME=` for block devices. `NAME=` renames the kernel device, which can break other tools; `SYMLINK+=` adds an alias and leaves the original name intact.

## Testing a Rule

```bash
# Test a rule against a device without applying it
sudo udevadm test /sys/class/block/sdb

# Check the udev log
sudo journalctl -u systemd-udevd -f
```

*Tip:* `udevadm test` shows exactly which rules match and what they do. It is the fastest way to debug a rule that is not firing.

## Persistent Network Interface Names

```bash
# /etc/udev/rules.d/70-persistent-net.rules (legacy)
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="00:11:22:33:44:55", NAME="eth0"
```

*Tip:* Modern systems use `systemd-networkd`'s `.link` files or `net.ifnames=0` on the kernel command line instead of `70-persistent-net.rules`. The udev rule still works, but it is no longer the recommended approach.

## Common Use Cases

```bash
# Stable name for a USB serial adapter
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", SYMLINK+="ttyUSB-console"

# Stable name for a disk by serial
SUBSYSTEM=="block", ENV{ID_SERIAL}=="ABC123", SYMLINK+="disk/backup"

# Run a script when a specific USB device is inserted
SUBSYSTEM=="usb", ATTR{idVendor}=="1234", ATTR{idProduct}=="5678", ACTION=="add", RUN+="/usr/local/bin/on-insert.sh"
```

*Tip:* `ATTRS{}` (plural) matches the parent device, while `ATTR{}` matches the device itself. For USB serial adapters, you almost always want `ATTRS{}`.

## Quick Reference

```bash
# Inspect a device
udevadm info --attribute-walk --name=/dev/sdb

# Reload rules
sudo udevadm control --reload-rules
sudo udevadm trigger

# Test a rule
sudo udevadm test /sys/class/block/sdb

# Watch events
udevadm monitor
```

*Tip:* Work in order: inspect the device, find a stable identifier, write the rule, reload, then test. Most udev incidents are a rule that matches too broadly or a missing `udevadm trigger`.
