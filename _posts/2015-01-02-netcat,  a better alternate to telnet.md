---
layout: post
title: "Why Use Telnet When You Have Netcat? A Swiss Army Knife for Sysadmins"
date: 2015-01-02 12:00:00 +0000
description: "Explore why Netcat (nc) is the ultimate multi-purpose tool for port scanning, file transfers, and terminal-based communication."
categories: [SysAdmin, Networking]
tags: [netcat, netcat, linux, terminal, tools, networking]
---
![ssh](/assets/img/posts/netcat.jpeg)

From the day I started my career as a *nix admin, I always used to wonder why people don't use multi-purpose tools or better alternates for existing ones. That is where **Netcat (nc)** kicked in—a zillion times better alternative to telnet with tons of functionality. 

In today's post, let's explore why Netcat is considered the "Swiss Army Knife" of networking.

## Basic Syntax

The basic syntax of the `nc` command is straightforward:

```bash
nc [options] [host] [port]
```
### Netcat as a Port Scanner
One of the most common uses for Netcat is checking connectivity.

Testing TCP Connectivity
Let's check connectivity to port 80 of a webserver. We use `-z` (zero-I/O mode) to tell Netcat to report the status and then immediately close the connection. The `-v `flag provides verbosity so we can see the result.

```Bash
nc -zv test.enigmabits.net 80
 Connection to test.enigmabits.net port 80 [tcp/http] succeeded!
```
We can do the same for port 22 (SSH):

```Bash
nc -zv test.enigmabits.net 22
# Connection to test.enigmabits.net port 22 [tcp/ssh] succeeded!
```
Setting Timeouts
In cases of high latency, we can use the `-w` option to set a timeout in seconds:

```Bash
nc -zv -w 10 test.enigmabits.net 235
 nc: connect to test.enigmabits.net port 235 (tcp) failed: Connection refused
```
### Checking UDP Ports
Netcat can also check UDP connectivity using the `-u `option:

```Bash
nc -uz 8.8.8.8 53
# Connection to 8.8.8.8 port 53 [udp/domain] succeeded!
```
Scanning a Range of Ports
You can scan multiple ports at once by providing a range:

```Bash
nc -zv -w 1 test.enigmabits.net 21-30
```
Changing the Source Port
If you need to originate your request from a specific local port, use the `-p` flag:

```Bash
nc -zv -p 16000 test.enigmabits.net 22
Connection to test.enigmabits.net port 22 [tcp/ssh] succeeded!
```
### Netcat as a Chat Medium
Who doesn't want to have a quick chat with fellow sysadmins while sitting on the terminal of a remote machine? Netcat makes this incredibly simple.

* 1. On Machine A (The Listener):

```Bash
nc -l 6666
```
* 2. On Machine B (The Connector):

```Bash
nc machine_a_fqdn 6666
```
Now, whatever you type on one terminal appears on the other and vice-versa. It’s a perfect, lightweight way to communicate during a maintenance window.

### Transferring Files
Netcat is surprisingly efficient for moving files without the overhead of SCP or FTP.

* 1. On the Receiving Machine:
Prepare the machine to `listen` and redirect the output to a file:

```Bash
nc -l 6666 > file_received
* 2. On the Sending Machine:
Pipe the file into the connection:

Bash
nc enigmabits.net 6666 < send_file
Once the transfer finishes, the connection closes, and your file is ready on the target machine.
```

### Conclusion
We've only scratched the surface of what Netcat can do. While it works as a great alternative to telnet for simple port checks, its ability to listen, scan, and pipe data makes it much more powerful.

Enjoy feeding your NetCat today! 🐱