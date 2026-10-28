---
layout: post
title: "journald Persistence and Forwarding: Keeping Logs Across Reboots"
date: Aug-04-2012
categories: [Linux, SysAdmin]
tags: [linux, journald, logging, systemd, rsyslog]
image: /assets/img/Linux_unix_bsd.png
---

By default, the systemd journal is stored in memory and lost on reboot. That is fine for a laptop and unacceptable for a server. This post covers making the journal persistent, capping its size, and forwarding it to a central collector.

## Checking the Current Storage Mode

```bash
# Is the journal persistent?
ls /var/log/journal/

# Current disk usage
journalctl --disk-usage

# Storage mode
grep -E "^Storage" /etc/systemd/journald.conf
```

*Tip:* If `/var/log/journal/` does not exist, the journal is volatile. Creating the directory and restarting `systemd-journald` is enough to make it persistent.

## Making the Journal Persistent

```bash
# Create the persistent journal directory
sudo mkdir -p /var/log/journal

# Set the correct ownership
sudo systemd-tmpfiles --create --prefix /var/log/journal

# Restart journald
sudo systemctl restart systemd-journald

# Verify
journalctl --disk-usage
```

*Tip:* `systemd-tmpfiles --create` sets the right permissions and ACLs. Doing it by hand with `chown` is easy to get wrong.

## Configuring journald

```bash
# /etc/systemd/journald.conf

[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=2G
SystemKeepFree=1G
SystemMaxFileSize=100M
MaxRetentionSec=30day
ForwardToSyslog=yes
```

```bash
# Apply the change
sudo systemctl restart systemd-journald
```

*Tip:* `SystemMaxUse=` caps the total size of the journal. Without it, a chatty service can fill the disk, and the journal is often the last thing anyone checks.

## Vacuuming the Journal

```bash
# Reduce to a size limit
sudo journalctl --vacuum-size=500M

# Reduce to a time limit
sudo journalctl --vacuum-time=30d

# Reduce to a file count
sudo journalctl --vacuum-files=10

# Check the result
journalctl --disk-usage
```

*Tip:* Vacuuming is safe and online. It removes the oldest entries first, so the most recent logs are always preserved.

## Forwarding to rsyslog

```bash
# /etc/systemd/journald.conf
ForwardToSyslog=yes

# Restart both
sudo systemctl restart systemd-journald
sudo systemctl restart rsyslog

# Verify rsyslog is receiving
sudo tail -f /var/log/syslog
```

*Tip:* Forwarding to rsyslog is useful when you already have a syslog pipeline. It duplicates storage, so cap both the journal and the rsyslog files.

## Forwarding to a Remote Collector

```bash
# /etc/systemd/journald.conf
ForwardToSyslog=yes

# /etc/rsyslog.d/50-forward.conf
*.* @@logserver.example.com:514

# Restart
sudo systemctl restart rsyslog
```

*Tip:* journald cannot forward directly over the network. The standard pattern is journald → rsyslog → remote collector, and it works well.

## Querying the Persistent Journal

```bash
# Logs from the previous boot
journalctl -b -1

# List all recorded boots
journalctl --list-boots

# Logs from a specific date
journalctl --since "2026-01-01" --until "2026-01-02"

# Logs for a specific unit across all boots
journalctl -u nginx --no-pager
```

*Tip:* `journalctl --list-boots` is the fastest way to find the boot you need. The index is stable, so `-b -1` always means "the boot before this one".

## systemd-journal-remote: Centralised Collection

`systemd-journal-remote` receives journal entries over HTTP or HTTPS and stores them in a local journal.

```bash
# Install
sudo apt install systemd-journal-remote

# Enable the receiver
sudo systemctl enable --now systemd-journal-remote.socket

# Configure the receiver
# /etc/systemd/journal-remote.conf
# [Remote]
# SplitMode=host
# ServerKeyFile=/etc/ssl/private/journal-remote.key
# ServerCertificateFile=/etc/ssl/certs/journal-remote.crt
# TrustedCertificateFile=/etc/ssl/certs/ca.crt
```

```bash
# On the client: forward to the collector
# /etc/systemd/journald.conf
# [Journal]
# UploadURL=https://logserver.example.com:19532
# ServerKeyFile=/etc/ssl/private/client.key
# ServerCertificateFile=/etc/ssl/certs/client.crt
# TrustedCertificateFile=/etc/ssl/certs/ca.crt
```

*Tip:* `systemd-journal-remote` is the native way to centralise journals. It preserves structured fields, which plain syslog forwarding does not.

## systemd-journal-gatewayd: HTTP Access to the Journal

`systemd-journal-gatewayd` exposes the journal over HTTP, which is useful for dashboards and remote queries.

```bash
# Enable the gateway
sudo systemctl enable --now systemd-journal-gatewayd.socket

# Query the journal over HTTP
curl -s http://localhost:19531/entries?follow

# Query with a filter
curl -s 'http://localhost:19531/entries?unit=nginx.service'
```

*Tip:* `systemd-journal-gatewayd` is read-only and does not require authentication by default. Put it behind a reverse proxy with authentication before exposing it.

## journalctl --output-fields and --facility

```bash
# Show only specific fields
journalctl -o json --output-fields=MESSAGE,PRIORITY,_PID

# Filter by facility
journalctl --facility=auth

# Filter by facility and priority
journalctl --facility=auth -p warning

# List available facilities
journalctl --facility=help
```

*Tip:* `--output-fields` is the fastest way to reduce the volume of `-o json` output. It is essential when piping into `jq`.

## Quick Reference

```bash
# Make the journal persistent
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald

# Cap the size
echo "SystemMaxUse=2G" | sudo tee -a /etc/systemd/journald.conf
sudo systemctl restart systemd-journald

# Vacuum
sudo journalctl --vacuum-size=500M
```

*Tip:* Work in order: make it persistent, cap the size, then forward. Most journal incidents are a volatile journal that lost the evidence, or a persistent one that filled the disk.
