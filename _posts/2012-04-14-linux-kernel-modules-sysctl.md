---
layout: post
title: "Linux Kernel and Module Management: lsmod, modprobe, and sysctl"
date: Apr-14-2012
categories: [Linux, SysAdmin]
tags: [linux, kernel, modules, sysctl, tuning, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

The kernel is the layer everything else depends on, and most admins never look at it until something breaks. This post covers the commands for inspecting the running kernel, managing modules, and tuning kernel parameters at runtime and persistently.

## Inspecting the Running Kernel

```bash
# Kernel version
uname -r
uname -a

# Distribution and kernel build info
cat /etc/os-release
cat /proc/version

# Kernel boot parameters
cat /proc/cmdline

# Kernel messages
dmesg -T
```

*Tip:* `uname -r` gives the running kernel version, which is what you need when matching a module or a driver to the kernel. `uname -a` adds architecture and build date.

## Listing and Inspecting Modules

```bash
# All loaded modules
lsmod

# Details about a specific module
modinfo nf_conntrack

# Which module provides a device?
lspci -k

# Which module owns a file?
modinfo -F filename ext4
```

*Tip:* `lsmod` output is just `/proc/modules` formatted. When you need the full picture, `cat /proc/modules` shows the same data with reference counts.

## Loading and Unloading Modules

```bash
# Load a module
sudo modprobe nf_conntrack

# Load with parameters
sudo modprobe nf_conntrack hashsize=65536

# Unload a module
sudo modprobe -r nf_conntrack

# Load a module at boot
echo "nf_conntrack" | sudo tee /etc/modules-load.d/conntrack.conf

# Pass parameters at boot
echo "options nf_conntrack hashsize=65536" | sudo tee /etc/modprobe.d/conntrack.conf
```

*Tip:* `modprobe` resolves dependencies; `insmod` does not. Always prefer `modprobe` unless you have a specific reason to load a module by path.

## Blacklisting Modules

```bash
# Prevent a module from loading
echo "blacklist usb_storage" | sudo tee /etc/modprobe.d/blacklist-usb.conf

# Rebuild the initramfs so the blacklist applies at boot
sudo update-initramfs -u    # Debian/Ubuntu
sudo dracut -f              # RHEL family
```

*Tip:* A blacklist entry alone does not stop a module that is loaded as a dependency. Use `install <module> /bin/false` in the same file to block it completely.

## sysctl: Kernel Parameters

`sysctl` reads and writes kernel parameters exposed under `/proc/sys/`.

```bash
# Read a parameter
sysctl net.ipv4.ip_forward

# List all parameters
sysctl -a

# Set a parameter at runtime
sudo sysctl -w net.ipv4.ip_forward=1

# Apply a file of settings
sudo sysctl -p /etc/sysctl.d/99-tuning.conf
```

*Tip:* `sysctl -w` changes are lost on reboot. To make them persistent, write them to a file under `/etc/sysctl.d/` and run `sysctl --system`.

## Common Tuning Parameters

```bash
# Enable IP forwarding
net.ipv4.ip_forward = 1

# Increase the conntrack table size
net.netfilter.nf_conntrack_max = 262144

# Increase the socket backlog
net.core.somaxconn = 4096

# Reduce swap aggressiveness
vm.swappiness = 10

# Increase the maximum number of open files
fs.file-max = 2097152
```

*Tip:* Change one parameter at a time and measure. Kernel tuning is not a checklist; it is a response to a specific bottleneck you have already identified.

## Persistent Configuration

```bash
# System-wide tuning
sudo vi /etc/sysctl.d/99-tuning.conf

# Apply all files in /etc/sysctl.d/
sudo sysctl --system

# Verify a setting
sysctl net.core.somaxconn
```

*Tip:* Files in `/etc/sysctl.d/` are applied in lexical order. Name your file `99-` something so it wins over distribution defaults.

## sysctl Deep Dive

```bash
# Show all parameters
sysctl -a

# Show a subtree
sysctl net.ipv4

# Show a parameter with its description
sysctl -d net.ipv4.ip_forward

# Write a parameter
sudo sysctl -w net.ipv4.ip_forward=1

# Apply all files in /etc/sysctl.d/
sudo sysctl --system

# Apply a single file
sudo sysctl -p /etc/sysctl.d/99-tuning.conf
```

*Tip:* `sysctl -d` shows the kernel's own description of a parameter, which is often more accurate than the documentation.

## kernel.sysrq: Magic SysRq

```bash
# Enable all SysRq functions
echo 1 | sudo tee /proc/sys/kernel/sysrq

# Sync and reboot safely
echo b | sudo tee /proc/sysrq-trigger

# Sync only
echo s | sudo tee /proc/sysrq-trigger

# Remount read-only
echo u | sudo tee /proc/sysrq-trigger

# Show memory info
echo m | sudo tee /proc/sysrq-trigger
```

*Tip:* `Alt+SysRq+REISUB` is the safe reboot sequence. It syncs, remounts read-only, and reboots, which avoids filesystem corruption on a hung system.

## kernel.panic and kernel.core_pattern

```bash
# Reboot 10 seconds after a panic
echo 10 | sudo tee /proc/sys/kernel/panic

# Reboot on oops
echo 1 | sudo tee /proc/sys/kernel/panic_on_oops

# Send cores to systemd-coredump
echo '|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h' | sudo tee /proc/sys/kernel/core_pattern

# Write cores to a file
echo '/var/crash/core.%e.%p' | sudo tee /proc/sys/kernel/core_pattern
```

*Tip:* `kernel.panic=10` is the standard setting on servers. Without it, a panicked host sits idle until someone notices, which can be hours.

## Quick Reference

```bash
# What kernel am I running?
uname -r

# What modules are loaded?
lsmod

# Load a module
sudo modprobe nf_conntrack

# Read a kernel parameter
sysctl net.ipv4.ip_forward

# Set it persistently
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```

*Tip:* Work in order: identify the kernel, list the modules, load what is missing, then tune parameters. Most kernel incidents are a missing module or a parameter that was set at runtime but never persisted.
