---
layout: post
title: "Essential Linux System Monitoring and Process Management Tricks"
date: Jan-01-2012
categories: [Linux, DevOps]
tags: [linux, system-monitoring, troubleshooting, performance, processes]
image: /assets/img/Linux_unix_bsd.png
---

When a Linux box starts misbehaving, the clock is already ticking — every second spent fumbling for the right command is a second of downtime. These are the fast, battle-tested tricks I reach for first when I need to see what the system is actually doing.

## Investigating Open Files and Active Network Ports with lsof

`lsof` ("list open files") shows every file, socket, and device held open by a process — on Linux, everything is a file.

- List all open files for a given process by PID.
- Find which process is holding a file open (handy before unmounting a busy filesystem).
- Map listening network ports back to their owning process.

```bash
# All open files for PID 1234
lsof -p 1234

# Which process has /var/log/syslog open?
lsof /var/log/syslog

# Who is listening on TCP port 8080?
lsof -iTCP:8080 -sTCP:LISTEN

# All network connections for a specific user
lsof -a -u deploy -i
```

*Tip:* `lsof +D /mnt/data` recursively lists every open file under a directory — perfect for tracking down the culprit when `umount` reports "target is busy".

## Checking Disk Usage Visually and Fast with ncdu

`ncdu` is an interactive `du` replacement that lets you drill into directories with the arrow keys instead of re-running commands.

- Scan a directory tree and browse it interactively.
- Sort by size to surface the biggest offenders instantly.
- Delete files from within the UI (with confirmation).

```bash
# Install on Debian/Ubuntu
sudo apt install ncdu

# Scan the current directory
ncdu

# Scan a specific path, excluding other filesystems
ncdu -x /var

# Export a scan for later inspection
ncdu -o /tmp/var-scan.json /var
```

*Tip:* On servers where you can't install packages, `du -xh --max-depth=1 /var | sort -rh | head -20` gets you 90% of the way there.

## Finding Process Resource Hogs Instantly with ps and awk

`ps` gives you a snapshot; `awk` turns that snapshot into a ranked list.

- Sort processes by CPU or memory consumption.
- Filter out kernel threads and noise.
- Print only the columns you care about.

```bash
# Top 10 processes by CPU usage
ps -eo pid,ppid,user,%cpu,%mem,comm --sort=-%cpu | head -n 11

# Top 10 processes by memory usage
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -n 11

# Sum RSS (in MB) per user with awk
ps -eo user,rss --no-headers | awk '{sum[$1]+=$2} END {for (u in sum) printf "%-15s %8.1f MB\n", u, sum[u]/1024}' | sort -k2 -rn
```

*Tip:* `ps aux --sort=-%cpu | head` is the muscle-memory one-liner, but `ps -eo ... --sort` avoids the parsing headaches of the BSD-style output.

## Monitoring Log Files in Real-Time with Color Highlighting Using tail and grep

Watching logs live is table stakes; highlighting the lines that matter is what saves your eyes at 3 a.m.

- Follow a log file as it grows.
- Highlight errors, warnings, or specific request IDs in color.
- Combine with `grep` filters to cut the noise.

```bash
# Follow a log with colored matches
tail -f /var/log/nginx/error.log | grep --color=always -iE "error|warn|critical"

# Follow multiple logs at once, prefixed by filename
tail -f /var/log/syslog /var/log/auth.log

# Watch for a specific request ID across a log
tail -f /var/log/app/app.log | grep --color=always "req_id=abc123"
```

*Tip:* Pipe through `grep --line-buffered` when chaining filters, otherwise output can sit in the buffer and defeat the point of `tail -f`.

## Tracking System Performance Metrics in Real-Time with htop

`htop` is the interactive upgrade to `top`: colorized meters, per-core CPU graphs, and mouse-friendly process management.

- View per-core CPU, memory, and swap usage at a glance.
- Search and filter processes with `F4` / `F3`.
- Kill, renice, or trace a process without leaving the UI.

```bash
# Install on Debian/Ubuntu
sudo apt install htop

# Launch with a tree view of processes
htop -t

# Filter to a specific user on startup
htop -u www-data
```

*Tip:* Press `F2` → *Display options* → enable *Detailed CPU time* to see steal time — the first thing to check when a VM feels slow but nothing inside it looks busy.

## Managing Background Jobs and Terminal Processes with jobs and fg

Shell job control lets you juggle long-running tasks in a single terminal without opening new tabs.

- List jobs started from the current shell.
- Bring a suspended job back to the foreground.
- Resume a job in the background.

```bash
# Start a long task, suspend it with Ctrl+Z, then:
jobs -l          # list jobs with PIDs
bg %1            # resume job 1 in the background
fg %2            # bring job 2 to the foreground
disown %1        # detach job 1 so it survives logout
```

*Tip:* For anything you actually care about surviving a dropped SSH session, use `tmux` or `nohup` — `disown` helps, but a multiplexer is the real answer.

## Added Bonus Command: Checking Available System Memory Instantly with free -h

`free -h` is the fastest way to answer "do we have memory pressure?" without opening a monitoring tool.

- Human-readable totals for RAM and swap.
- The `available` column is the one that matters — `free` is misleading on Linux because of caching.

```bash
# Human-readable memory summary
free -h

# Refresh every 2 seconds
free -h -s 2

# Show only the memory line, useful in scripts
free -h | awk '/^Mem:/ {print "Available: " $7}'
```

*Tip:* Ignore the `free` column and watch `available` — Linux deliberately uses spare RAM for page cache, and a low `free` value is usually a healthy system, not a problem.
