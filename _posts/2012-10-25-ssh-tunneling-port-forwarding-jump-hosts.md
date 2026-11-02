---
layout: post
title: "SSH Tunneling: Port Forwarding, Jump Hosts, and SOCKS Proxies"
date: Oct-25-2012
categories: [Linux, Networking, Security]
tags: [linux, ssh, networking, tunneling, security]
image: /assets/img/ssh-tunnel-network-cables.jpg
---

An SSH connection is already encrypted and already authenticated, which makes it a convenient way to reach a service that has no business being exposed to the internet directly: a database behind a bastion, an internal web dashboard, a whole private network. This post covers the three forwarding modes SSH offers and how to chain them through a jump host.

## Local Port Forwarding (-L): Reach a Remote Service Locally

`-L` opens a port on your machine and tunnels anything sent to it, through the SSH connection, to a host and port reachable from the remote side.

```bash
# Forward local port 5433 to the remote-only Postgres instance on db-internal:5432,
# reached via a bastion host that has SSH access to both networks
ssh -L 5433:db-internal:5432 alice@bastion.example.com

# Now, in another terminal, connect as if Postgres were running locally
psql -h localhost -p 5433 -U dbuser mydb
```

*Tip:* The destination host/port (`db-internal:5432`) is resolved from the *remote* side, not your laptop — this is exactly why local forwarding works even when your machine has no route to `db-internal` at all.

## Remote Port Forwarding (-R): Expose Your Local Service to the Remote Side

`-R` is the mirror image: it opens a port on the remote host and tunnels connections back to a service running on your side.

```bash
# Let the remote host reach a web server running on your laptop at :3000,
# by exposing it as localhost:8080 on the remote machine
ssh -R 8080:localhost:3000 alice@remote.example.com

# On remote.example.com:
curl http://localhost:8080
```

*Tip:* By default the forwarded port only listens on the remote host's loopback interface. To let *other* machines on the remote network reach it, the remote `sshd_config` needs `GatewayPorts yes`, and even then, think carefully before exposing something that way.

## Dynamic Forwarding (-D): A Full SOCKS Proxy

`-D` turns the SSH connection into a generic SOCKS5 proxy, routing arbitrary TCP traffic through the remote host rather than forwarding one fixed port.

```bash
# Open a SOCKS proxy on localhost:1080
ssh -D 1080 alice@bastion.example.com

# Point a browser or curl at it
curl --socks5 localhost:1080 http://internal-dashboard.example.com

# Firefox: Settings > Network Settings > Manual proxy configuration
# SOCKS Host: localhost  Port: 1080  SOCKS v5
```

*Tip:* `-D` is the right tool when you need to reach an unknown number of internal hosts and ports through one bastion — rather than setting up a `-L` forward per service, configure your browser once to use the SOCKS proxy for anything on the internal domain.

## Jump Hosts with -J (ProxyJump)

Reaching a host that's only accessible from a bastion used to mean a manual two-hop SSH session; `-J` does it in one command.

```bash
# Jump through bastion.example.com to reach internal-host, which only
# accepts SSH connections from the bastion's network
ssh -J alice@bastion.example.com bob@internal-host

# Chain multiple jumps, comma-separated, if there are several hops
ssh -J alice@bastion1,alice@bastion2 bob@internal-host
```

```
# ~/.ssh/config equivalent, so you can just run `ssh internal-host`
Host bastion
    HostName bastion.example.com
    User alice

Host internal-host
    HostName internal-host.internal
    User bob
    ProxyJump bastion
```

*Tip:* `-J`/`ProxyJump` (OpenSSH 7.3+) replaced the older `ProxyCommand ssh -W %h:%p bastion` idiom. Both end up tunneling the full SSH protocol through the bastion, but `ProxyJump` is far easier to read and maintain in a config file.

## ProxyCommand for Anything -J Can't Express

For older SSH versions, or forwarding through something that isn't itself SSH, `ProxyCommand` is the more general mechanism `-J` is built on.

```
Host internal-host
    HostName internal-host.internal
    User bob
    ProxyCommand ssh -W %h:%p alice@bastion.example.com
```

```bash
# Equivalent one-off, without touching ~/.ssh/config
ssh -o "ProxyCommand=ssh -W %h:%p alice@bastion.example.com" bob@internal-host
```

*Tip:* `%h` and `%p` are substituted with the target host and port by SSH itself — don't hardcode `internal-host:22` into the `ProxyCommand`, since that breaks the moment you connect to a different host using the same config stanza pattern.

## Keeping Tunnels Alive

A tunnel sitting idle behind a NAT device or firewall with aggressive connection timeouts can silently die, which is especially disruptive for a long-running port forward.

```
# ~/.ssh/config
Host bastion
    HostName bastion.example.com
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

```bash
# ServerAliveInterval 30 + ServerAliveCountMax 3 means SSH gives up and exits
# after roughly 90 seconds of no response, rather than hanging forever
```

*Tip:* `ServerAliveInterval` is a client-side keepalive sent over the encrypted channel, which is different from TCP keepalives — it reliably detects a dead connection even through a NAT device that would otherwise let the TCP connection linger silently.

## Running Tunnels in the Background

A forwarding-only SSH session doesn't need an interactive shell; `-f` backgrounds it and `-N` skips allocating one.

```bash
# Background the tunnel, don't run a remote command, exit once authenticated
ssh -fN -L 5433:db-internal:5432 alice@bastion.example.com

# Find it later to kill it
ps aux | grep "5433:db-internal"
kill <pid>

# Or use autossh to automatically restart a dropped tunnel
autossh -M 0 -fN -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" \
  -L 5433:db-internal:5432 alice@bastion.example.com
```

*Tip:* Plain `ssh -fN` won't reconnect if the tunnel drops. For anything that needs to survive a flaky link or a bastion reboot unattended, `autossh` (or a systemd service with `Restart=always`) is worth the extra setup.

## Locking Down What a Bastion Allows

A bastion host that exists purely to forward connections shouldn't necessarily allow every kind of forwarding to every user, and `sshd_config` lets you be specific.

```
# /etc/ssh/sshd_config on the bastion

# Disable remote forwarding from untrusted users entirely
AllowTcpForwarding no

# Or allow only local forwarding (most bastions need this, not remote/dynamic)
AllowTcpForwarding local

# Never let forwarded ports bind beyond loopback unless you mean to
GatewayPorts no

# Restrict a specific key to tunneling only, no shell
Match User tunnel-only
    AllowTcpForwarding yes
    X11Forwarding no
    PermitTTY no
    ForceCommand /usr/sbin/nologin
```

*Tip:* `ForceCommand /usr/sbin/nologin` combined with `AllowTcpForwarding yes` gives you an account that can open tunnels through the bastion but can never get an interactive shell on it — a common pattern for a bastion dedicated purely to forwarding.
