---
layout: post
title: "Core Dumps and coredumpctl: Capturing and Analysing Crashes"
date: Jul-21-2012
categories: [Linux, DevOps]
tags: [linux, coredump, coredumpctl, debugging, systemd]
image: /assets/img/Linux_unix_bsd.png
---

When a process crashes, the kernel can write its memory to a core file. That file is the only record of what the process was doing at the moment it died, and on systemd hosts it is managed by `systemd-coredump` and accessed through `coredumpctl`.

## Enabling Core Dumps

```bash
# Check the current limit
ulimit -c

# Enable for the current shell
ulimit -c unlimited

# Enable system-wide
echo '* soft core unlimited' | sudo tee -a /etc/security/limits.conf

# Check where cores go
cat /proc/sys/kernel/core_pattern
```

*Tip:* On systemd hosts, `core_pattern` is usually `|/usr/lib/systemd/systemd-coredump ...`, which means cores go to the journal, not to a file. `coredumpctl` is the way to read them.

## Listing Cores with coredumpctl

```bash
# All recorded cores
coredumpctl list

# Cores for a specific binary
coredumpctl list myapp

# Cores since a specific time
coredumpctl list --since "1 hour ago"

# Details of the most recent core
coredumpctl info
```

*Tip:* `coredumpctl list` shows the PID, the signal, and the time of each crash. It is the fastest way to confirm whether a process is crashing repeatedly.

## Analysing a Core

```bash
# Open the most recent core in gdb
coredumpctl gdb

# Open a specific core
coredumpctl gdb 1234

# Inside gdb:
#   bt
#   info registers
#   thread apply all bt
#   quit
```

*Tip:* `coredumpctl gdb` loads the binary and the core together, so you do not need to find either by hand. It is the fastest path from "it crashed" to a backtrace.

## Extracting a Core File

```bash
# Dump the core to a file
coredumpctl dump 1234 --output /tmp/core.1234

# Dump the most recent core
coredumpctl dump --output /tmp/core.latest
```

*Tip:* Extracting the core is useful when you need to analyse it on a different machine or attach it to a bug report.

## Configuring systemd-coredump

```bash
# /etc/systemd/coredump.conf

[Coredump]
Storage=external
Compress=yes
ProcessSizeMax=2G
ExternalSizeMax=2G
MaxUse=1G
```

```bash
# Apply the change
sudo systemctl restart systemd-coredump.socket
```

*Tip:* `Storage=external` writes cores to `/var/lib/systemd/coredump/`. `Storage=journal` keeps them in the journal, which is convenient but can fill the disk quickly.

## Cleaning Up Cores

```bash
# Remove cores older than 7 days
sudo journalctl --vacuum-time=7d

# Remove all cores
sudo rm -rf /var/lib/systemd/coredump/*

# Check disk usage
du -sh /var/lib/systemd/coredump/
```

*Tip:* Cores are large. A single core can be gigabytes, and a crash loop can fill a disk in minutes. Set `MaxUse` in `coredump.conf` to cap the total size.

## coredumpctl debug: Interactive Debugging

`coredumpctl debug` opens the core in `gdb` with the binary and symbols already loaded.

```bash
# Open the most recent core in gdb
coredumpctl debug

# Open a specific core
coredumpctl debug 1234

# Pass extra arguments to gdb
coredumpctl debug 1234 -- -ex "thread apply all bt"
```

*Tip:* `coredumpctl debug` is the fastest path from "it crashed" to a backtrace. It handles the binary, the core, and the debug symbols in one command.

## systemd-coredump Configuration

```bash
# /etc/systemd/coredump.conf

[Coredump]
Storage=external
Compress=yes
ProcessSizeMax=2G
ExternalSizeMax=2G
MaxUse=1G
KeepFree=1G
```

```bash
# Apply the change
sudo systemctl restart systemd-coredump.socket
```

*Tip:* `KeepFree=` reserves free space on the filesystem, which prevents a crash loop from filling the disk. It is the single most important setting on a production host.

## systemd-pstore: Persistent Storage for Cores

`systemd-pstore` saves kernel oops and panic data from the firmware's persistent storage (pstore) to disk, so it survives a reboot.

```bash
# Is it running?
systemctl status systemd-pstore

# Show saved pstore data
ls /var/lib/systemd/pstore/

# Trigger a save manually
sudo systemctl start systemd-pstore
```

*Tip:* `systemd-pstore` is the only way to recover a kernel panic log after a reboot. It is enabled by default on most modern distributions.

## Quick Reference

```bash
# List cores
coredumpctl list

# Analyse the most recent core
coredumpctl gdb

# Extract a core
coredumpctl dump 1234 --output /tmp/core.1234

# Clean up
sudo journalctl --vacuum-time=7d
```

*Tip:* Work in order: list the cores, open the most recent in gdb, get a backtrace, then fix the bug. Most crash investigations are resolved by `coredumpctl gdb` and `bt`.
