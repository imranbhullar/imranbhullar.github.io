---
layout: post
title: "mdadm Software RAID: Building, Monitoring, and Recovering Arrays"
date: Jul-07-2012
categories: [Linux, SysAdmin]
tags: [linux, raid, mdadm, storage, recovery]
image: /assets/img/Linux_unix_bsd.png
---

Software RAID with `mdadm` is the standard way to get redundancy on Linux without a hardware controller. It is also the layer most admins ignore until a disk fails, at which point the recovery steps matter enormously. This post covers building, monitoring, and recovering arrays.

## Checking the Current State

```bash
# All arrays
cat /proc/mdstat

# Detailed view
sudo mdadm --detail /dev/md0

# Scan for arrays
sudo mdadm --examine --scan
```

*Tip:* `/proc/mdstat` is the one file to check first. It shows every array, its state, and any rebuild in progress in a single view.

## Creating an Array

```bash
# Create a RAID 1 (mirror) from two disks
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Create a RAID 5 from three disks
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# Create a RAID 10 from four disks
sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde

# Format and mount
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /mnt/raid
sudo mount /dev/md0 /mnt/raid
```

*Tip:* `--raid-devices` must match the number of devices you pass. A mismatch creates an array that will not assemble at boot.

## Persisting the Array

```bash
# Save the array configuration
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf

# Update the initramfs so the array assembles at boot
sudo update-initramfs -u    # Debian/Ubuntu
sudo dracut -f              # RHEL family
```

*Tip:* Without the `mdadm.conf` entry and an updated initramfs, the array may not assemble at boot, and the system drops to emergency mode.

## Monitoring an Array

```bash
# Watch the array state
watch -n 2 cat /proc/mdstat

# Check for degraded arrays
cat /proc/mdstat | grep -i degraded

# Check SMART on the member disks
sudo smartctl -H /dev/sdb
sudo smartctl -H /dev/sdc
```

*Tip:* A degraded array is not an emergency, but it is a warning. Replace the failed disk before a second failure takes the array down.

## Replacing a Failed Disk

```bash
# Mark the failed disk as removed
sudo mdadm /dev/md0 --fail /dev/sdb
sudo mdadm /dev/md0 --remove /dev/sdb

# Physically replace the disk, then add the new one
sudo mdadm /dev/md0 --add /dev/sdb

# Watch the rebuild
watch -n 2 cat /proc/mdstat
```

*Tip:* The rebuild can take hours on large disks. Run it in `tmux` so a dropped SSH session does not interrupt your monitoring.

## Growing an Array

```bash
# Add a new disk to a RAID 5
sudo mdadm /dev/md0 --add /dev/sdf

# Grow the array to use it
sudo mdadm --grow /dev/md0 --raid-devices=4

# Grow the filesystem
sudo resize2fs /dev/md0
```

*Tip:* Growing a RAID 5 is a multi-hour operation with a real risk of failure. Back up before you start, and never grow an array that is already degraded.

## Recovering a Broken Array

```bash
# Stop the array
sudo mdadm --stop /dev/md0

# Assemble it manually
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc

# Force assembly if the superblock is out of date
sudo mdadm --assemble --force /dev/md0 /dev/sdb /dev/sdc

# Examine a member disk
sudo mdadm --examine /dev/sdb
```

*Tip:* `--force` assembly is a last resort. It can bring up an array with stale data, so back up whatever you can read before you try it.

## Quick Reference

```bash
# What arrays exist?
cat /proc/mdstat

# Create a mirror
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Replace a failed disk
sudo mdadm /dev/md0 --fail /dev/sdb
sudo mdadm /dev/md0 --remove /dev/sdb
sudo mdadm /dev/md0 --add /dev/sdb

# Persist the config
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
```

*Tip:* Work in order: check `/proc/mdstat`, replace the failed disk, watch the rebuild, then verify SMART on the remaining disks. Most RAID incidents are a failed disk that nobody noticed.
