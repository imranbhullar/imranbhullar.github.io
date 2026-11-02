---
layout: post
title: "lsof, ss, and netstat: Inspecting Open Files and Sockets on Linux"
date: Dec-06-2012
categories: [Linux, SysAdmin]
tags: [linux, lsof, ss, netstat, troubleshooting]
image: /assets/img/lsof-ss-netstat-terminal.jpg
---

"Address already in use" on a port you're sure is free. A disk at 100% that `du` can't account for. A service you want to restart but can't be sure what else is touching its files first. All three are answered by the same question: what does this process actually have open right now?

## netstat Is Deprecated, ss Replaced It

netstat still works on most distros, but it's unmaintained, and `ss` (socket statistics) reads the same kernel data faster and with better filtering.

```bash
# Old way, still common in scripts and muscle memory
netstat -tulpn

# Current way, same information, faster on busy hosts
ss -tulpn
```

*Tip:* If a distro doesn't ship `net-tools` by default anymore (several don't), `netstat` simply won't be there. Learn `ss` as the primary tool and treat `netstat` as something you might see in an old script.

## Listing Listening Ports

The single most common use of `ss`: what's bound to what, and which process owns it.

```bash
# -t tcp, -u udp, -l listening only, -p show process, -n don't resolve names
ss -tulpn

# Just TCP, listening sockets
ss -tlpn
```

*Tip:* The `Local Address:Port` column showing `0.0.0.0` means the service listens on every interface; `127.0.0.1` means localhost only. That distinction alone explains a lot of "why can't I connect from another host" tickets.

## Finding Which Process Holds a File Open

`lsof` (list open files) answers "who has this open" for any path, not just network sockets, everything on Linux is a file.

```bash
# Who has this specific file open
lsof /var/log/app.log

# Everything a given process has open
lsof -p 1234

# Every open file under a directory
lsof +D /var/www
```

*Tip:* `lsof +D` walks the whole tree and can be slow on a directory with thousands of files. Prefer `lsof /exact/path` when you already know the file.

## The Deleted-But-Still-Open Disk Usage Mystery

`df` says a filesystem is full; `du` can't find where the space went. This is almost always a deleted file some process still has open.

```bash
# Find open files marked (deleted)
lsof | grep deleted

# Example output:
# app       1234  www-data  3w  REG  8,1  2147483648  12345  /var/log/app.log (deleted)
```

*Tip:* Deleting a log file while the writing process keeps its file descriptor open doesn't free the space, the inode isn't released until the process closes it or exits. Restart the process, or `truncate` the file instead of deleting it next time.

## Finding the Process Bound to a Port

Before restarting a service that won't start because its port is "already in use," find out what's actually using it.

```bash
# Using ss
ss -tlpn | grep :8080

# Using lsof, often more readable for this specific question
sudo lsof -i :8080
```

*Tip:* A previous instance of the same service hung in a zombie or stuck state is the most common reason for this. Confirm the PID with `ps -p <pid>` before killing it, to make sure it's actually what you think it is.

## lsof -i for Network Connections

Beyond listening ports, `lsof -i` shows established connections too, filtered the same way `ss` filters by protocol and state.

```bash
# All network activity for a process
lsof -i -a -p 1234

# All established connections to a specific host
lsof -i @10.0.0.15

# Only TCP, only a specific port
lsof -i tcp:443
```

*Tip:* `-a` makes lsof AND multiple filters together rather than OR them; without it, `-i` and `-p` combined would show everything matching either condition, which is rarely what you want.

## Killing a Process by What It Has Open

Sometimes the fastest fix is identifying and ending whatever's misbehaving, directly from the lsof output.

```bash
# Find the PID holding a port, then kill it
sudo lsof -t -i :8080
sudo kill $(sudo lsof -t -i :8080)
```

*Tip:* `-t` on lsof prints bare PIDs with no formatting, built specifically to feed into `kill` or `xargs`. It's the difference between reading output and piping it.

## ss Socket States

Beyond listening sockets, `ss` shows connection state, which matters for diagnosing things like a backlog of unclosed connections.

```bash
# All TCP sockets with state
ss -tan

# Just connections stuck in TIME_WAIT
ss -tan state time-wait

# Count connections by state
ss -tan | awk '{print $1}' | sort | uniq -c
```

*Tip:* A large number of `CLOSE_WAIT` sockets usually points at the application, not the network, it means the remote side closed the connection but the local app never called `close()`.

## Quick Reference: Which Tool for Which Question

Each tool has a sweet spot; reaching for the right one first saves a step.

```text
"What's listening on this port?"        -> ss -tlpn
"What process has this file open?"      -> lsof /path
"What's stuck using space I can't see?" -> lsof | grep deleted
"What state are my connections in?"     -> ss -tan
"Old script already uses netstat?"      -> it'll still work, just slower
```

*Tip:* Keep `ss` as the default reflex for anything port- or connection-related, and reach for `lsof` the moment the question is about a specific file or a specific process's full file-descriptor table.
