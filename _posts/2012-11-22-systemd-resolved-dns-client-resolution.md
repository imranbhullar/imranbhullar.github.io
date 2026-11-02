---
layout: post
title: "DNS Resolution on Linux: systemd-resolved, resolv.conf, and nsswitch"
date: Nov-22-2012
categories: [Linux, Networking]
tags: [linux, dns, systemd-resolved, networking]
image: /assets/img/dns-resolution-network-cables.jpg
---

Running your own DNS server is one thing; understanding how your own machine decides where to send a lookup is a different, and more commonly needed, skill. Most "DNS is broken" tickets turn out to be a stale cache, a confusing `resolv.conf` symlink, or a resolver nobody realised was in the chain.

## The Resolution Chain: nsswitch.conf

Name resolution on Linux doesn't go straight to DNS; it goes through `/etc/nsswitch.conf`, which decides the order of sources to check.

```bash
grep hosts /etc/nsswitch.conf
# hosts: files dns myhostname resolve [!UNAVAIL=return]
```

*Tip:* `files` before `dns` means `/etc/hosts` always wins for any name it defines. A hardcoded `/etc/hosts` entry is a common reason "DNS" looks wrong when DNS itself is actually fine.

## /etc/hosts: The First Stop

Simple, static, and checked before anything else touches the network.

```bash
cat /etc/hosts
# 127.0.0.1   localhost
# 10.0.0.5    db01.internal db01

# Quick local override for testing, bypassing DNS entirely
echo "10.0.0.99 staging.example.com" | sudo tee -a /etc/hosts
```

*Tip:* Use `/etc/hosts` for quick, temporary overrides when testing a change before it's live in DNS. Remove the line afterward; a forgotten override is a nasty, hard-to-spot bug months later.

## How systemd-resolved Fits In

On most current distros, `resolv.conf` doesn't point straight at your real DNS servers anymore, it points at a local stub resolver.

```bash
systemctl status systemd-resolved

# The stub listens here
ss -tulpn | grep 53
# udp  UNCONN  0  0  127.0.0.53:53  0.0.0.0:*  users:(("systemd-resolve",...))
```

*Tip:* Everything on the box talks to `127.0.0.53`, and systemd-resolved forwards to your real upstream servers and caches the results. This is why restarting systemd-resolved can "fix" DNS without touching your actual nameservers.

## Why /etc/resolv.conf Is Often a Symlink

Confusion about "which resolv.conf is real" is one of the most common DNS debugging dead ends.

```bash
ls -l /etc/resolv.conf
# lrwxrwxrwx 1 root root 39 Jan  1 00:00 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

cat /etc/resolv.conf
# nameserver 127.0.0.53
# options edns0 trust-ad
```

*Tip:* If `/etc/resolv.conf` is a regular file rather than this symlink, something (a VPN client, cloud-init, a container runtime) has overwritten it, and systemd-resolved's per-interface configuration is probably being bypassed.

## Per-Interface DNS with resolvectl

The actual upstream servers and search domains are configured per network interface, which `resolvectl` exposes.

```bash
# Overall status, per interface
resolvectl status

# Just the DNS servers in use
resolvectl status | grep -A2 "Current DNS Server"

# Set DNS servers for a specific interface manually
sudo resolvectl dns eth0 1.1.1.1 8.8.8.8
sudo resolvectl domain eth0 "~."
```

*Tip:* `resolvectl status` showing different servers per interface is normal and intentional on a multi-homed box, a VPN link should usually only resolve its own internal domain, not take over all lookups.

## DNS Caching Behaviour

systemd-resolved caches answers to cut down repeat lookups, which is great for performance and occasionally confusing when a record just changed.

```bash
# See cache statistics
resolvectl statistics

# Flush the cache after a DNS record change you need to see immediately
sudo resolvectl flush-caches

# Verify
resolvectl query example.com
```

*Tip:* Before assuming a DNS change "hasn't propagated yet," flush the local cache and query again. Caching on your own box is a far more common culprit than slow propagation upstream.

## Setting a Static Resolver Without systemd-resolved

Not every environment wants the stub resolver layer, particularly minimal containers or older-style setups.

```bash
# Disable systemd-resolved
sudo systemctl disable --now systemd-resolved

# Write resolv.conf directly as a real file
sudo rm /etc/resolv.conf
sudo tee /etc/resolv.conf <<'EOF'
nameserver 1.1.1.1
nameserver 8.8.8.8
EOF
```

*Tip:* NetworkManager will happily overwrite a hand-edited `resolv.conf` the next time it reconfigures an interface unless you also tell it to stop managing DNS (`dns=none` in `NetworkManager.conf`). Edit both or neither.

## Troubleshooting with resolvectl and dig

Two different tools answer two different questions: what did the system actually resolve, versus what does the authoritative server say.

```bash
# What systemd-resolved returned, including which server answered and from cache or not
resolvectl query example.com

# Bypass the stub entirely and ask a specific server directly
dig @8.8.8.8 example.com

# Check which record type and TTL you're actually getting
dig example.com +noall +answer
```

*Tip:* If `resolvectl query` and `dig @<upstream>` disagree, the problem is in caching or forwarding on your box. If they agree and both are wrong, the problem is upstream or in the zone itself.

## A Quick Reference for the Resolution Path

Worth keeping in mind as a single mental model when something looks wrong.

```text
Application lookup
  -> nsswitch.conf order (files, dns, ...)
      -> /etc/hosts  (if "files" matches first)
      -> 127.0.0.53  (systemd-resolved stub, if "dns" reaches it)
          -> cache, or forward to real upstream servers per interface
```

*Tip:* Walk this chain top to bottom when debugging. Nine times out of ten the fault is found before you ever have to question the actual authoritative DNS server.
