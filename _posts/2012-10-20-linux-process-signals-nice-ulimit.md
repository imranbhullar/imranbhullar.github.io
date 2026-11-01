---
layout: post
title: "Linux Process Control: Signals, trap, nice/ionice, and ulimit"
date: Oct-20-2012
categories: [Linux, SysAdmin]
tags: [linux, signals, trap, nice, ionice, ulimit, process-management]
image: /assets/img/linux-signals-ulimit-terminal.jpg
---

`ps` and `top` show you what's running, but actually controlling a process, telling it to shut down cleanly, limiting how much CPU or memory it can consume, or deciding what happens when it receives an unexpected signal, is a different set of tools entirely. `kill -9` on everything works until the one time it corrupts a database mid-write.

## Why "kill -9 Everything" Is a Bad Habit

`SIGKILL` (signal 9) cannot be caught, blocked, or ignored; the kernel terminates the process immediately, with no chance to clean up open files, flush buffers, or release locks. `SIGTERM` (signal 15), the default `kill` sends, asks the process to shut down and gives it the chance to do so gracefully.

```bash
# Default kill sends SIGTERM
kill 1234

# Explicitly send SIGKILL, only once SIGTERM has had a chance and failed
kill -9 1234

# List every signal name the kernel knows
kill -l
```

*Tip:* Give a process a few seconds after `SIGTERM` before escalating to `SIGKILL`. A script pattern like `kill 1234; sleep 5; kill -0 1234 2>/dev/null && kill -9 1234` automates exactly that courtesy.

## Sending Signals by Name, Pattern, or Process Group

`kill` needs a PID, but `pkill` and `killall` can target processes by name or pattern, which is usually what you actually know.

```bash
# Kill by exact process name
killall nginx

# Kill by pattern match against the full command line
pkill -f "python manage.py runserver"

# Send a specific signal by name
pkill -SIGHUP nginx

# List matching PIDs without killing anything, to check first
pgrep -fl nginx
```

*Tip:* Always run `pgrep` with the same pattern before `pkill` with that pattern. `pkill -f` matches against the full command line, and an overly broad pattern can catch far more processes than you intended.

## The Signals That Actually Matter

A handful of the roughly 64 signals Linux defines come up constantly; the rest are rare enough you can look them up when needed.

```bash
# SIGHUP   (1)  traditionally "terminal closed"; many daemons reload config on it
# SIGINT   (2)  what Ctrl+C sends
# SIGKILL  (9)  immediate, uncatchable termination
# SIGTERM  (15) polite request to terminate, the kill default
# SIGUSR1  (10) and SIGUSR2 (12), free for an application to define its own meaning
# SIGCHLD  (17) sent to a parent when a child process exits

kill -HUP $(pgrep nginx)      # reload nginx config without restarting
kill -USR1 $(pgrep some-app)  # whatever some-app defined USR1 to mean
```

*Tip:* `SIGHUP` triggering a config reload rather than termination is a widely followed convention, not a kernel guarantee. Check a specific daemon's documentation before assuming `SIGHUP` is safe to send.

## Trapping Signals in Shell Scripts

`trap` lets a script run cleanup code when it receives a signal, which is what turns "the script left a lock file behind" into "the script cleaned up after itself even when interrupted."

```bash
#!/bin/bash
LOCKFILE=/tmp/myscript.lock
touch "$LOCKFILE"

cleanup() {
    echo "Cleaning up..."
    rm -f "$LOCKFILE"
    exit 1
}

# Run cleanup on Ctrl+C, termination, or normal exit
trap cleanup SIGINT SIGTERM
trap 'rm -f "$LOCKFILE"' EXIT

# Rest of the script
sleep 300
```

*Tip:* A `trap` on `EXIT` runs no matter how the script ends, normal completion, an error, or a signal, which makes it the most reliable place to put cleanup code that must always run.

## Adjusting CPU Priority with nice and renice

`nice` values run from -20 (highest priority) to 19 (lowest), and they're a hint to the scheduler, not a hard guarantee, about how much CPU time a process should get relative to others.

```bash
# Start a process with lower priority (higher niceness)
nice -n 10 ./long_batch_job.sh

# Start one with higher priority, requires root for negative values
sudo nice -n -5 ./latency_sensitive_task

# Change the priority of an already-running process
renice -n 15 -p 1234

# Check current niceness
ps -o pid,ni,comm -p 1234
```

*Tip:* A background compression or backup job is the textbook case for `nice -n 15` or higher: it still finishes, just without competing for CPU with whatever's actually serving users.

## Adjusting I/O Priority with ionice

CPU isn't the only resource worth deprioritizing. `ionice` does the same job for disk I/O scheduling, which matters more than CPU niceness for I/O-heavy background tasks.

```bash
# Run a backup with the "idle" I/O class, lowest priority, only uses I/O when nothing else wants it
ionice -c 3 rsync -a /data/ /backup/

# Best-effort class with a specific priority level (0-7, lower is higher priority)
ionice -c 2 -n 7 ./some_batch_job.sh

# Check a running process's I/O class and priority
ionice -p 1234
```

*Tip:* `ionice -c 3` (idle class) is the right choice for backup jobs on a spinning disk. It makes the backup effectively invisible to interactive workloads, at the cost of taking longer when the disk is busy.

## Checking and Setting Resource Limits with ulimit

`ulimit` controls what a shell, and anything it launches, is allowed to consume: open files, processes, memory, and more.

```bash
# See all current limits
ulimit -a

# Check the open-file limit specifically
ulimit -n

# Raise it for the current shell session (up to the hard limit)
ulimit -n 4096

# See both soft and hard limits
ulimit -Sn
ulimit -Hn
```

*Tip:* `ulimit` changes made in a shell only apply to that shell and anything it launches afterward. A long-running service started by systemd or at boot needs its limit set elsewhere, which is exactly what the next section covers.

## Persistent Limits with /etc/security/limits.conf

For limits that need to survive a reboot and apply to a specific user or group system-wide, `limits.conf` is the traditional PAM-enforced mechanism.

```bash
# /etc/security/limits.conf
# <domain>  <type>  <item>  <value>
postgres    soft    nofile  8192
postgres    hard    nofile  16384
@developers soft    nproc   2048
```

*Tip:* `limits.conf` is enforced by PAM at login time. It has no effect on services started directly by systemd, which bypasses the login path entirely, so a database service's `ulimit` often needs setting in its systemd unit instead, not here.

## systemd Equivalents

For anything running as a systemd service, the unit file itself is the right place to set resource limits and scheduling priority, not `limits.conf` or a shell `ulimit`.

```ini
# /etc/systemd/system/myapp.service
[Service]
LimitNOFILE=16384
Nice=10
IOSchedulingClass=idle
MemoryMax=2G
CPUQuota=50%
```

*Tip:* `systemctl show myapp -p LimitNOFILE` confirms what limit a running service actually has. It's the fastest way to verify a change in the unit file actually took effect after a `daemon-reload` and restart.

## Bonus: Finding Why a Process Was Killed

A process that vanishes without an obvious cause usually left evidence somewhere, either in the kernel log or in its own exit status.

```bash
# Check the kernel log for OOM killer activity
dmesg | grep -i "killed process"

# journald equivalent
journalctl -k | grep -i oom

# Check the exit code of the last foreground command
echo $?

# Exit codes above 128 indicate death by signal: subtract 128 to get the signal number
# e.g. exit code 137 = 128 + 9 = killed by SIGKILL
```

*Tip:* An exit code of exactly `137` is one of the most useful numbers in troubleshooting: it almost always means the OOM killer or an external `kill -9` ended the process, not a crash in the application's own code.
