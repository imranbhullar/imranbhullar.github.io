---
layout: post
title: "Linux Capabilities: Fine-Grained Privileges Beyond root or Nothing"
date: Sep-22-2012
categories: [Linux, Security]
tags: [linux, capabilities, setcap, security, privileges]
image: /assets/img/linux-capabilities-keys.jpg
---

For decades, Linux privilege was binary: a process ran as root and could do anything, or it ran as a regular user and could do almost nothing that mattered. Capabilities split "anything root can do" into roughly 40 distinct permissions, so a process can bind to port 80 or set the system clock without ever needing full root.

## Why "root or Nothing" Is the Wrong Model

A web server that needs to bind to port 80 traditionally had to start as root just to open that one privileged port, then drop privileges afterward if the developer remembered to. Capabilities let you grant exactly the one permission a program needs, `CAP_NET_BIND_SERVICE`, without the brief window, or the mistake, of running as root at all.

```bash
# See the full list of capabilities the kernel knows about
man capabilities

# Quick list via capsh
capsh --print | grep Current
```

*Tip:* `man 7 capabilities` is one of the most information-dense man pages in the system. Keep it bookmarked; you will refer back to it every time you're deciding which capability a binary actually needs.

## Inspecting the Capabilities a Process Holds

Every running process has several capability sets. The kernel exposes them in `/proc`, but `getpcaps` and `capsh` present the same data far more readably.

```bash
# Capabilities of a running process by PID
getpcaps 1234

# Same information, straight from the kernel
grep Cap /proc/1234/status

# Decode a hex capability bitmask into names
capsh --decode=0000003fffffffff
```

*Tip:* The raw `Cap*` lines in `/proc/PID/status` are hexadecimal bitmasks, not human-readable on their own. Always pipe them through `capsh --decode` rather than trying to remember which bit is which capability.

## The Four Capability Sets

A process doesn't just "have" a capability; it holds it across up to four different sets, and the interaction between them is what actually determines what the process can do.

```bash
# Effective: capabilities currently in use for permission checks
# Permitted: the ceiling of what effective can be raised to
# Inheritable: preserved across execve() to a new program
# Ambient: inherited by child processes without requiring the exec'd binary to be capability-aware

# View all four for the current shell
capsh --print
```

*Tip:* `Ambient` is the newest of the four sets, added specifically so a non-setuid program can pass a capability down to a child process without that child needing to understand capabilities itself.

## Granting a Capability to a Binary with setcap

Instead of making a binary setuid-root, you can attach exactly the capability it needs directly to the executable file.

```bash
# The classic example: ping needs CAP_NET_RAW to send ICMP packets
sudo setcap cap_net_raw+ep /bin/ping

# Grant a capability to bind low ports to a custom binary
sudo setcap cap_net_bind_service+ep /usr/local/bin/mywebserver

# Check what capabilities are set on a file
getcap /bin/ping
```

*Tip:* The `+ep` means "add to the Effective and Permitted sets." Without `e`, the capability is present but never actually active, which is a common reason `setcap` "doesn't work" for someone trying it for the first time.

## Dropping Capabilities for a Safer Shell or Process

`capsh` can also launch a new shell with specific capabilities removed, which is a fast way to test whether a program actually needs the privilege it's been granted.

```bash
# Drop a specific capability and start a new shell without it
sudo capsh --drop=cap_sys_admin -- -c "/bin/bash"

# Drop everything except one capability
capsh --drop=all --keep-caps=cap_net_bind_service -- -c "/bin/bash"
```

*Tip:* Use `capsh --drop` in a test environment before deciding a process genuinely needs a capability. It's a quick way to confirm "this breaks without `CAP_SYS_ADMIN`" instead of granting it on a guess.

## Capabilities in systemd Units

systemd lets you bound and grant capabilities for a service without touching the binary at all, which is the preferred approach for anything you run as a unit.

```ini
# /etc/systemd/system/mywebserver.service
[Service]
ExecStart=/usr/local/bin/mywebserver
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
User=www-data
```

*Tip:* Combine `AmbientCapabilities` with `CapabilityBoundingSet` set to the same narrow list. The bounding set caps what the process could ever acquire, even if something later tries to escalate it; the ambient set is what it actually starts with.

## Capabilities in Containers

Container runtimes drop most capabilities by default and let you re-add only what a specific container needs, which is a large part of what makes containers safer than running the same process directly on the host as root.

```bash
# Docker: add exactly one capability, don't just reach for --privileged
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myimage

# See what a running container currently has
docker inspect --format '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}' mycontainer

# Podman uses the same flags
podman run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myimage
```

*Tip:* `--privileged` grants every capability plus device access, which defeats the entire point of containerizing something in the first place. Reach for `--cap-add` with the one or two specific capabilities you need instead.

## Auditing Capability-Bearing Binaries on a System

A binary with capabilities attached is nearly as interesting, from a security standpoint, as one that's setuid-root. Knowing what's out there is part of a basic hardening review.

```bash
# Find every file on the system with capabilities set
sudo getcap -r / 2>/dev/null

# Compare against setuid binaries, the older mechanism for the same problem
sudo find / -perm -4000 -type f 2>/dev/null
```

*Tip:* Run the `getcap -r /` audit periodically and keep a record of the expected output. A new, unexpected entry appearing after a package update is worth investigating before assuming it's benign.

## Bonus: A Capabilities Cheat Sheet

```bash
# CAP_NET_BIND_SERVICE   bind to ports below 1024
# CAP_NET_RAW            create raw sockets (ping, packet capture)
# CAP_SYS_ADMIN          the infamous "almost root" catch-all, avoid granting this
# CAP_DAC_OVERRIDE       bypass file read/write/execute permission checks
# CAP_CHOWN              change file ownership
# CAP_SYS_TIME           set the system clock
# CAP_KILL               send signals to processes owned by other users
```

*Tip:* `CAP_SYS_ADMIN` covers so many unrelated operations that granting it is barely different from granting full root. If a program "needs `CAP_SYS_ADMIN`," dig further; there's often a narrower, more specific capability that actually covers what it's doing.
