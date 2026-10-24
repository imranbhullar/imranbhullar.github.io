---
layout: post
title: "Monitoring Real-Time System Calls with strace -p: A Practical Guide"
date: Jan-28-2012
categories: [Linux, DevOps]
tags: [linux, strace, troubleshooting, debugging, performance, syscalls]
image: /assets/img/Linux_unix_bsd.png
---

When a process hangs, spins, or fails silently, logs and metrics often tell you nothing useful. `strace -p <PID>` attaches to a running process and shows every system call it makes in real time, which turns a black box into a readable transcript of what the kernel is actually being asked to do.

## What strace Actually Shows You

Every interaction a process has with the outside world goes through a system call: opening files, reading sockets, allocating memory, forking children, waiting on locks. `strace` intercepts those calls with `ptrace` and prints each one with its arguments and return value. That makes it the fastest way to answer questions like "what file is it stuck on?", "is it even trying to connect?", and "why is it looping?".

```bash
# Install on Debian/Ubuntu
sudo apt install strace

# Install on RHEL/CentOS/Fedora
sudo dnf install strace

# Check the version
strace -V
```

*Tip:* `strace` needs `CAP_SYS_PTRACE` or root to attach to another process. If you get `Operation not permitted`, check `sysctl kernel.yama.ptrace_scope` before assuming the tool is broken.

## Attaching to a Running Process

The core use case is attaching to something already running, without restarting it or losing its state.

```bash
# Attach to a PID and print every syscall
sudo strace -p 1234

# Attach and follow forked children too
sudo strace -f -p 1234

# Attach, show timestamps, and time each call
sudo strace -tt -T -p 1234

# Detach cleanly with Ctrl+C, the process keeps running
```

*Tip:* Always start with `-tt -T`. The timestamps tell you when something happened, and the duration tells you where the time went. Without them you are reading a wall of text with no sense of scale.

## Filtering to the Calls That Matter

A busy process produces thousands of syscalls per second. Filtering is not optional, it is the difference between insight and noise.

```bash
# Only file-related syscalls
sudo strace -e trace=file -p 1234

# Only network syscalls
sudo strace -e trace=network -p 1234

# Only a specific set of calls
sudo strace -e trace=openat,read,write,close -p 1234

# Exclude noisy calls
sudo strace -e trace=!futex,poll,epoll_wait -p 1234

# Only calls touching a specific path
sudo strace -e trace=openat -P /etc/myapp/config.yml -p 1234
```

*Tip:* `-e trace=file` is the single most useful filter in practice. Most "the app is stuck" investigations end with a missing config file, a permission error, or a path that does not exist, and all three show up immediately.

## Finding Out Why a Process Is Hanging

A hung process is usually blocked in one syscall. `strace` shows you exactly which one and what it is waiting on.

```bash
# Attach and watch what it is blocked on
sudo strace -tt -T -p 1234

# Common culprits: futex (lock contention), poll/epoll_wait (idle),
# read (waiting on a socket or pipe), connect (waiting on the network)

# Check the current syscall without strace
cat /proc/1234/wchan
cat /proc/1234/stack
```

*Tip:* A process sitting in `futex` is waiting on a lock, not on I/O. That distinction changes the investigation entirely, and it is invisible in `top` or `ps`.

## Diagnosing Slow File and Disk Access

When an application feels slow but CPU is idle, the delay is usually in the filesystem.

```bash
# Time every file syscall
sudo strace -tt -T -e trace=file -p 1234

# Watch a specific process open files during startup
strace -f -e trace=openat,stat,access ./myapp

# Find every file a command touches
strace -f -e trace=openat -o /tmp/opens.log ./myapp
grep -v ENOENT /tmp/opens.log
```

*Tip:* `strace -c` gives you a summary table of syscall counts and total time, which is often enough to spot the one call responsible for most of the latency without reading a single line of output.

```bash
# Summary of syscall counts and time
sudo strace -c -p 1234
```

## Tracing Network Behaviour Without tcpdump

`strace` shows you the application's view of the network, which is often more useful than the packets themselves.

```bash
# Every connect, send, and receive
sudo strace -f -e trace=network -p 1234

# Show full string contents of buffers
sudo strace -s 256 -e trace=network -p 1234

# Trace a curl request from the start
strace -f -e trace=connect,sendto,recvfrom curl https://example.com
```

*Tip:* A `connect` that returns `EINPROGRESS` followed by a long pause is a firewall drop, not a slow server. The timing from `-T` makes that obvious in seconds.

## Following Child Processes and Threads

Modern applications fork and spawn threads constantly. Without `-f` you only see the parent, which is usually the least interesting part.

```bash
# Follow forks and threads
sudo strace -f -p 1234

# Follow forks and prefix each line with the PID
sudo strace -f -p 1234 -o /tmp/trace.log

# Trace a shell script and everything it launches
strace -f -o /tmp/script.log ./deploy.sh
```

*Tip:* When writing to a file with `-o`, each line is prefixed with the PID, which makes it trivial to separate the parent from its children afterwards with `grep`.

## Reducing Overhead and Capturing to a File

`strace` slows the target down, sometimes dramatically. On production systems, capture to a file and keep the trace as narrow as possible.

```bash
# Write to a file instead of the terminal
sudo strace -f -tt -T -o /tmp/trace.log -p 1234

# Limit the string length printed per call
sudo strace -s 128 -p 1234

# Limit the number of calls captured
sudo strace -e trace=openat -p 1234 -o /tmp/trace.log

# Count calls without printing them, minimal overhead
sudo strace -c -p 1234
```

*Tip:* Never leave an unfiltered `strace -f` attached to a busy production process. The overhead can be an order of magnitude, and the log file can fill a disk in minutes.

## Useful Companion Tools

`strace` is the workhorse, but a few related tools answer questions it cannot.

```bash
# Library call tracing, one level up from syscalls
sudo ltrace -p 1234

# Per-process syscall summary over time
sudo pidstat -w 2 5

# Which files a process has open right now
sudo lsof -p 1234

# Kernel-level tracing for deeper analysis
sudo perf trace -p 1234
```

*Tip:* If `strace` shows the process is not making syscalls at all, the time is being spent in userspace. That is when `perf top -p 1234` becomes the right tool instead.

## Bonus: A Repeatable strace Workflow

Most investigations follow the same shape, and having a routine saves time under pressure.

```bash
# 1. Confirm the process is alive and what it is blocked on
ps -o pid,stat,wchan:32,cmd -p 1234

# 2. Attach with timestamps and durations, filtered to file and network
sudo strace -f -tt -T -e trace=file,network -p 1234

# 3. If it is too noisy, capture to a file and analyse afterwards
sudo strace -f -tt -T -o /tmp/trace.log -p 1234
grep -E "ENOENT|EACCES|ETIMEDOUT|ECONNREFUSED" /tmp/trace.log

# 4. Get the summary to see where the time actually went
sudo strace -c -p 1234
```

*Tip:* Grepping the trace for `E` error codes is the fastest way to find the real problem. A process that looks healthy from the outside is often quietly retrying a failing `openat` or `connect` thousands of times per second.
