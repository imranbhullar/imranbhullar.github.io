---
layout: post
title: "Linux Memory Troubleshooting: RAM, Swap, and OOM Kills"
date: Feb-04-2012
categories: [Linux, DevOps]
tags: [linux, memory, swap, oom, troubleshooting, performance]
image: /assets/img/Linux_unix_bsd.png
---

Memory problems on Linux are deceptive. A host can look perfectly healthy while the kernel is quietly killing your processes, or it can look "out of memory" while half the RAM is sitting in page cache doing useful work. This post walks through the commands and counters that tell you what is actually happening.

## Reading `free` Correctly

The `free` column is the most misunderstood number on a Linux box. Linux deliberately uses spare RAM for page cache, so a low `free` value is usually a healthy system. The column that matters is `available`.

```bash
# Human-readable summary
free -h

# Refresh every 2 seconds
free -h -s 2

# Show only the memory line, useful in scripts
free -h | awk '/^Mem:/ {print "Available: " $7}'
```

*Tip:* If `available` is comfortably above zero, you do not have a memory problem, regardless of what `free` says.

## Finding the Processes Using the Most Memory

Before tuning anything, find out who is consuming the RAM.

```bash
# Top 10 processes by memory usage
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -n 11

# Sum RSS (in MB) per user
ps -eo user,rss --no-headers | awk '{sum[$1]+=$2} END {for (u in sum) printf "%-15s %8.1f MB\n", u, sum[u]/1024}' | sort -k2 -rn

# Per-process memory at a fixed interval
pidstat -r 2 5
```

*Tip:* RSS counts shared pages once per process, so summing RSS across processes overstates real usage. Use `smem` when you need proportional set size (PSS) for an accurate picture.

## Understanding Swap

Swap is not inherently bad. It becomes a problem when the system is actively swapping pages in and out under pressure, which shows up as high `si`/`so` in `vmstat` and high `wa` (I/O wait) in `top`.

```bash
# Swap usage per process
for f in /proc/[0-9]*/status; do awk '/^Name:/{n=$2} /^VmSwap:/{if ($2>0) print $2, n}' "$f"; done | sort -rn | head

# Swap in/out activity every 2 seconds
vmstat 2

# Swap devices and files
swapon --show
cat /proc/swaps
```

*Tip:* A steady `si`/`so` of zero with a full swap device is fine. Non-zero `si`/`so` sustained over time means you are short on RAM.

## Detecting OOM Kills

When the kernel runs out of memory it invokes the OOM killer, which picks a victim and sends `SIGKILL`. The process simply disappears, often with no application-level log entry.

```bash
# Recent OOM kills from the kernel ring buffer
dmesg -T | grep -i -E "killed process|out of memory|oom"

# Same, via the journal
journalctl -k | grep -i oom

# Check the OOM score of a running process
cat /proc/$(pgrep -f myapp | head -1)/oom_score

# Adjust the OOM score (higher = more likely to be killed)
echo 500 | sudo tee /proc/$(pgrep -f myapp | head -1)/oom_score_adj
```

*Tip:* `dmesg -T` prints human-readable timestamps. Correlate the OOM kill time with your application logs to confirm the process was killed rather than crashing on its own.

## Tuning the OOM Killer

You can influence which process the kernel kills, or disable the OOM killer for a specific process entirely.

```bash
# Protect a critical process from being killed
echo -1000 | sudo tee /proc/$(pgrep -f critical-app | head -1)/oom_score_adj

# Check the system-wide overcommit policy
sysctl vm.overcommit_memory
sysctl vm.overcommit_ratio

# Check swappiness (0-100, lower = less aggressive swapping)
sysctl vm.swappiness
```

*Tip:* Setting `oom_score_adj` to `-1000` makes a process effectively immune to the OOM killer. Use it sparingly, and only for processes whose loss would be worse than the OOM event itself.

## smem: Accurate Memory Accounting

RSS counts shared pages once per process, which overstates real usage. `smem` reports PSS (proportional set size), which divides shared pages among the processes that use them.

```bash
# Install
sudo apt install smem

# Per-process PSS
smem -t -p

# Per-user summary
smem -t -u

# Sort by PSS
smem -t -k -c "pid user command pss" | sort -k4 -n
```

*Tip:* When you need to know how much memory a set of processes *actually* uses, `smem` is the correct tool. RSS is fine for a rough ranking, but it double-counts shared libraries.

## Pressure Stall Information (PSI)

PSI measures how much time tasks are stalled waiting for CPU, memory, or I/O. It is the most accurate signal of real resource pressure.

```bash
# Memory pressure
cat /proc/pressure/memory

# CPU pressure
cat /proc/pressure/cpu

# I/O pressure
cat /proc/pressure/io

# Watch memory pressure live
watch -n 1 cat /proc/pressure/memory
```

*Tip:* A `full` value above zero in `/proc/pressure/memory` means at least one task was stalled on memory. It is a far better signal than `free` or swap usage.

## cgroup v2 Memory Limits

On cgroup v2, memory limits are enforced per cgroup, which is how systemd and containers cap memory.

```bash
# Show the memory limit for a service
cat /sys/fs/cgroup/system.slice/myapp.service/memory.max

# Show current usage
cat /sys/fs/cgroup/system.slice/myapp.service/memory.current

# Show the peak usage
cat /sys/fs/cgroup/system.slice/myapp.service/memory.peak

# Show memory events (OOM kills, high watermarks)
cat /sys/fs/cgroup/system.slice/myapp.service/memory.events
```

*Tip:* `memory.events` shows `oom_kill` and `high` counters. A non-zero `oom_kill` confirms the cgroup, not the kernel, killed the process.

## systemd-oomd

`systemd-oomd` kills cgroups under memory pressure before the kernel OOM killer has to act.

```bash
# Is it running?
systemctl status systemd-oomd

# Show the current pressure
systemd-oomdctl

# Check the configured thresholds
cat /etc/systemd/oomd.conf
```

*Tip:* `systemd-oomd` acts on cgroup-level pressure, which is far more targeted than the kernel OOM killer. It is enabled by default on Fedora and many desktop distributions.

## numastat: NUMA Memory Distribution

On multi-socket systems, memory access is not uniform. `numastat` shows how memory is distributed across NUMA nodes.

```bash
# Install
sudo apt install numactl

# Show NUMA topology
numactl --hardware

# Show per-node memory statistics
numastat

# Show per-process NUMA memory
numastat -p 1234
```

*Tip:* A process with most of its memory on a remote NUMA node will be slow even when the system has plenty of free RAM. `numastat -p` shows the distribution per process.

## Quick Reference

```bash
# Is it memory pressure?
free -h

# Who is using it?
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head

# Is it swapping?
vmstat 2

# Was something OOM-killed?
dmesg -T | grep -i oom
```

*Tip:* Work in order: check `available`, find the top consumers, check swap activity, then check for OOM kills. Most memory incidents are resolved by the second step.
