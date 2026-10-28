---
layout: post
title: "Linux Disk Usage Tips: Finding What Filled the Disk"
date: Feb-01-2026
categories: [Linux, SysAdmin]
tags: [linux, disk, df, du, ncdu, lsof, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

A full disk is one of the most common day-to-day Linux problems, and the fix is almost never "delete random files". This post is the practical playbook: find where the space went, find what is holding deleted space, and reclaim it safely.

## Step 1: Confirm the Filesystem Is Actually Full

```bash
# Human-readable usage per filesystem
df -h

# Inodes can run out before bytes do
df -i
```

*Tip:* If `df -h` shows space free but writes fail with `No space left on device`, check `df -i`. A filesystem full of tiny files can exhaust inodes while showing plenty of free bytes.

## Step 2: Find the Big Directories

```bash
# Top-level usage, one level deep
sudo du -xh --max-depth=1 / | sort -rh | head -20

# Drill into the biggest directory
sudo du -xh --max-depth=1 /var | sort -rh | head -20
```

*Tip:* `-x` keeps `du` on one filesystem so it does not wander into mounts. `-h` makes the output readable. Always `sort -rh` so the biggest offenders are at the top.

## Step 3: Use ncdu for Interactive Hunting

```bash
sudo apt install ncdu
sudo ncdu -x /
```

`ncdu` lets you walk the tree, sort by size, and delete files in place. It is far faster than repeated `du` runs when you do not know where the space went.

## Step 4: Find Large Files Directly

```bash
# Files over 1G, largest first
sudo find / -xdev -type f -size +1G -exec ls -lh {} + | sort -k5 -rh | head -20
```

## Step 5: Deleted Files Still Held Open

This is the classic trap: you deleted a huge log, `df` still shows the disk full, because a process still has the file open.

```bash
# Find deleted files still held open
sudo lsof +L1

# Or with the file descriptor size
sudo lsof -nP | grep deleted
```

*Tip:* Restarting the process that holds the file releases the space. Truncating instead of deleting avoids the problem entirely: `sudo truncate -s 0 /var/log/huge.log`.

## Step 6: Reclaim Space from Logs and Journals

```bash
# Journal size and vacuum
journalctl --disk-usage
sudo journalctl --vacuum-size=500M
sudo journalctl --vacuum-time=14d

# Rotate logs now
sudo logrotate -f /etc/logrotate.conf
```

## Step 7: Package and Container Caches

```bash
# Debian family
sudo apt clean
sudo apt autoremove --purge

# RHEL family
sudo dnf clean all

# Docker
docker system df
docker system prune -a --volumes
```

## Quick Checklist

1. `df -h` and `df -i` — confirm what is full.
2. `du -xh --max-depth=1` — find the big directory.
3. `ncdu -x /` — interactive hunt.
4. `lsof +L1` — deleted-but-open files.
5. `journalctl --vacuum-size` and `logrotate -f` — logs.
6. `apt clean` / `dnf clean all` / `docker system prune` — caches.

*Tip:* Set up a simple alert before the disk fills. A cron job that runs `df -h` and mails when usage exceeds 85% saves far more time than any cleanup session.
