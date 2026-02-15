---
layout: post
title: "Moving from ifconfig to ip: A Cheat Sheet for Sysadmins"
date: 2018-06-09 14:00:00 +0500
description: "The classic ifconfig command is being phased out. Learn how to perform common networking tasks using the modern 'ip' utility."
categories: [Linux, Networking]
tags: [ifconfig, iproute2, sysadmin, terminal, networking]
---

Like most system admins, I love the standard set of Unix commands we've used for decades. One of the most widely used has always been `ifconfig`. However, as many of you have noticed, `ifconfig` is being replaced by the more powerful `ip` command.

While `ifconfig` isn't going anywhere immediately, it is considered legacy. With the `ip` command now standard on almost all modern Linux distributions, it’s time to make the switch. In this post, we’ll look at how to get the same functionality from the `ip` utility that we used to get from `ifconfig` and `route`.

---
![nix](/assets/img/posts/iconfig_ip.png)

### Network Interface Information
To view your network devices and their current configurations.

| Task | Legacy (`ifconfig`) | Modern (`ip`) |
| :--- | :--- | :--- |
| **View all interfaces** | `ifconfig` | `ip addr show` |
| **Enable interface** | `ifconfig eth1 up` | `ip link set eth1 up` |
| **Disable interface** | `ifconfig eth1 down` | `ip link set eth1 down` |

---

### Managing IP Addresses
The `ip` command handles CIDR notation much more naturally than the legacy tools.

#### Basic Assignment
**Legacy:**

```bash
$ ifconfig eth1 172.24.1.1
```

**Modern:**

```bash
$ ip addr add 172.24.1.1/24 dev eth1
```

#### Detailed Assignment (Netmask & Broadcast)
**Legacy:**

```bash
$ ifconfig eth1 172.24.1.1 netmask 255.255.255.0 broadcast 172.24.1.255
```

**Modern:**

```bash
$ ip addr add 172.24.1.1/24 broadcast 172.24.1.255 dev eth1
```
#### Deleting an IP Address
```bash
$ ip addr del 172.24.1.1/24 dev eth1
```
#### Adding an Alias Interface
**Legacy:**
```bash
$ ifconfig eth1:1 172.24.2.0/24
```
**alternate**
```bash
$ ip addr add 172.24.2.0/24 dev eth1 label eth1:1
```

#### Managing the Routing Table

The logic remains similar, but the syntax for ip route is more consistent.

Adding a Route to a Network

**Legacy:**
```bash
$ route add -net 172.24.3.0/24 dev eth2
```

**Alternate:**

```Bash
$ ip route add 172.24.3.0/24 dev eth2
```

Removing a Route

**Legacy:**

```bash
$ route del -net 172.24.3.0/24 dev eth2
```

**Alternate:**

```bash
$ ip route del 172.24.3.0/24 dev eth2
```

Adding a Default Gateway

**Legacy:**

```bash
$ route add -net 172.24.4.0/24 gw 192.168.4.1
```
**Alternate:**

```bash
$ ip route add 172.24.4.0/24 via 192.168.4.1
```

#### Final Thoughts
The ip utility is much more than just a replacement for ifconfig; it is a robust tool that can manage policy routing, tunnels, and more. While ifconfig will likely remain available for some time, mastering the ip command is essential for any modern sysadmin to stay current with the evolving Linux ecosystem.

Happy networking!