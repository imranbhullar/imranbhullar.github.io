---
layout: post
title: "Linux Time and NTP: chrony, timedatectl, and Clock Drift"
date: May-26-2012
categories: [Linux, SysAdmin]
tags: [linux, time, ntp, chrony, timedatectl]
image: /assets/img/Linux_unix_bsd.png
---

Time is the one thing every distributed system depends on and almost nobody configures correctly. A few seconds of drift breaks TLS validation, Kerberos authentication, log correlation, and database replication. This post covers the tools that keep time correct and the commands that diagnose it.

## Checking the Current Time

```bash
# Current date and time
date

# UTC
date -u

# Timezone
timedatectl

# Hardware clock
sudo hwclock --show
```

*Tip:* `timedatectl` shows the local time, UTC, timezone, and whether NTP synchronisation is active, all in one command. It is the first thing to run on any time-related problem.

## timedatectl: The Modern Interface

```bash
# Show status
timedatectl status

# List available timezones
timedatectl list-timezones

# Set the timezone
sudo timedatectl set-timezone Europe/London

# Enable NTP synchronisation
sudo timedatectl set-ntp true

# Set the time manually (disable NTP first)
sudo timedatectl set-ntp false
sudo timedatectl set-time "2026-01-01 12:00:00"
```

*Tip:* `timedatectl set-ntp true` enables `systemd-timesyncd` or `chronyd`, depending on what is installed. It is the correct way to enable NTP on modern systems.

## chrony: The Recommended NTP Client

`chrony` is the default on RHEL family and many Debian systems. It handles intermittent connectivity and large clock jumps better than the old `ntpd`.

```bash
# Is chrony running?
systemctl status chronyd

# Show current synchronisation status
chronyc tracking

# Show configured sources
chronyc sources -v

# Show source statistics
chronyc sourcestats

# Force an immediate step
sudo chronyc makestep
```

*Tip:* `chronyc tracking` shows the offset, the drift rate, and the reference source. An offset of a few milliseconds is normal; an offset of seconds means the clock is not synchronised.

## Configuring chrony

```bash
# /etc/chrony.conf

# Use public NTP servers
pool 2.pool.ntp.org iburst

# Allow large steps at startup
makestep 1.0 3

# Serve time to the local network
allow 10.0.0.0/24

# Local reference clock (if no internet)
# local stratum 10
```

*Tip:* `iburst` sends a burst of packets at startup, which brings the clock into sync much faster than the default. It is almost always worth adding.

## Diagnosing Clock Drift

```bash
# Compare local time to an NTP server
ntpdate -q pool.ntp.org

# Show the offset from chrony
chronyc tracking | grep -E "System time|Last offset"

# Check the hardware clock against the system clock
sudo hwclock --show
date
```

*Tip:* If the system clock is correct but the hardware clock is wrong, the problem is the RTC battery or the `hwclock` configuration, not NTP.

## Time in Containers and VMs

```bash
# Containers share the host clock; check the host
date

# VMs may drift if the hypervisor does not provide time
chronyc tracking

# KVM: enable the paravirtual clock
# Add to the VM XML: <timer name='kvmclock' present='yes'/>
```

*Tip:* Containers do not have their own clock. If a container reports the wrong time, the host is wrong, and fixing the host fixes every container on it.

## Quick Reference

```bash
# What time is it, and is NTP on?
timedatectl

# Is chrony synchronised?
chronyc tracking

# What sources is it using?
chronyc sources -v

# Force a step
sudo chronyc makestep

# Set the timezone
sudo timedatectl set-timezone UTC
```

*Tip:* Work in order: check `timedatectl`, check `chronyc tracking`, check the sources, then check the hardware clock. Most time incidents are a disabled NTP client or a firewall blocking UDP 123.
