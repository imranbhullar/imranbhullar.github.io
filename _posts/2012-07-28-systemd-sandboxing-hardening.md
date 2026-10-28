---
layout: post
title: "systemd Sandboxing: Hardening Services with Unit Directives"
date: Jul-28-2012
categories: [Linux, Security]
tags: [linux, systemd, hardening, sandboxing, security]
image: /assets/img/Linux_unix_bsd.png
---

systemd can do more than start services. It can restrict what a service can see, what it can write, and which system calls it can make, all without changing the application. These directives are the cheapest security win available on any systemd host.

## The Core Sandboxing Directives

```ini
# /etc/systemd/system/myapp.service
[Service]
# Prevent privilege escalation
NoNewPrivileges=true

# Give the service a private /tmp
PrivateTmp=true

# Give the service a private /dev
PrivateDevices=true

# Mount the filesystem read-only except for specific paths
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp

# Hide /home
ProtectHome=true

# Hide kernel logs and tunables
ProtectKernelLogs=true
ProtectKernelTunables=true
ProtectKernelModules=true

# Restrict the address families the service can use
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Restrict system calls
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
```

*Tip:* `ProtectSystem=strict` makes the entire filesystem read-only except `/dev`, `/proc`, and `/sys`. Add `ReadWritePaths=` for every directory the service legitimately needs to write.

## Restricting Filesystem Access

```ini
# Read-only access to a specific path
ReadOnlyPaths=/etc/myapp

# Inaccessible paths
InaccessiblePaths=/root /home

# Bind-mount a path into the service's namespace
BindPaths=/srv/data:/data

# Bind-mount read-only
BindReadOnlyPaths=/etc/ssl:/etc/ssl
```

*Tip:* `BindPaths=` is the cleanest way to give a service access to a directory outside its normal view without loosening `ProtectSystem`.

## Restricting Network Access

```ini
# Only allow IPv4 and IPv6
RestrictAddressFamilies=AF_INET AF_INET6

# Only allow Unix sockets
RestrictAddressFamilies=AF_UNIX

# Give the service its own network namespace
PrivateNetwork=true

# Bind to a specific port with a capability instead of root
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

*Tip:* `PrivateNetwork=true` gives the service no network at all. It is the right choice for a service that only needs to read local files.

## Restricting System Calls

```ini
# Allow only the system-service set
SystemCallFilter=@system-service

# Deny specific calls
SystemCallFilter=~@mount @debug

# Return EPERM instead of killing the process
SystemCallErrorNumber=EPERM
```

*Tip:* `SystemCallFilter=@system-service` is a good default for most services. It blocks the dangerous calls (`mount`, `reboot`, `ptrace`) while allowing everything a normal service needs.

## Restricting Resources

```ini
# CPU quota
CPUQuota=50%

# Memory limit
MemoryMax=1G

# Task limit
TasksMax=100

# File descriptor limit
LimitNOFILE=65535

# I/O weight
IOWeight=100
```

*Tip:* `MemoryMax=` is enforced by cgroups and is far more reliable than `ulimit`. A service that exceeds it is OOM-killed within its own cgroup, not the whole system.

## Verifying the Sandbox

```bash
# Show the effective settings
systemctl show myapp | grep -E "Protect|Private|Restrict|NoNew"

# Check for sandbox violations
sudo journalctl -u myapp | grep -i -E "denied|blocked|seccomp"

# Use systemd-analyze to check the unit
systemd-analyze verify /etc/systemd/system/myapp.service

# Show the security score
systemd-analyze security myapp
```

*Tip:* `systemd-analyze security myapp` gives the unit a score from 0 to 10 and lists every directive you have not set. It is the fastest way to find the next hardening step.

## A Complete Hardened Unit

```ini
[Unit]
Description=My Hardened Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp
Restart=on-failure
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectSystem=strict
ProtectHome=true
ProtectKernelLogs=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true
LockPersonality=true
MemoryDenyWriteExecute=true
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
ReadWritePaths=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

*Tip:* Apply directives one at a time and test after each. A single over-restrictive directive can break a service in a way that is hard to diagnose.

## Quick Reference

```bash
# Score a unit
systemd-analyze security myapp

# Show effective settings
systemctl show myapp | grep -E "Protect|Private|Restrict"

# Check for violations
sudo journalctl -u myapp | grep -i denied
```

*Tip:* Work in order: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=strict`, then `SystemCallFilter`. Those four cover the majority of the hardening benefit.
