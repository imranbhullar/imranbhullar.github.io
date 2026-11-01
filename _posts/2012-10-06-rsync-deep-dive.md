---
layout: post
title: "rsync Deep Dive: Fast, Reliable Backups and Syncs"
date: Oct-06-2012
categories: [Linux, SysAdmin]
tags: [linux, rsync, backup, sync, ssh]
image: /assets/img/rsync-backup-floppy-disks.jpg
---

`cp -r` copies everything, every time, whether it changed or not. `tar` is great for archiving but clumsy for keeping two directories in sync over time. `rsync` transfers only the differences, can resume a broken transfer, and is still, decades later, the tool every serious backup script eventually calls.

## Why rsync Still Beats cp and tar for Backups

rsync's delta-transfer algorithm compares source and destination and sends only the changed blocks, not entire files. Over a slow link or with large files that change only slightly, that difference is the difference between a sync that takes seconds and one that takes hours.

```bash
# Install it, though it's preinstalled on nearly every distro
sudo apt install rsync   # Debian/Ubuntu
sudo dnf install rsync   # RHEL/CentOS/Fedora

# Check the version
rsync --version
```

*Tip:* rsync's delta algorithm only pays off when a file already exists at the destination and has changed. For a first-time copy of all-new data, it has no real advantage over `cp`, beyond its other features like resuming and exclusions.

## Basic Syntax: Local and Remote

The same command works for a purely local copy or a copy to/from a remote host over SSH; only the destination path changes.

```bash
# Local sync
rsync -av /home/alice/projects/ /mnt/backup/projects/

# Push to a remote host over SSH
rsync -av /home/alice/projects/ alice@backup-server:/backups/projects/

# Pull from a remote host
rsync -av alice@backup-server:/backups/projects/ /home/alice/projects/
```

*Tip:* `-a` ("archive") is shorthand for `-rlptgoD`: recursive, preserve symlinks, permissions, times, group, owner, and device files. It's almost always the flag you want as your starting point.

## The Trailing Slash Gotcha

This single detail trips up more rsync users than anything else: whether the source path ends in `/` changes what gets copied.

```bash
# Copies the CONTENTS of projects/ into target/
rsync -av /home/alice/projects/ /mnt/backup/target/

# Copies the projects/ DIRECTORY ITSELF into target/, giving target/projects/
rsync -av /home/alice/projects /mnt/backup/target/
```

*Tip:* "Trailing slash means contents, no trailing slash means the directory itself" is worth memorizing outright. Running a sync without checking this is the single most common cause of files ending up one directory level deeper, or shallower, than intended.

## Dry Runs Before You Trust a Command

`--dry-run` shows exactly what rsync would do without touching a single file, which is essential before running anything with `--delete`.

```bash
# See what would happen, without actually doing it
rsync -av --dry-run /home/alice/projects/ /mnt/backup/projects/

# Add -v twice and --itemize-changes for a detailed per-file breakdown
rsync -avvi --dry-run /home/alice/projects/ /mnt/backup/projects/
```

*Tip:* Make `--dry-run` a reflex for any rsync command involving `--delete` or a remote destination you can't easily inspect afterward. It costs nothing and catches mistakes before they're irreversible.

## Excluding Files

Backups almost never need everything in a directory tree. `--exclude` and `--exclude-from` keep build artifacts, caches, and temp files out of the sync.

```bash
# Exclude a pattern directly
rsync -av --exclude 'node_modules/' --exclude '*.tmp' /home/alice/projects/ /mnt/backup/

# Exclude using a file listing patterns, one per line
rsync -av --exclude-from=/home/alice/.rsync-exclude /home/alice/projects/ /mnt/backup/

# Common contents of .rsync-exclude
cat /home/alice/.rsync-exclude
# node_modules/
# *.log
# .cache/
# __pycache__/
```

*Tip:* Patterns in `--exclude` are matched relative to each directory rsync descends into, not the full path. `node_modules/` excludes it everywhere in the tree, which is exactly what you want for something like that.

## Deleting Files That No Longer Exist at the Source

By default, rsync only adds and updates files; it never removes anything from the destination, even if the source no longer has it. `--delete` changes that.

```bash
# Remove files at the destination that no longer exist at the source
rsync -av --delete /home/alice/projects/ /mnt/backup/projects/

# Delete excluded files at the destination too, not just missing ones
rsync -av --delete-excluded --exclude 'node_modules/' /home/alice/projects/ /mnt/backup/
```

*Tip:* `--delete` turns rsync from "copy forward" into "mirror exactly." Always pair it with `--dry-run` the first time you run a new command, since it will happily delete a destination's worth of files you forgot you needed.

## Bandwidth Limiting and Compression Over SSH

A backup job competing with production traffic for bandwidth is a problem rsync can solve on its own, without a separate traffic-shaping tool.

```bash
# Limit transfer rate to 5000 KB/s
rsync -av --bwlimit=5000 /data/ remote:/backup/

# Compress data in transit, worthwhile over a slow WAN link
rsync -avz /data/ remote:/backup/

# Use a specific SSH port or identity file
rsync -av -e "ssh -p 2222 -i ~/.ssh/backup_key" /data/ remote:/backup/
```

*Tip:* Skip `-z` compression for already-compressed data like JPEGs or video; you'll burn CPU for no benefit. It earns its keep on text, logs, source code, and databases dumps.

## Incremental Backups with --link-dest

`--link-dest` is how you get the effect of daily full backups while only using the disk space of the files that actually changed, by hard-linking unchanged files to a previous backup.

```bash
# First backup, full
rsync -av /data/ /backups/2026-01-01/

# Each subsequent backup, hard-linking unchanged files to yesterday's backup
rsync -av --link-dest=/backups/2026-01-01/ /data/ /backups/2026-01-02/

# Disk usage shows the real story: unchanged files cost nothing extra
du -sh /backups/2026-01-01 /backups/2026-01-02
```

*Tip:* This is exactly the technique tools like Time Machine and `rsnapshot` are built on. Each backup directory looks like a complete, independent snapshot, but unchanged files are hard links, not copies.

## Bonus: Resuming Interrupted Transfers

A dropped connection partway through a large file transfer doesn't have to mean starting over from zero.

```bash
# Keep partially transferred files instead of deleting them on interruption
rsync -av --partial /bigfile remote:/backup/

# Resume a transfer, appending to what's already there and verifying the rest
rsync -av --partial --append-verify /bigfile remote:/backup/

# Combine with a progress bar for long transfers
rsync -av --partial --progress /bigfile remote:/backup/
```

*Tip:* `--append-verify` checksums the already-transferred portion before continuing, which `--append` alone does not. For anything important, the extra verification is worth the small time cost.
