---
layout: post
title: "Linux Hardware Inspection: lscpu, lsblk, lspci, lsusb, and smartctl"
date: Jun-02-2012
categories: [Linux, SysAdmin]
tags: [linux, hardware, lscpu, lspci, smartctl, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

When you inherit a server, or when hardware starts failing, you need to know exactly what is inside the box. Linux exposes all of it through a handful of commands, most of which read from `/sys` and `/proc`.

## CPU

```bash
# Summary of the CPU
lscpu

# Per-core details
cat /proc/cpuinfo

# Number of logical CPUs
nproc

# CPU flags (useful for virtualisation and crypto)
grep -m1 flags /proc/cpuinfo
```

*Tip:* `lscpu` shows sockets, cores per socket, threads per core, and NUMA nodes in one view. It is far more readable than `/proc/cpuinfo`.

## Memory

```bash
# Summary
free -h

# Detailed hardware info
sudo dmidecode -t memory

# NUMA topology
numactl --hardware
```

*Tip:* `dmidecode -t memory` shows the physical DIMMs, their speed, and which slots are populated. It is the only way to know whether you can add RAM without opening the case.

## Disks and Storage

```bash
# Block devices in a tree
lsblk -f

# Detailed disk info
sudo hdparm -I /dev/sda

# SMART health
sudo smartctl -H /dev/sda

# SMART attributes
sudo smartctl -A /dev/sda

# Run a short self-test
sudo smartctl -t short /dev/sda

# Show the test results
sudo smartctl -l selftest /dev/sda
```

*Tip:* `smartctl -H` returns `PASSED` or `FAILED`. A `FAILED` result means the disk is dying and should be replaced immediately, regardless of what the filesystem says.

## PCI Devices

```bash
# All PCI devices
lspci

# With kernel driver in use
lspci -k

# Verbose
lspci -vv

# Network controllers only
lspci | grep -i ethernet
```

*Tip:* `lspci -k` shows which kernel driver is bound to each device. It is the fastest way to confirm whether a NIC or HBA is using the driver you expect.

## USB Devices

```bash
# All USB devices
lsusb

# Verbose
lsusb -v

# Tree view
lsusb -t
```

*Tip:* `lsusb -t` shows the USB topology, including hubs and the speeds of each device. It is useful when a device works on one port but not another.

## Sensors and Temperatures

```bash
# Install lm-sensors
sudo apt install lm-sensors

# Detect sensors
sudo sensors-detect

# Read temperatures, voltages, and fan speeds
sensors
```

*Tip:* `sensors` is the fastest way to check whether a server is overheating. High CPU temperatures under load usually mean a failed fan or dried thermal paste, not a software problem.

## Firmware and BIOS

```bash
# BIOS version and date
sudo dmidecode -t bios

# System manufacturer and model
sudo dmidecode -t system

# Firmware update tool (vendor-specific)
sudo fwupdmgr get-devices
sudo fwupdmgr refresh
sudo fwupdmgr update
```

*Tip:* `fwupdmgr` is the standard way to update firmware on supported hardware. It is far safer than vendor-specific tools and works across most modern servers and laptops.

## Quick Reference

```bash
# CPU
lscpu

# Memory
sudo dmidecode -t memory

# Disks
lsblk -f
sudo smartctl -H /dev/sda

# PCI
lspci -k

# USB
lsusb -t

# Temperatures
sensors
```

*Tip:* Work in order: CPU, memory, disks, PCI, USB, sensors. Most hardware incidents are a failing disk (`smartctl -H`) or an overheating CPU (`sensors`), and both are visible in seconds.
