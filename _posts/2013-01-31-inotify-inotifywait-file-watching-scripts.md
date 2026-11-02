---
layout: post
title: "inotify and inotifywait: Reacting to File Changes on Linux"
date: Jan-31-2013
categories: [Linux, SysAdmin]
tags: [linux, inotify, inotifywait, automation, scripting]
image: /assets/img/inotify-file-folders.jpg
---

Polling a directory in a `while sleep 5` loop works, but it's slow to react and wastes cycles checking nothing has changed. `inotify` is the kernel subsystem that reports filesystem events as they happen, and `inotifywait` is the command-line tool that makes it scriptable without writing any C.

## Kernel Events vs Polling

inotify watches are set on specific files or directories and the kernel pushes events to your process; nothing is checked unless something actually changed.

```bash
# The polling approach this replaces:
while true; do
  find /watched -newer /tmp/last-check -type f
  touch /tmp/last-check
  sleep 5
done
# Works, but has up to a 5-second lag and burns CPU checking idle directories
```

*Tip:* The polling version will eventually miss short-lived files that are created and deleted inside one sleep interval. inotify doesn't have that blind spot, because it's event-driven, not sample-driven.

## Installing inotify-tools

The kernel feature is always present on a modern Linux kernel; the command-line wrappers are a separate package.

```bash
sudo apt install inotify-tools   # Debian/Ubuntu
sudo dnf install inotify-tools   # RHEL/CentOS/Fedora

inotifywait --help | head -5
```

*Tip:* `inotify-tools` provides two binaries: `inotifywait`, which blocks until an event fires, and `inotifywatch`, which gathers event statistics over a time window instead of reacting to them.

## A Basic Watch Loop

The simplest useful pattern: block until something happens, print what happened, then loop back and wait again.

```bash
inotifywait -m /watched/incoming/
# Setting up watches.
# Watches established.
# /watched/incoming/ CREATE report.csv
# /watched/incoming/ CLOSE_WRITE,CLOSE report.csv
```

*Tip:* `-m` (monitor) keeps watching indefinitely and prints every event as it occurs. Without it, `inotifywait` exits after the very first event, which is useful in a script but not for just watching a directory interactively.

## Watching Recursively and Excluding Noise

A single watch only covers one directory; `-r` descends into subdirectories, and `--exclude` keeps generated or temporary files out of the output.

```bash
inotifywait -m -r --exclude '\.(tmp|swp)$' /watched/project/
# Watches every subdirectory under project/ at the time the command starts

# Add new subdirectories after a batch of output? Restart the watch,
# or structure the script to re-run inotifywait after each event
```

*Tip:* `-r` establishes one watch per existing subdirectory at startup. A directory created after that point inside the watched tree won't be covered until you restart `inotifywait`, which matters for scripts expected to run for a long time.

## The Events That Actually Matter

inotify reports far more event types than most scripts need; a handful cover the vast majority of real use cases.

```bash
inotifywait -m -e create -e close_write -e delete -e moved_to /watched/
# -e create      : a new file or directory appeared
# -e close_write : a file that was open for writing just got closed
# -e delete      : a file or directory was removed
# -e moved_to    : something was moved or renamed into this directory
```

*Tip:* Filtering with `-e` cuts down noise from events you don't care about (like `ACCESS` firing every time something merely reads a file) and makes the output of a long-running watch much easier to act on.

## CLOSE_WRITE vs MODIFY: Picking the Right Trigger

`MODIFY` fires on every single write, which for a large file being copied or uploaded can mean dozens of events before it's actually complete.

```bash
# Noisy: fires repeatedly while a file is still being written
inotifywait -m -e modify /watched/uploads/

# Correct: fires exactly once, when the file handle is closed after writing
inotifywait -m -e close_write /watched/uploads/
```

*Tip:* `CLOSE_WRITE` is almost always the event you actually want for "a file just finished arriving." Reacting to `MODIFY` on a file still being copied means processing a half-written file.

## A Practical Watch-and-Process Script

The common real pattern: something drops files into a directory, and a script should pick each one up exactly once, as soon as it's complete.

```bash
#!/usr/bin/env bash
set -euo pipefail

WATCH_DIR="/watched/incoming"
PROCESSED_DIR="/watched/processed"

inotifywait -m -e close_write --format '%f' "$WATCH_DIR" | while read -r filename; do
  src="$WATCH_DIR/$filename"
  echo "Processing: $src"
  # Your actual work goes here, e.g. validate, convert, import
  mv "$src" "$PROCESSED_DIR/$filename"
done
```

*Tip:* Piping into a `while read` loop like this is the standard shape for an inotify-driven worker script. Keep the per-file work idempotent where possible, since a crash mid-loop can leave a file half-processed.

## Running It as a systemd Service

A watch script that's meant to run continuously belongs under systemd, not a detached terminal session or a `nohup` hack.

```bash
# /etc/systemd/system/incoming-watcher.service
[Unit]
Description=Watch incoming/ and process dropped files

[Service]
ExecStart=/usr/local/bin/watch-incoming.sh
Restart=on-failure
User=appuser

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now incoming-watcher.service
journalctl -u incoming-watcher.service -f
```

*Tip:* `Restart=on-failure` matters here: if `inotifywait` ever dies (lost watch, killed process), the whole pipeline stops silently unless systemd is set up to bring it back.

## The max_user_watches Limit

Each watched file or directory consumes one kernel watch; recursively watching a large tree can exhaust the per-user default limit.

```bash
# Check the current limit
cat /proc/sys/fs/inotify/max_user_watches

# inotifywait on a huge tree may fail with:
# Failed to watch /watched/huge-tree/...; upper limit on inotify watches reached

# Raise it
echo "fs.inotify.max_user_watches=524288" | sudo tee /etc/sysctl.d/99-inotify.conf
sudo sysctl --system
```

*Tip:* The default limit (often 8192 or 65536 depending on distro) is tuned for desktop use, not for recursively watching a directory tree with hundreds of thousands of files. Raise it deliberately rather than discovering the ceiling in production.
