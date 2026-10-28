---
layout: post
title: "Linux Debugging: strace, ltrace, and gdb"
date: Jul-14-2012
categories: [Linux, DevOps]
tags: [linux, debugging, strace, ltrace, gdb, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

When a program misbehaves and the logs are silent, you need to see what it is actually doing. `strace` shows system calls, `ltrace` shows library calls, and `gdb` shows the program's internal state. Together they turn a black box into a readable trace.

## strace: System Calls

```bash
# Trace a command from the start
strace ls /tmp

# Attach to a running process
sudo strace -p 1234

# Follow forks and threads
sudo strace -f -p 1234

# Show timestamps and time spent in each call
sudo strace -tt -T -p 1234

# Trace only network syscalls
sudo strace -f -e trace=network -p 1234

# Trace only file-related syscalls
sudo strace -f -e trace=file -p 1234

# Write the trace to a file
sudo strace -f -o /tmp/trace.log -p 1234
```

*Tip:* `-T` prints the time spent in each syscall. A `connect` that takes 30 seconds is a firewall drop, not a slow server, and the timing makes that obvious immediately.

## Reading strace Output

```text
connect(3, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("93.184.216.34")}, 16) = -1 ETIMEDOUT (Connection timed out)
```

* The syscall name is `connect`.
* The return value is `-1`, meaning failure.
* `ETIMEDOUT` is the error, and it tells you the connection was dropped, not refused.

*Tip:* `ECONNREFUSED` means the port is closed. `ETIMEDOUT` means a firewall or a routing problem. The distinction saves hours.

## ltrace: Library Calls

```bash
# Trace library calls
ltrace ls /tmp

# Attach to a running process
sudo ltrace -p 1234

# Show only calls to a specific library
sudo ltrace -l libc.so.6 -p 1234

# Show timestamps
sudo ltrace -tt -p 1234
```

*Tip:* `ltrace` is useful when the problem is inside a library, not the kernel. If `strace` shows a successful `read` but the program still misbehaves, `ltrace` shows what the library did with the data.

## gdb: The Program's Internal State

```bash
# Start a program under gdb
gdb ./myapp

# Attach to a running process
sudo gdb -p 1234

# Inside gdb:
#   bt          backtrace
#   info threads
#   thread apply all bt
#   frame 3
#   print variable
#   quit
```

*Tip:* `thread apply all bt` is the single most useful gdb command for a hung process. It shows what every thread is doing, which is usually enough to identify the deadlock.

## Debugging a Hung Process

```bash
# Find the PID
pgrep -f myapp

# Attach and dump all thread backtraces
sudo gdb -p $(pgrep -f myapp | head -1) -batch -ex "thread apply all bt"

# Or use pstack, which wraps the same thing
sudo pstack $(pgrep -f myapp | head -1)
```

*Tip:* `gdb -batch -ex "thread apply all bt"` is scriptable and safe to run on a production process. It pauses the process briefly, but it does not kill it.

## Debugging a Crash

```bash
# Enable core dumps
ulimit -c unlimited

# Check where cores go
cat /proc/sys/kernel/core_pattern

# Analyse a core with gdb
gdb ./myapp core.1234

# Inside gdb:
#   bt
#   info registers
#   quit
```

*Tip:* On systemd hosts, cores are usually handled by `systemd-coredump`. Use `coredumpctl list` and `coredumpctl gdb` instead of hunting for a `core` file.

## perf trace: System Calls with Less Overhead

`perf trace` is a lower-overhead alternative to `strace` for system call tracing.

```bash
# Trace a command
sudo perf trace ls /tmp

# Attach to a running process
sudo perf trace -p 1234

# Trace only network syscalls
sudo perf trace -e 'net:*' -p 1234

# Show timestamps
sudo perf trace -p 1234 --duration 10
```

*Tip:* `perf trace` uses kernel tracepoints instead of `ptrace`, which makes it dramatically faster than `strace` on a busy process.

## bpftrace: One-Liners for Live Tracing

`bpftrace` is a high-level tracing language built on eBPF. It is the fastest way to answer a specific question about a running system.

```bash
# Count syscalls by process
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[comm] = count(); }'

# Trace file opens
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# Trace slow disk I/O
sudo bpftrace -e 'tracepoint:block:block_rq_issue { @[comm] = hist(args->bytes); }'
```

*Tip:* `bpftrace` one-liners are the fastest way to answer "who is doing this?" without writing a kernel module or attaching a debugger.

## py-spy: Python Profiling Without Code Changes

`py-spy` profiles a running Python process without modifying it or restarting it.

```bash
# Install
pip install py-spy

# Top-like view of a running process
sudo py-spy top --pid 1234

# Dump the current stack
sudo py-spy dump --pid 1234

# Record a flamegraph
sudo py-spy record -o profile.svg --pid 1234 --duration 30
```

*Tip:* `py-spy dump` is the fastest way to see what a hung Python process is doing. It works on production processes without any code changes.

## gdb Scripting

`gdb` can run a script of commands, which makes it scriptable and repeatable.

```bash
# Run a script of commands
gdb -batch -x commands.gdb ./myapp

# Example commands.gdb
# set pagination off
# break main
# run
# bt
# quit

# Attach and dump all threads
gdb -p 1234 -batch -ex "thread apply all bt"
```

*Tip:* `gdb -batch -ex "thread apply all bt"` is the single most useful gdb invocation for a hung process. It is scriptable and safe to run on production.

## rr: Record and Replay

`rr` records a program's execution and lets you replay it deterministically, which makes debugging a crash far easier.

```bash
# Install
sudo apt install rr

# Record a run
rr record ./myapp

# Replay the last recording
rr replay

# Inside rr:
#   continue
#   reverse-continue
#   break main
```

*Tip:* `rr` lets you step backwards through a crash, which is impossible with a normal debugger. It is the single biggest productivity improvement for debugging complex crashes.

## Quick Reference

```bash
# What syscalls is it making?
sudo strace -f -p 1234

# What is it waiting on?
sudo strace -tt -T -p 1234

# What are the threads doing?
sudo gdb -p 1234 -batch -ex "thread apply all bt"

# Analyse a core
gdb ./myapp core.1234
```

*Tip:* Work in order: `strace` for syscalls, `ltrace` for library calls, `gdb` for internal state. Most "it hangs" incidents are resolved by the first two.
