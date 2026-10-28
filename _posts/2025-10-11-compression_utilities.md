---
layout: post
title: "Linux Archiving & Compression: From Tar to Restic"
date: 2025-10-11
categories: [Linux, SysAdmin]
tags: [LFCS, bash, archiving, backup, devops]
---
![nix](/assets/img/Linux_unix_bsd.png)

Efficiently managing files is a cornerstone of Linux administration. Whether you are backing up a configuration directory like `/etc` or preparing files for a deployment, understanding the nuances of `tar`, `cpio`, and various compression algorithms is essential.

## 1. The Foundation: Tape Archives (tar)

The `tar` (Tape ArChive) command is the most common utility for collecting multiple files into a single archive. By default, `tar` archives are **not** compressed; they simply group files together. You might notice a slightly smaller size than the source due to efficient block allocation, but no actual compression algorithm is applied.

* **-c**: Create a new archive.
* **-v**: Verbose (see the files being added).
* **-f**: Specify the filename.
* **-S**:  Handle sparse files.
```bash
tar -cvf etc.tar /etc
#Decompress (Extract) -x: Extract files from an archive.

tar -xvf etc.tar
```

### Standard Compression

```bash
#-z: Filter the archive through gzip.
tar -czvf etc.tar.gz /etc
#Decompress via tar
tar -xzvf etc.tar.gz

```
### High Efficiency
```bash
#Compress via tar -j: Filter the archive through bzip2.
tar -cjvf etc.tar.bz2 /etc
# Decompress via tar
tar -xjvf etc.tar.bz2
```
### Maximum Compression
```bash
#Compress via tar -J: Filter the archive through xz.
tar -cJvf etc.tar.xz /etc
# Decompress via tar
tar -xJvf etc.tar.xz
```
### Creating a Backup of Sparse Files
Sparse files are files that contain empty blocks, which can take up less disk space. To efficiently create and manage backups of sparse files, you can use the tar command with the --sparse option. Here's a quick guide on how to do this.
```bash
#Creating a Backup of Sparse Files

tar cvSf file.tar.bz2 sparsefile
# Decompress via tar
tar xSf file.tar.bz2
```

## 2. gzip Command

`gzip` compresses files using the **DEFLATE** algorithm and appends a `.gz` extension.

### Compress a File

```bash
gzip file.txt

# Decompress a File
gunzip file.txt.gz
#or
gzip -d file.txt.gz
#Keep Original File While Compressing
gzip -k file.txt

#Compress Multiple Files
gzip file1.log file2.log file3.log
#Compress Recursively
gzip -r /var/log/myapp/
#View Compressed File Contents
zcat file.txt.gz
#or
zless file.txt.gz
```
gzip Compression Levels
```bash
gzip -1 file.txt   # fastest, less compression
gzip -9 file.txt   # best compression

```

## 3.xz Command
`xz` provides very high compression using the LZMA2 algorithm. It is commonly used for kernel sources, backups, and long-term storage.
```bash
#Compress a File
xz file.txt

#Decompress a File
unxz file.txt.xz
#or
xz -d file.txt.xz
# Keep Original File While Compressing
xz -k file.txt
# Compress with Maximum Compression
xz -9 file.txt
# Compress Using Multiple CPU Cores, this will automatically use all available cores
xz -T0 file.txt
#View Contents Without Extracting
xzcat file.txt.xz
#or
xzless file.txt.xz
```



| Feature           | gzip       | xz               |
| ----------------- | ---------- | ---------------- |
| Compression Speed | Fast       | Slower           |
| Compression Ratio | Medium     | Very High        |
| CPU Usage         | Low        | High             |
| Common Usage      | Logs, temp | Archives, source |
| File Extension    | `.gz`      | `.xz`            |

## 4. rsync: The Backup Workhorse

`rsync` is not a compression tool, but it is the tool you will reach for most often when moving or backing up data. It transfers only the differences between source and destination, which makes it dramatically faster than `scp` or `cp` for repeated transfers.

```bash
# Local copy, preserving permissions and timestamps
rsync -av /src/ /dst/

# Remote copy over SSH
rsync -avz /src/ user@host:/dst/

# Dry run, show what would change
rsync -avn /src/ /dst/

# Delete files in destination that no longer exist in source
rsync -av --delete /src/ /dst/

# Exclude patterns
rsync -av --exclude='*.log' --exclude='.git/' /src/ /dst/

# Resume a partial transfer
rsync -av --partial --progress /src/ user@host:/dst/
```

> **Pro Tip:** The trailing slash on the source matters. `rsync -av /src/ /dst/` copies the *contents* of `/src` into `/dst`. `rsync -av /src /dst/` copies the *directory itself*, creating `/dst/src`. This is the single most common rsync mistake.

## 5. dd: Raw Copy and Disk Images

`dd` copies raw bytes. It is the tool for disk images, bootable USB drives, and wiping devices.

```bash
# Write an ISO to a USB drive
sudo dd if=image.iso of=/dev/sdb bs=4M status=progress oflag=sync

# Create a disk image
sudo dd if=/dev/sda of=/backup/sda.img bs=4M status=progress

# Wipe a disk with zeros
sudo dd if=/dev/zero of=/dev/sdb bs=1M status=progress

# Wipe a disk with random data
sudo dd if=/dev/urandom of=/dev/sdb bs=1M status=progress
```

> **Pro Tip:** `dd` has no confirmation prompt and no undo. Double-check `of=` before pressing enter. A single wrong character can destroy a disk.

## 6. Modern Backup Tools: borg and restic

For real backups, `tar` and `rsync` are not enough. You want deduplication, encryption, and retention policies. `borg` and `restic` provide all three.

```bash
# borg: initialise a repository
borg init --encryption=repokey /backup/repo

# borg: create an archive
borg create /backup/repo::backup-{now} /etc /home

# borg: list archives
borg list /backup/repo

# borg: restore
borg extract /backup/repo::backup-2026-01-01

# restic: initialise a repository
restic init --repo /backup/restic

# restic: back up
restic -r /backup/restic backup /etc /home

# restic: list snapshots
restic -r /backup/restic snapshots

# restic: restore
restic -r /backup/restic restore latest --target /restore
```

> **Pro Tip:** Both tools deduplicate at the block level, so a second backup of the same data is nearly free. Run them from cron or a systemd timer, and always test a restore before you trust the backup.

## Quick Reference

```bash
# Compress a directory
tar -czvf archive.tar.gz /etc

# Sync a directory
rsync -av --delete /src/ /dst/

# Write an ISO to USB
sudo dd if=image.iso of=/dev/sdb bs=4M status=progress oflag=sync

# Encrypted, deduplicated backup
restic -r /backup/restic backup /etc /home
```
