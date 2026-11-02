---
layout: post
title: "Persistent Linux Networking: NetworkManager, Netplan, Bonding, and VLANs"
date: Nov-01-2012
categories: [Linux, Networking]
tags: [linux, networking, networkmanager, netplan, vlan]
image: /assets/img/network-config-switch-ethernet.jpg
---

`ip addr add` and `ifconfig` change network state immediately, and just as immediately disappear on reboot. Real network configuration lives in a config file one of a small number of tools is responsible for reading at boot: NetworkManager, netplan, or the older `/etc/network/interfaces`. This post covers all three, plus bonding and VLANs, which sit on top of whichever one you use.

## Why ip/ifconfig Changes Don't Survive a Reboot

Commands run directly against the kernel's networking stack take effect immediately but are never written anywhere persistent — that's the job of a separate configuration layer.

```bash
# This works right now...
sudo ip addr add 192.168.1.50/24 dev eth0

# ...but after a reboot, eth0 is back to whatever NetworkManager/netplan
# configured it as, with no trace of the manual change
```

*Tip:* `ip`/`ifconfig` are the right tools for diagnosing and temporarily patching network state; for anything that needs to survive a reboot, it has to go into the config that the boot-time tool actually manages.

## The Landscape: Which Tool Owns Your Interfaces

Which layer is authoritative depends on the distro and installation type, and running two of them against the same interface causes real conflicts.

```bash
# Check what's actually managing things
systemctl status NetworkManager
systemctl status systemd-networkd
cat /etc/netplan/*.yaml 2>/dev/null

# Debian/older Ubuntu: look here instead
cat /etc/network/interfaces
```

*Tip:* Ubuntu Desktop defaults to NetworkManager; Ubuntu Server defaults to netplan (which itself usually renders down to either NetworkManager or systemd-networkd as a backend). Check which is active before editing config files by hand — editing the wrong layer's file has no effect.

## NetworkManager: nmcli Basics

NetworkManager is driven by `nmcli` on the command line, with connection profiles stored under `/etc/NetworkManager/system-connections/`.

```bash
# List devices and their current connection
nmcli device status

# List all saved connection profiles
nmcli connection show

# Bring a connection up/down
nmcli connection up "Wired connection 1"
nmcli connection down "Wired connection 1"

# Show full detail for one profile
nmcli connection show "Wired connection 1"
```

*Tip:* Every change made with `nmcli connection modify` is written straight to the connection's keyfile and takes effect on the next `connection up` (or immediately, for some properties) — there's no separate "apply" step the way netplan has.

## Configuring a Static IP with NetworkManager

```bash
# Create a new static-IP profile on eth0
nmcli connection add type ethernet con-name static-eth0 ifname eth0 \
  ipv4.addresses 192.168.1.50/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1 8.8.8.8" \
  ipv4.method manual

nmcli connection up static-eth0
```

*Tip:* `ipv4.method manual` is what actually switches the profile off DHCP; setting `ipv4.addresses` alone while `ipv4.method` stays `auto` gets silently overridden by whatever DHCP hands back.

## Netplan: YAML Configuration for Ubuntu Server

Netplan reads YAML from `/etc/netplan/` and renders it into either NetworkManager or systemd-networkd configuration underneath.

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

```bash
# Validate and apply
sudo netplan generate
sudo netplan apply

# See what netplan would actually render, without applying it
sudo netplan get
```

*Tip:* Netplan YAML files are applied in lexical filename order, and later files can override earlier ones — stick to one file per logical purpose (`01-netcfg.yaml`, `90-vlans.yaml`) rather than scattering settings for the same interface across several files.

## /etc/network/interfaces: The Debian Classic

Still found on Debian and older Ubuntu installs or minimal containers, this file predates both netplan and NetworkManager.

```
# /etc/network/interfaces
auto eth0
iface eth0 inet static
    address 192.168.1.50
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 1.1.1.1 8.8.8.8
```

```bash
sudo systemctl restart networking
# or, for just one interface:
sudo ifdown eth0 && sudo ifup eth0
```

*Tip:* `ifup`/`ifdown` and `/etc/network/interfaces` only work if the `ifupdown` package owns the interface — if NetworkManager is also installed and managing the same interface, expect it to fight back and win.

## Bonding: Combining Interfaces for Redundancy or Throughput

A bond presents several physical NICs as one logical interface, either for failover (active-backup) or combined throughput (802.3ad / LACP, which needs matching switch-side configuration).

```yaml
# netplan: active-backup bond of eth0 and eth1
network:
  version: 2
  ethernets:
    eth0: {}
    eth1: {}
  bonds:
    bond0:
      interfaces: [eth0, eth1]
      addresses: [192.168.1.50/24]
      parameters:
        mode: active-backup
        primary: eth0
```

```bash
# Verify bond state after applying
cat /proc/net/bonding/bond0
# Bonding Mode: fault-tolerance (active-backup)
# Currently Active Slave: eth0
# MII Status: up
```

*Tip:* `802.3ad` (LACP) mode gives aggregated throughput, not just failover, but only works correctly if the physical switch ports are also configured as a matching LACP port-channel — plugging into unconfigured switch ports and expecting LACP to negotiate on its own won't work.

## VLANs: Tagged Sub-Interfaces

A VLAN sub-interface lets one physical NIC carry traffic for multiple logical networks, each tagged with an 802.1Q VLAN ID, as long as the upstream switch port is configured as a trunk.

```bash
# Quick, non-persistent VLAN interface with ip link
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 10.0.100.10/24 dev eth0.100
sudo ip link set eth0.100 up
```

```yaml
# netplan: persistent equivalent
network:
  version: 2
  ethernets:
    eth0: {}
  vlans:
    eth0.100:
      id: 100
      link: eth0
      addresses: [10.0.100.10/24]
```

*Tip:* The VLAN ID has to match what the switch port tags that traffic with — a VLAN sub-interface configured correctly on the Linux side but plugged into an access port (or a trunk missing that VLAN) will show `up` but never actually pass traffic.

## Verifying With ip -d link

Once bonds and VLANs are in the mix, `ip -d link show` surfaces the extra detail plain `ip link show` hides.

```bash
ip -d link show bond0
# bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 ...
#     bond mode active-backup active_slave eth0 ...

ip -d link show eth0.100
# eth0.100@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
#     vlan protocol 802.1Q id 100 ...
```

*Tip:* The `@eth0` suffix on a VLAN interface name tells you its parent at a glance, which matters once a box has several bonds and VLANs and `ip link show` output stops being self-explanatory.
