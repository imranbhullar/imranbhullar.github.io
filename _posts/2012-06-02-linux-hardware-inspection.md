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

## dmidecode Deep Dive

```bash
# All DMI types
sudo dmidecode -t all

# BIOS
sudo dmidecode -t bios

# System
sudo dmidecode -t system

# Baseboard
sudo dmidecode -t baseboard

# Chassis
sudo dmidecode -t chassis

# Processor
sudo dmidecode -t processor

# Memory
sudo dmidecode -t memory

# Memory devices only
sudo dmidecode -t 17
```

*Tip:* `dmidecode -t 17` shows the physical DIMMs, their speed, and which slots are populated. It is the only way to know whether you can add RAM without opening the case.

## nvme-cli: NVMe Health and Management

```bash
# Install
sudo apt install nvme-cli

# List NVMe devices
sudo nvme list# Show SMART data
sudo nvme smart-log /dev/nvme0

# Show error log
sudo nvme error-log /dev/nvme0

# Show firmware slot information
sudo nvme fw-log /dev/nvme0

# Run a self-test
sudo nvme self-test /dev/nvme0
```

*Tip:* `nvme smart-log` shows the wear level, the temperature, and the media errors. It is the NVMe equivalent of `smartctl -A`, and it is the only way to see NVMe-specific health data.

## ipmitool: Out-of-Band Management

```bash
# Install
sudo apt install ipmitool

# Show the BMC information
sudo ipmitool mc info

# Show the sensor list
sudo ipmitool sensor list

# Show the chassis status
sudo ipmitool chassis status

# Show the event log
sudo ipmitool sel list

# Power cycle the host
sudo ipmitool chassis power cycle
```

*Tip:* `ipmitool sel list` shows the hardware event log, which is the only record of a hardware failure that happened before the OS booted.

## Redfish: The Modern BMC API

```bash
# Query the Redfish root
curl -sk https://bmc.example.com/redfish/v1/ | jq .

# Show the system inventory
curl -sk https://bmc.example.com/redfish/v1/Systems/1 | jq .

# Show the thermal data
curl -sk https://bmc.example.com/redfish/v1/Chassis/1/Thermal | jq .

# Power cycle
curl -sk -X POST -H 'Content-Type: application/json' \
  -d '{"ResetType":"ForceRestart"}' \
  https://bmc.example.com/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
```

*Tip:* Redfish is the modern replacement for IPMI. It is a REST API, which makes it far easier to automate than the binary IPMI protocol.

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
