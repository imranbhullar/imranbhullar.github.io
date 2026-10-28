---
layout: post
title: "Linux Networking and Troubleshooting: The Commands a Sysadmin Actually Uses"
date: Jan-14-2012
categories: [Linux, Networking]
tags: [linux, networking, troubleshooting, dns, tcpdump, performance]
image: /assets/img/Linux_unix_bsd.png
---

Networking problems rarely announce themselves politely. A service is "down", a deploy "hangs", and the real cause is a stale ARP entry, a full conntrack table, or a DNS resolver that answers in 5 seconds instead of 5 milliseconds. These are the commands I reach for first when I need to prove where the packets are actually going.

## Starting With the Interface and Address Layer

Before you blame the network, confirm the host itself is configured the way you think it is. `ip` has replaced `ifconfig`, `route`, and `arp` on every modern distribution.

```bash
# All interfaces with addresses and state
ip -br addr

# Detailed view of a single interface
ip addr show eth0

# Link state, speed, and duplex
ip -s link show eth0

# Routing table
ip route

# Which route would be used to reach a destination
ip route get 8.8.8.8

# ARP and neighbour table
ip neigh
```

*Tip:* `ip -br addr` is the one command to run first on any unfamiliar box. It shows every interface, its state, and its addresses in a single compact table, which immediately reveals a missing address or a `DOWN` link.

## Testing Reachability Without Guessing

`ping` proves basic reachability, but it is only the first step. `traceroute` and `mtr` show where the path breaks, and `nc` proves whether a specific port is actually open.

```bash
# Basic reachability, stop after 4 packets
ping -c 4 8.8.8.8

# Ping by hostname to test DNS resolution at the same time
ping -c 4 example.com

# Trace the path, using ICMP
traceroute example.com

# Continuous path analysis with per-hop loss and latency
mtr -rw example.com

# Test whether a TCP port is open
nc -zv example.com 443

# Test a range of ports
nc -zv example.com 8000-8100
```

*Tip:* `mtr -rw` gives you a single report instead of an interactive UI, which is exactly what you want when pasting evidence into a ticket or a chat.

## Checking Listening Ports and Active Connections

Knowing what is listening and who is connected is the fastest way to distinguish "the service is not running" from "the firewall is blocking it".

```bash
# Listening TCP and UDP sockets with owning process
sudo ss -tulpn

# All established TCP connections
ss -tan state established

# Connections to a specific port
ss -tan '( sport = :443 or dport = :443 )'

# Socket summary by state
ss -s

# Legacy equivalent, still useful on older hosts
netstat -tulpn
```

*Tip:* `ss -tulpn` is the modern replacement for `netstat -tulpn` and is dramatically faster on hosts with thousands of connections. If `ss` is missing, install `iproute2`.

## Resolving DNS Problems Methodically

DNS is the most common cause of "the network is slow" complaints. Test resolution at each layer: the resolver config, the query itself, and the response time.

```bash
# What resolver is configured
cat /etc/resolv.conf

# Resolve a name using the system resolver
getent hosts example.com

# Query a specific record type
dig example.com A
dig example.com MX
dig example.com TXT

# Query a specific DNS server directly
dig @8.8.8.8 example.com

# Show the full resolution path including CNAMEs
dig +trace example.com

# Measure resolver latency
dig example.com | grep "Query time"
```

*Tip:* `getent hosts` uses the same NSS path as your applications, including `/etc/hosts` and LDAP. If `dig` works but `getent` fails, the problem is in `/etc/nsswitch.conf`, not in DNS.

## Capturing and Inspecting Traffic with tcpdump

When logs and counters are not enough, capture the packets. `tcpdump` is the tool that ends arguments.

```bash
# Capture on an interface, write to a file for later analysis
sudo tcpdump -i eth0 -w /tmp/capture.pcap

# Capture only traffic to or from a host
sudo tcpdump -i any host 10.0.0.5

# Capture a specific port, with readable output
sudo tcpdump -i any -nn port 443

# Capture and print packet contents in ASCII
sudo tcpdump -i any -A port 80

# Read a capture back with timestamps
sudo tcpdump -r /tmp/capture.pcap -nn
```

*Tip:* Always capture with `-w` and analyse the file afterwards. Live `tcpdump` output drops packets under load, and a saved `.pcap` can be opened in Wireshark where the real analysis happens.

## Measuring Latency, Loss, and Bandwidth

Sometimes the network works but is slow. These tools separate latency from loss from throughput.

```bash
# Round-trip latency and jitter to a host
ping -c 20 example.com

# Per-hop latency and loss
mtr -rwc 20 example.com

# TCP throughput test against a public server
iperf3 -c iperf.example.com

# Server side of the same test
iperf3 -s

# HTTP request timing breakdown
curl -o /dev/null -s -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} total:%{time_total}\n" https://example.com
```

*Tip:* The `curl -w` timing breakdown is the single most useful command for diagnosing slow APIs. It tells you instantly whether the delay is in DNS, TCP connect, TLS handshake, or the server itself.

## Inspecting Firewall Rules and NAT

A surprising number of "network outages" are firewall rules doing exactly what they were told.

```bash
# nftables ruleset
sudo nft list ruleset

# iptables rules with packet counters
sudo iptables -L -n -v

# NAT table specifically
sudo iptables -t nat -L -n -v

# firewalld zones and services
sudo firewall-cmd --list-all

# UFW status with rules
sudo ufw status verbose
```

*Tip:* Always check counters with `-v`. A rule with zero matches is not the rule that is blocking you, and chasing it wastes time.

## Tracing a Connection End to End with strace

When an application claims the network is broken but the network looks fine, `strace` shows you exactly what the process is doing.

```bash
# Trace network syscalls for a running process
sudo strace -f -e trace=network -p 1234

# Trace a command from the start
strace -f -e trace=connect,sendto,recvfrom curl https://example.com

# Show timestamps to spot slow calls
strace -tt -T -e trace=network -p 1234
```

*Tip:* `-T` prints the time spent in each syscall. A `connect` that takes 30 seconds is a firewall drop, not a slow server, and the timing makes that obvious immediately.

## Checking Kernel Network Counters and Tuning

The kernel keeps detailed counters for every protocol. They are the first place to look when connections fail under load.

```bash
# Protocol-level statistics
netstat -s

# Conntrack table usage, a classic cause of dropped connections
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Socket buffer and backlog settings
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog

# Interface error and drop counters
ip -s link show eth0
```

*Tip:* A full conntrack table silently drops new connections while everything else looks healthy. If `nf_conntrack_count` is close to `nf_conntrack_max`, raise the limit before you debug anything else.

## Bonus: Quick Reference for the Most Common Failures

Most network incidents fall into a small number of categories, and each has a fast first check.

```bash
# Is it DNS?
dig +short example.com

# Is it routing?
ip route get 8.8.8.8

# Is it the port?
nc -zv example.com 443

# Is it the firewall?
sudo nft list ruleset | grep -i drop

# Is it the application?
strace -f -e trace=network -p $(pgrep -f myapp | head -1)
```

*Tip:* Work the layers in order: link, address, route, port, application. Skipping ahead is how you end up restarting a service that was never the problem.

## Advanced `ip`: VLANs, Bonds, and Bridges

The `ip` command covers far more than addresses and routes. These are the constructs you will meet on any host with more than one NIC.

```bash
# Create a VLAN interface on eth0 with VLAN ID 100
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 10.0.100.5/24 dev eth0.100
sudo ip link set eth0.100 up

# Create a bond from two interfaces
sudo ip link add bond0 type bond mode active-backup
sudo ip link set eth0 master bond0
sudo ip link set eth1 master bond0
sudo ip addr add 10.0.0.5/24 dev bond0
sudo ip link set bond0 up

# Create a bridge and attach an interface
sudo ip link add br0 type bridge
sudo ip link set eth0 master br0
sudo ip addr add 10.0.0.5/24 dev br0
sudo ip link set br0 up
```

*Tip:* These commands are not persistent. On RHEL family systems, use `nmcli` or `ifcfg` files; on Debian family, use `/etc/network/interfaces` or `netplan`. The `ip` commands are for testing and for understanding what the config files produce.

## Policy Routing with `ip rule`

The main routing table is only one of 256 tables. Policy routing lets you send traffic to different tables based on source address, mark, or interface.

```bash
# List the routing policy rules
ip rule show

# Add a rule: traffic from 10.0.0.0/24 uses table 100
sudo ip rule add from 10.0.0.0/24 table 100

# Add a route to table 100
sudo ip route add default via 10.0.0.1 table 100

# Delete a rule
sudo ip rule del from 10.0.0.0/24 table 100

# Show a specific table
ip route show table 100
```

*Tip:* Policy routing is how you run multiple uplinks or route VPN traffic selectively. `ip rule show` is the first command to run when traffic takes an unexpected path.

## VRFs: Virtual Routing and Forwarding

A VRF is a separate routing table attached to a set of interfaces, which is how you isolate routing domains on a single host.

```bash
# Create a VRF
sudo ip link add vrf-blue type vrf table 100
sudo ip link set vrf-blue up

# Attach an interface to the VRF
sudo ip link set eth1 master vrf-blue

# Show the VRF's routing table
ip route show table 100

# Show VRF membership
ip -d link show type vrf
```

*Tip:* VRFs are the Linux equivalent of a VRF on a router. They are the cleanest way to run overlapping address space on a single host.

## systemd-networkd and netplan

On servers, `systemd-networkd` or `netplan` often owns the configuration instead of NetworkManager.

```bash
# systemd-networkd status
networkctl status

# Reload networkd
sudo networkctl reload

# netplan: show the current config
netplan get

# netplan: apply a config
sudo netplan apply

# netplan: generate backend configs without applying
sudo netplan generate
```

*Tip:* `netplan apply` is the correct way to apply a change on Ubuntu. Editing the generated files under `/run/systemd/network/` directly is overwritten on the next apply.

## Wireless: iw and wpa_supplicant

```bash
# List wireless interfaces
iw dev

# Scan for networks
sudo iw dev wlan0 scan | grep SSID

# Show link quality
iw dev wlan0 link

# Connect with wpa_supplicant
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf

# Get a DHCP lease
sudo dhclient wlan0
```

*Tip:* `iw dev wlan0 link` shows the signal strength, the bitrate, and the associated BSSID. It is the fastest way to confirm whether a wireless problem is signal or configuration.

## ethtool: Link Speed, Duplex, and Driver Info

```bash
# Link speed and duplex
sudo ethtool eth0

# Driver and firmware version
sudo ethtool -i eth0

# Ring buffer and offload settings
sudo ethtool -k eth0

# Statistics
sudo ethtool -S eth0
```

*Tip:* A link that negotiates 100 Mbps half-duplex on a gigabit network is a classic cause of "the network is slow". `ethtool eth0` shows the negotiated speed and duplex in one line.

## nmcli: NetworkManager from the CLI

On most modern desktop and RHEL family systems, NetworkManager owns the network configuration, and `nmcli` is the way to drive it.

```bash
# Show all connections
nmcli connection show

# Show device status
nmcli device status

# Bring a connection up or down
nmcli connection up eth0
nmcli connection down eth0

# Create a static connection
nmcli connection add type ethernet ifname eth0 con-name eth0-static \
  ipv4.method manual ipv4.addresses 10.0.0.5/24 ipv4.gateway 10.0.0.1 \
  ipv4.dns 8.8.8.8

# Modify an existing connection
nmcli connection modify eth0 ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection up eth0
```

*Tip:* `nmcli connection show` lists saved profiles, while `nmcli device status` shows the current state of each interface. When they disagree, the profile exists but is not applied, and `nmcli connection up <name>` fixes it.
