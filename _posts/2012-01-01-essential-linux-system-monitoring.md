---
layout: post
title: "Essential Linux System Monitoring and Process Management Tricks"
date: Jan-01-2012
categories: [Linux, DevOps]
tags: [linux, system-monitoring, troubleshooting, performance, processes]
image: /assets/img/Linux_unix_bsd.png
---

When a Linux box starts misbehaving, the clock is already ticking. Every second spent fumbling for the right command is a second of downtime. These are the fast, battle-tested tricks I reach for first when I need to see what the system is actually doing.

## Investigating Open Files and Active Network Ports with lsof

`lsof` ("list open files") shows every file, socket, and device held open by a process. On Linux, everything is a file. You can list everything a single PID has open, find which process is holding a file (handy before unmounting a busy filesystem), or map a listening network port back to its owning process.

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

*Tip:* `lsof +D /mnt/data` recursively lists every open file under a directory, which is perfect for tracking down the culprit when `umount` reports "target is busy".

## Checking Disk Usage Visually and Fast with df, du, and ncdu

`df` answers "how full is each filesystem?" while `du` answers "what is eating the space inside this directory?". Together they cover the two questions you ask first when a disk fills up, and `ncdu` gives you an interactive view on top of `du`.

```bash
# Filesystem-level usage, human-readable, excluding tmpfs and friends
df -hT -x tmpfs -x devtmpfs

# Inodes, which fill up independently of bytes
df -i

# Directory sizes, one level deep, sorted largest first
du -xh --max-depth=1 /var | sort -rh | head -20

# Total size of a single directory
du -sh /var/log
```

`ncdu` is an interactive `du` replacement that lets you drill into directories with the arrow keys instead of re-running commands. It scans a tree you can browse, sorts by size so the biggest offenders surface instantly, and even lets you delete files from within the UI (with confirmation).

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

*Tip:* A full disk is not always a full disk. If `df -h` shows space free but writes still fail, check `df -i` for exhausted inodes, and remember that deleted files held open by a process keep consuming space until that process is restarted or the file is truncated.

## Finding Process Resource Hogs Instantly with ps and awk

`ps` gives you a snapshot; `awk` turns that snapshot into a ranked list. Together they let you sort processes by CPU or memory consumption, filter out kernel threads and noise, and print only the columns you actually care about.

```bash
# Top 10 processes by CPU usage
ps -eo pid,ppid,user,%cpu,%mem,comm --sort=-%cpu | head -n 11

# Top 10 processes by memory usage
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -n 11

# Sum RSS (in MB) per user with awk
ps -eo user,rss --no-headers | awk '{sum[$1]+=$2} END {for (u in sum) printf "%-15s %8.1f MB\n", u, sum[u]/1024}' | sort -k2 -rn
```

*Tip:* `ps aux --sort=-%cpu | head` is the muscle-memory one-liner, but `ps -eo ... --sort` avoids the parsing headaches of the BSD-style output.

For a rolling view rather than a snapshot, `pidstat` reports per-process CPU, memory, and I/O at a fixed interval, which is far more useful than a single `ps` sample when you are chasing a spike.

```bash
# Per-process CPU every 2 seconds, 5 samples
pidstat -u 2 5

# Per-process memory and I/O
pidstat -r -d 2 5

# Disk I/O by process, the fastest way to find a noisy writer
sudo iotop -oPa
```

## Monitoring Log Files in Real-Time with Color Highlighting Using tail and grep

Watching logs live is table stakes; highlighting the lines that matter is what saves your eyes at 3 a.m. Follow a log as it grows, colorize errors, warnings, or specific request IDs, and chain `grep` filters to cut the noise.

```bash
# Follow a log with colored matches
tail -f /var/log/nginx/error.log | grep --color=always -iE "error|warn|critical"

# Follow multiple logs at once, prefixed by filename
tail -f /var/log/syslog /var/log/auth.log

# Watch for a specific request ID across a log
tail -f /var/log/app/app.log | grep --color=always "req_id=abc123"
```

*Tip:* Pipe through `grep --line-buffered` when chaining filters, otherwise output can sit in the buffer and defeat the point of `tail -f`.

On systemd hosts, `journalctl` replaces most of the file hunting and adds filtering by unit, priority, and time window. The cheatsheet below covers the queries you will reach for most often.

```bash
# Follow a specific service's logs
journalctl -u nginx -f

# Only errors and above, since the last boot
journalctl -p err -b

# Everything from the last 15 minutes
journalctl --since "15 min ago"

# Logs between two timestamps
journalctl --since "2026-01-01 00:00" --until "2026-01-01 12:00"

# Logs for the previous boot
journalctl -b -1

# List all recorded boots
journalctl --list-boots

# Logs for a specific PID
journalctl _PID=1234

# Logs for a specific executable
journalctl /usr/sbin/sshd

# Kernel messages only
journalctl -k

# Filter by priority (0=emerg, 3=err, 4=warning, 6=info, 7=debug)
journalctl -p warning

# Show only the last 100 lines
journalctl -n 100

# Disk usage of the journal
journalctl --disk-usage

# Vacuum the journal to a size limit
sudo journalctl --vacuum-size=500M

# Output as JSON for parsing
journalctl -u nginx -o json-pretty

# Disable the pager when piping into grep or less
journalctl -u nginx --no-pager | grep -i error
```

*Tip:* `journalctl -p err -b` is the single most useful command when a service fails to start. It shows every error since the last boot, across all units, in one shot.

## Tracking System Performance Metrics in Real-Time with htop

`htop` is the interactive upgrade to `top`: colorized meters, per-core CPU graphs, and mouse-friendly process management. You get per-core CPU, memory, and swap usage at a glance, can search and filter processes with `F4` / `F3`, and can kill, renice, or trace a process without ever leaving the UI.

```bash
# Install on Debian/Ubuntu
sudo apt install htop

# Launch with a tree view of processes
htop -t

# Filter to a specific user on startup
htop -u www-data
```

*Tip:* Press `F2` → *Display options* → enable *Detailed CPU time* to see steal time. It is the first thing to check when a VM feels slow but nothing inside it looks busy.

When you need the same data without an interactive UI, `vmstat` and `ss` cover the two most common follow-up questions: is the system swapping, and what is actually listening.

```bash
# CPU, memory, swap, and I/O every 2 seconds
vmstat 2

# Listening TCP sockets with the owning process
sudo ss -tlnp

# All established connections, grouped by state
ss -tan state established
```

## Managing Background Jobs and Terminal Processes with jobs and fg

Shell job control lets you juggle long-running tasks in a single terminal without opening new tabs. You can list the jobs started from the current shell, bring a suspended job back to the foreground, or resume one in the background.

```bash
# Start a long task, suspend it with Ctrl+Z, then:
jobs -l          # list jobs with PIDs
bg %1            # resume job 1 in the background
fg %2            # bring job 2 to the foreground
disown %1        # detach job 1 so it survives logout
```

*Tip:* For anything you actually care about surviving a dropped SSH session, use `tmux` or `nohup`. `disown` helps, but a multiplexer is the real answer.

## Modern Monitoring Tools: btop, glances, atop, nmon, and dstat

`htop` is the classic interactive monitor, but the modern alternatives each have a niche.

```bash
# btop: modern, fast, with graphs
sudo apt install btop
btop

# glances: cross-platform, with a web UI
sudo apt install glances
glances
glances -w    # web UI on port 61208

# atop: full system snapshot with per-process detail
sudo apt install atop
sudo atop

# nmon: interactive monitor with per-CPU and per-disk graphs
sudo apt install nmon
nmon

# dstat: combined vmstat + iostat + netstat
sudo apt install dstat
dstat -tcmnd
```

*Tip:* `atop` is the only one of these that records historical data by default, which makes it invaluable when you need to know what happened before you were paged.

## Pressure Stall Information (PSI)

PSI measures how much time tasks are stalled waiting for CPU, memory, or I/O. It is the most accurate signal of real resource pressure.

```bash
# Memory pressure
cat /proc/pressure/memory

# CPU pressure
cat /proc/pressure/cpu

# I/O pressure
cat /proc/pressure/io

# Watch all three live
watch -n 1 'cat /proc/pressure/{cpu,memory,io}'
```

*Tip:* A `full` value above zero means at least one task was stalled. It is a far better signal than load average, which counts uninterruptible sleep as load.

## Added Bonus Command: Checking Available System Memory Instantly with free -h

`free -h` is the fastest way to answer "do we have memory pressure?" without opening a monitoring tool. It gives you human-readable totals for RAM and swap, and the `available` column is the one that matters. The `free` column is misleading on Linux because of caching.

```bash
# Human-readable memory summary
free -h

# Refresh every 2 seconds
free -h -s 2

# Show only the memory line, useful in scripts
free -h | awk '/^Mem:/ {print "Available: " $7}'
```

*Tip:* Ignore the `free` column and watch `available`. Linux deliberately uses spare RAM for page cache, so a low `free` value is usually a healthy system, not a problem.
