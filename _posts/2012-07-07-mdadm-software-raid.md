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

## RAID 6: Double Parity

RAID 6 tolerates two simultaneous disk failures, at the cost of two parity disks.

```bash
# Create a RAID 6 from four disks
sudo mdadm --create /dev/md0 --level=6 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde

# Check the array
cat /proc/mdstat
```

*Tip:* RAID 6 is the right choice for arrays with more than six disks. The rebuild window on large RAID 5 arrays is long enough that a second failure is a real risk.

## Reshaping an Array

`mdadm --grow` changes the RAID level, the number of devices, or the chunk size without losing data.

```bash
# Add a disk and grow a RAID 5 to four devices
sudo mdadm /dev/md0 --add /dev/sdf
sudo mdadm --grow /dev/md0 --raid-devices=4

# Change the RAID level from RAID 1 to RAID 5
sudo mdadm --grow /dev/md0 --level=5 --raid-devices=3

# Change the chunk size
sudo mdadm --grow /dev/md0 --chunk=512

# Watch the reshape
watch -n 2 cat /proc/mdstat
```

*Tip:* A reshape is a multi-hour operation with a real risk of failure. Back up before you start, and never reshape an array that is already degraded.

## Write-Intent Bitmap

A write-intent bitmap speeds up rebuilds after an unclean shutdown by tracking which blocks have changed.

```bash
# Add an internal bitmap
sudo mdadm --grow /dev/md0 --bitmap=internal

# Add an external bitmap
sudo mdadm --grow /dev/md0 --bitmap=/var/lib/md0.bitmap

# Remove the bitmap
sudo mdadm --grow /dev/md0 --bitmap=none

# Check the bitmap status
cat /proc/mdstat
```

*Tip:* An internal bitmap is the right choice for most arrays. It costs a small amount of write performance but dramatically reduces rebuild time after a crash.

## mdadm --monitor: Alerting on Failures

`mdadm --monitor` watches arrays and sends alerts when a disk fails or a rebuild starts.

```bash
# Run in the foreground
sudo mdadm --monitor --scan --mail=ops@example.com

# Run as a daemon
sudo mdadm --monitor --scan --daemonise --mail=ops@example.com

# Test the alert path
sudo mdadm --monitor --scan --test --mail=ops@example.com
```

*Tip:* `mdadm --monitor --test` sends a test alert, which is the only way to confirm the mail path works before a disk actually fails.

## mdadm.conf: Persistent Configuration

```bash
# /etc/mdadm/mdadm.conf

# Scan for arrays at boot
DEVICE partitions

# Array definitions
ARRAY /dev/md0 metadata=1.2 name=host:0 UUID=12345678:abcdef01:23456789:abcdef01

# Monitoring
MAILADDR ops@example.com
PROGRAM /usr/share/mdadm/checkarray --cron --all --idle --quiet
```

*Tip:* The `ARRAY` line is generated by `mdadm --detail --scan`. Do not hand-write it; a typo means the array will not assemble at boot.

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
