---
layout: post
title: "Linux Performance Analysis: sar, iostat, vmstat, and Load Average"
date: Apr-21-2012
categories: [Linux, DevOps]
tags: [linux, performance, sar, iostat, vmstat, monitoring]
image: /assets/img/Linux_unix_bsd.png
---

Performance problems are rarely what they look like. A "slow database" is often a saturated disk, a "slow API" is often a full conntrack table, and a "high load average" is often a process stuck in uninterruptible sleep. This post covers the tools that separate the symptom from the cause.

## Load Average Explained

Load average is the number of processes in runnable or uninterruptible sleep state, averaged over 1, 5, and 15 minutes. It is **not** CPU utilisation.

```bash
# Current load average
uptime
cat /proc/loadavg

# Number of CPUs, to interpret load
nproc
```

*Tip:* A load average of 4 on a 4-CPU system means the CPUs are fully busy. A load average of 4 on a 16-CPU system means they are mostly idle. Always compare load to CPU count.

## vmstat: The First Tool to Run

```bash
# Every 2 seconds, 5 samples
vmstat 2 5

# With timestamps
vmstat -t 2 5
```

Key columns:

* `r` — runnable processes. Consistently higher than CPU count means CPU saturation.
* `b` — blocked processes. High values mean I/O wait.
* `si` / `so` — swap in/out. Non-zero sustained means memory pressure.
* `wa` — I/O wait percentage. High values mean disk saturation.
* `us` / `sy` — user and system CPU time.

*Tip:* `vmstat` is the single most useful performance command. It answers CPU, memory, swap, and I/O questions in one line of output.

## iostat: Disk Performance

```bash
# Install sysstat
sudo apt install sysstat

# Per-device I/O every 2 seconds
iostat -xz 2

# With timestamps
iostat -xz -t 2
```

Key columns:

* `%util` — device utilisation. Above 80% means the disk is saturated.
* `await` — average wait time per I/O in milliseconds. High values mean slow storage.
* `r/s` / `w/s` — reads and writes per second.
* `rkB/s` / `wkB/s` — throughput.

*Tip:* `%util` near 100% with high `await` is the classic signature of a saturated disk. If `%util` is low but `await` is high, the problem is upstream, not the disk.

## sar: Historical Performance Data

`sar` records performance data over time, which is invaluable when you need to know what happened at 3 a.m.

```bash
# Enable data collection
sudo systemctl enable --now sysstat

# CPU usage today
sar -u

# Memory usage today
sar -r

# Disk I/O today
sar -d

# Network today
sar -n DEV

# Specific time window
sar -u -s 03:00:00 -e 04:00:00
```

*Tip:* `sar` is the only tool that tells you what the system was doing before you were paged. Enable it on every production host.

## perf: CPU Profiling

When you know the CPU is busy but not why, `perf` shows you which functions are consuming it.

```bash
# Install perf
sudo apt install linux-tools-common linux-tools-$(uname -r)

# Top functions by CPU
sudo perf top

# Record and report
sudo perf record -g -p 1234
sudo perf report
```

*Tip:* `perf top` is the fastest way to find a CPU hotspot in a running process. It requires debug symbols for meaningful function names, but even without them the module names are useful.

## iotop: Per-Process I/O

```bash
# Install
sudo apt install iotop

# Only show processes doing I/O
sudo iotop -oPa
```

*Tip:* `iotop -oPa` shows only active processes with accumulated totals. It is the fastest way to find a noisy writer.

## Putting It Together

A typical investigation:

1. `uptime` — is load high relative to CPU count?
2. `vmstat 2` — is it CPU, memory, or I/O?
3. `iostat -xz 2` — is a disk saturated?
4. `iotop -oPa` — which process is doing the I/O?
5. `perf top` — which function is burning CPU?

*Tip:* Work the layers in order: load, then CPU/memory/I/O, then the device, then the process, then the function. Skipping ahead is how you end up restarting a service that was never the problem.

## Pressure Stall Information (PSI)

PSI is the most accurate signal of real resource pressure. It measures how much time tasks are stalled waiting for CPU, memory, or I/O.

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

## perf Flamegraphs

`perf` can record stack traces and render them as a flamegraph, which is the fastest way to find a CPU hotspot.

```bash
# Record with call graphs
sudo perf record -F 99 -g -p 1234 -- sleep 30

# Generate a folded stack file
sudo perf script | stackcollapse-perf.pl > out.folded

# Render a flamegraph
flamegraph.pl out.folded > flamegraph.svg
```

*Tip:* `perf record -F 99 -g` samples at 99 Hz with call graphs, which is the standard recipe. The resulting flamegraph shows exactly which functions are consuming CPU.

## bpftrace: One-Liners for Live Tracing

`bpftrace` is a high-level tracing language built on eBPF. It is the fastest way to answer a specific question about a running system.

```bash
# Install
sudo apt install bpftrace

# Count syscalls by process
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[comm] = count(); }'

# Trace slow disk I/O
sudo bpftrace -e 'tracepoint:block:block_rq_issue { @[comm] = hist(args->bytes); }'

# Trace file opens
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
```

*Tip:* `bpftrace` one-liners are the fastest way to answer "who is doing this?" without writing a kernel module or attaching a debugger.

## ftrace: Kernel Function Tracing

`ftrace` is built into the kernel and traces kernel functions without any extra tooling.

```bash
# Available tracers
cat /sys/kernel/debug/tracing/available_tracers

# Enable function tracing
echo function > /sys/kernel/debug/tracing/current_tracer

# Trace a specific function
echo 'do_sys_open' > /sys/kernel/debug/tracing/set_ftrace_filter

# Read the trace
cat /sys/kernel/debug/tracing/trace

# Disable tracing
echo nop > /sys/kernel/debug/tracing/current_tracer
```

*Tip:* `ftrace` is the lowest-overhead way to trace kernel functions. It is what `perf` and `bpftrace` use under the hood.

## atop, nmon, and dstat

These are the classic all-in-one performance monitors, and each has a niche.

```bash
# atop: full system snapshot with per-process detail
sudo atop

# atop: replay a previous snapshot
sudo atop -r /var/log/atop/atop_20260101

# nmon: interactive monitor with per-CPU and per-disk graphs
nmon

# dstat: combined vmstat + iostat + netstat
dstat -tcmnd
```

*Tip:* `atop` is the only one of the three that records historical data by default, which makes it invaluable when you need to know what happened before you were paged.

## pidstat: Per-Process Statistics

`pidstat` reports per-process CPU, memory, and I/O at a fixed interval, which is far more useful than a single `ps` sample.

```bash
# Per-process CPU every 2 seconds, 5 samples
pidstat -u 2 5

# Per-process memory
pidstat -r 2 5

# Per-process I/O
pidstat -d 2 5

# Per-thread CPU
pidstat -t -u 2 5
```

*Tip:* `pidstat -d` is the fastest way to find a noisy writer without the interactive UI of `iotop`.

## Quick Reference

```bash
# Load vs CPU count
uptime; nproc

# CPU, memory, swap, I/O
vmstat 2

# Disk saturation
iostat -xz 2

# Per-process I/O
sudo iotop -oPa

# Historical data
sar -u
```

*Tip:* `vmstat` and `iostat` together answer 90% of performance questions. Learn to read them and you will rarely need anything else.
