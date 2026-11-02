---
layout: post
title: "tcpdump Basics: Capturing and Reading Packets on Linux"
date: Nov-15-2012
categories: [Linux, Networking]
tags: [linux, tcpdump, networking, packet-capture, troubleshooting]
image: /assets/img/tcpdump-network-cables.jpg
---

`ping` tells you a host is reachable. `curl -v` tells you what your own client did. Neither tells you what actually went across the wire. `tcpdump` captures the raw traffic itself, which is the only way to settle an argument between "the client sent it" and "the server never received it."

## Installing tcpdump

It needs raw socket access, so almost every capture command runs as root or via sudo.

```bash
sudo apt install tcpdump   # Debian/Ubuntu
sudo dnf install tcpdump   # RHEL/CentOS/Fedora

tcpdump --version
```

*Tip:* tcpdump only needs to see packets, not modify them, so capability `CAP_NET_RAW` is enough to run it without full root if you set it on the binary with `setcap`. Most admins don't bother and just use sudo.

## Picking an Interface

A capture has to be bound to one interface at a time, or explicitly to all of them.

```bash
# List available interfaces
sudo tcpdump -D

# Capture on a specific interface
sudo tcpdump -i eth0

# Capture on every interface (useful on a multi-homed box)
sudo tcpdump -i any
```

*Tip:* `-i any` is convenient for a quick look, but it strips some per-interface detail (like VLAN tags on some kernels). Pin to the real interface once you know which one matters.

## Writing to a File vs Watching Live

Live output scrolls past before you can read it on a busy host. Writing to a file lets you capture first and analyse calmly after.

```bash
# Watch live, one line per packet
sudo tcpdump -i eth0 -n

# Write to a pcap file instead of printing
sudo tcpdump -i eth0 -w /tmp/capture.pcap

# Stop after 500 packets
sudo tcpdump -i eth0 -w /tmp/capture.pcap -c 500
```

*Tip:* Always add `-n` for live viewing. Without it, tcpdump tries to resolve every IP to a hostname, which is slow and can make a fast-moving capture impossible to read.

## Filter Expressions: host, port, net

tcpdump's filter language (BPF) is what turns an unreadable firehose into exactly the packets you care about.

```bash
# Traffic to or from a specific host
sudo tcpdump -i eth0 host 10.0.0.15

# Traffic on a specific port, either direction
sudo tcpdump -i eth0 port 443

# Traffic to or from an entire subnet
sudo tcpdump -i eth0 net 10.0.0.0/24

# Only incoming SYN packets on port 80 (new connections)
sudo tcpdump -i eth0 'tcp dst port 80 and tcp[tcpflags] & tcp-syn != 0'
```

*Tip:* Filters push work down into the kernel before packets ever reach tcpdump's userspace process, so a tight filter isn't just more readable, it's measurably cheaper on a busy host.

## Combining Filters with and / or / not

Real troubleshooting filters are rarely a single condition.

```bash
# One host, specific port, excluding SSH so your own session doesn't show up
sudo tcpdump -i eth0 'host 10.0.0.15 and port 443 and not port 22'

# Either of two hosts
sudo tcpdump -i eth0 'host 10.0.0.15 or host 10.0.0.16'
```

*Tip:* Quote multi-word filter expressions. Without quotes, the shell will try to interpret `and`, `or`, and the parentheses itself and you'll get a confusing syntax error instead of a capture.

## Controlling How Much of Each Packet Is Captured

By default tcpdump grabs enough of each packet to show headers; grabbing full payloads costs more disk and CPU.

```bash
# Capture full packets, not just headers (needed to see payload content)
sudo tcpdump -i eth0 -s0 -w /tmp/full.pcap

# Older tcpdump defaults snaplen to 68 or 96 bytes; -s0 means "capture it all"
tcpdump --version
```

*Tip:* For HTTP/DNS payload inspection you need `-s0`. For "is traffic even arriving on this port," default snaplen headers are plenty and the capture file stays small.

## Reading a Saved Capture

A `.pcap` file is just as readable with `-r` as a live interface is with `-i`, and the same filters apply.

```bash
# Replay a saved capture through the normal filter/print pipeline
tcpdump -r /tmp/capture.pcap -n

# Apply a filter when reading back, same syntax as live capture
tcpdump -r /tmp/capture.pcap -n port 443
```

*Tip:* You don't need root to read a capture file you already own, only to capture live traffic. Hand a `.pcap` to a colleague and they can analyse it without any special privileges.

## Following a Single TCP Conversation

Isolating one client-server exchange out of a noisy capture is one of the most common real tasks.

```bash
# Everything between one client and one server on one port
tcpdump -r /tmp/capture.pcap -n 'host 10.0.0.15 and host 10.0.0.20 and port 443'

# Add -S for absolute sequence numbers instead of relative, useful for retransmission analysis
tcpdump -r /tmp/capture.pcap -n -S host 10.0.0.15
```

*Tip:* Repeated SYNs with no SYN-ACK response means the packet is leaving the host but nothing is answering, usually a firewall drop somewhere in the path, not an application bug.

## ASCII and Hex Dumps

For protocols that are plain text, or when you need to see exact bytes, `-X` prints both hex and ASCII side by side.

```bash
# Hex + ASCII dump of each packet
sudo tcpdump -i eth0 -X port 80

# ASCII only, handy for spotting plaintext credentials or headers
sudo tcpdump -i eth0 -A port 80
```

*Tip:* `-A` on port 80 traffic is a fast way to demonstrate, in a security review, exactly why plaintext HTTP for anything sensitive is a problem: the request and response are sitting right there in cleartext.

## Handing a Capture to Wireshark

tcpdump is unbeatable for grabbing traffic on a headless server; Wireshark is unbeatable for making sense of it visually afterward.

```bash
# Capture on the server
sudo tcpdump -i eth0 -w /tmp/capture.pcap -c 2000

# Copy it to your workstation
scp admin@server:/tmp/capture.pcap ./capture.pcap

# Open in Wireshark for protocol dissection, "Follow TCP Stream", and graphing
wireshark capture.pcap
```

*Tip:* Capture with tcpdump over SSH on the box that actually sees the problem, then analyse with Wireshark locally. Trying to run Wireshark's GUI over a remote session is painful; shipping the file is not.
