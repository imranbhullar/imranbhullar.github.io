---
layout: post
title: "systemd-tmpfiles and /tmp Management"
date: Aug-11-2012
categories: [Linux, SysAdmin]
tags: [linux, systemd, tmpfiles, tmp, cleanup]
image: /assets/img/Linux_unix_bsd.png
---

`/tmp` is the directory everyone writes to and nobody cleans up. On systemd hosts, `systemd-tmpfiles` is the tool that creates, cleans, and ages temporary files and directories, and it is the correct way to manage `/tmp`, `/run`, and application-specific scratch space.

## What systemd-tmpfiles Does

`systemd-tmpfiles` reads configuration files from `/usr/lib/tmpfiles.d/`, `/run/tmpfiles.d/`, and `/etc/tmpfiles.d/` and applies them. Each line describes a file or directory to create, clean, or age.

```bash
# Show the current configuration
systemd-tmpfiles --cat-config

# Apply all rules now
sudo systemd-tmpfiles --create

# Clean up expired files
sudo systemd-tmpfiles --clean

# Dry run
sudo systemd-tmpfiles --clean --dry-run
```

*Tip:* `--clean` is what runs on a timer. It removes files older than the age specified in the configuration, which is how `/tmp` stays under control.

## The Configuration Format

```text
# Type  Path              Mode  User  Group  Age  Argument
d       /run/myapp        0755  myapp myapp  -    -
f       /run/myapp/pid    0644  myapp myapp  -    -
r       /run/myapp/stale  -     -     -      -    -
```

* `d` — create a directory.
* `f` — create a file.
* `r` — remove a path.
* `e` — clean a directory, keeping the directory itself.
* `v` — create a subvolume (btrfs).

*Tip:* `e` is the type you want for `/tmp`. It cleans the contents but keeps the directory, which is exactly the behaviour you expect.

## Managing /tmp

```bash
# /etc/tmpfiles.d/tmp.conf

# Clean /tmp files older than 10 days
e /tmp 1777 root root 10d

# Clean /var/tmp files older than 30 days
e /var/tmp 1777 root root 30d
```

*Tip:* The default on most distributions is 10 days for `/tmp` and 30 days for `/var/tmp`. Shortening it is a common response to a full disk.

## Creating Application Directories

```bash
# /etc/tmpfiles.d/myapp.conf

# Create the runtime directory
d /run/myapp 0755 myapp myapp -

# Create the state directory
d /var/lib/myapp 0750 myapp myapp -

# Create a lock file
f /run/myapp/lock 0644 myapp myapp -
```

*Tip:* `RuntimeDirectory=` and `StateDirectory=` in a systemd unit do the same thing more cleanly. Use `tmpfiles.d` when you need the directory to exist before the service starts.

## The Cleanup Timer

```bash
# Is the timer active?
systemctl status systemd-tmpfiles-clean.timer

# When does it run?
systemctl list-timers systemd-tmpfiles-clean.timer

# Run it manually
sudo systemctl start systemd-tmpfiles-clean.service
```

*Tip:* The timer runs daily by default. If `/tmp` fills faster than that, shorten the age in `tmpfiles.d` rather than running the cleanup more often.

## PrivateTmp for Services

```ini
# /etc/systemd/system/myapp.service
[Service]
PrivateTmp=true
```

*Tip:* `PrivateTmp=true` gives the service its own `/tmp` and `/var/tmp`, invisible to other processes. It is the cleanest way to stop a service from interfering with system temporary files.

## Quick Reference

```bash
# Show the configuration
systemd-tmpfiles --cat-config

# Apply rules
sudo systemd-tmpfiles --create

# Clean expired files
sudo systemd-tmpfiles --clean

# Run the cleanup timer now
sudo systemctl start systemd-tmpfiles-clean.service
```

*Tip:* Work in order: check the configuration, check the timer, then shorten the age if `/tmp` fills too fast. Most `/tmp` incidents are a missing cleanup rule or a service writing gigabytes of scratch files.
