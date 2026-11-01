---
layout: post
title: "Running Your Own DNS Server: BIND and dnsmasq"
date: Oct-27-2012
categories: [Linux, Networking]
tags: [linux, dns, bind, dnsmasq, named, networking]
image: /assets/img/dns-server-bind-fiber-cables.jpg
---

`dig` and `getent hosts` tell you how to query DNS, but every one of those queries lands on a server that someone configured, maintains, and troubleshoots. Whether it's an internal zone for hosts that have no business being in public DNS, or a lightweight resolver for a home lab, running your own name server is a core piece of Linux networking that client-side tools alone never touch.

## Why Run Your Own DNS Server

Internal hostnames, split-horizon setups where internal and external clients should get different answers for the same name, and local network devices that public DNS has never heard of are all reasons a network ends up needing its own authoritative or caching name server rather than relying entirely on a public one.

```bash
# Confirm which resolver a client currently uses
cat /etc/resolv.conf

# Check if BIND or dnsmasq packages are already present
which named dnsmasq
```

*Tip:* You don't need to run your own DNS server for every network. The moment you have internal hostnames that should never leak into public DNS, or more than a handful of static IP-to-name mappings to manage, it starts paying for itself.

## Installing BIND

BIND (Berkeley Internet Name Domain) is the reference DNS server implementation, powering a large share of the internet's authoritative name servers.

```bash
# Debian/Ubuntu
sudo apt install bind9 bind9utils bind9-dnsutils

# RHEL/CentOS/Fedora
sudo dnf install bind bind-utils

# Confirm the service and its version
named -v
sudo systemctl status named    # or bind9 on Debian/Ubuntu
```

*Tip:* The package and service are named `bind9` on Debian/Ubuntu but `bind` (service `named`) on RHEL-family systems. The binary itself is `named` on both; only the package and unit names differ.

## Anatomy of a Zone File

A zone file is the authoritative record of every name-to-address mapping for a domain, starting with an SOA record that defines the zone's own metadata.

```text
; /etc/bind/zones/db.example.local
$TTL 86400
@   IN  SOA   ns1.example.local. admin.example.local. (
            2026010101  ; serial
            3600        ; refresh
            1800        ; retry
            604800      ; expire
            86400 )     ; minimum TTL

    IN  NS    ns1.example.local.
ns1 IN  A     192.168.1.10
web IN  A     192.168.1.20
db  IN  A     192.168.1.21
www IN  CNAME web
```

*Tip:* The serial number has to increase every time you edit the zone, or secondary name servers and resolvers that cached an earlier version won't notice the change. The common convention, `YYYYMMDDnn`, makes it obvious at a glance when a zone was last touched.

## Configuring a Forward Zone

Telling BIND the zone file exists and where to find it is a separate step from writing the zone file itself, done in the main configuration.

```text
// /etc/bind/named.conf.local
zone "example.local" {
    type master;
    file "/etc/bind/zones/db.example.local";
};
```

```bash
# Check the zone file syntax before reloading
sudo named-checkzone example.local /etc/bind/zones/db.example.local

# Check the overall configuration syntax
sudo named-checkconf

# Apply the change
sudo systemctl reload named
```

*Tip:* `named-checkzone` catches a missing trailing dot, a malformed serial, or a bad record long before `named` refuses to start. Run it after every edit, the same reflex as `nginx -t` before a reload.

## Configuring a Reverse Zone

Forward zones answer "what's the IP for this name"; reverse zones answer the opposite question, and they're easy to forget until something that depends on reverse lookups, like certain mail servers, starts failing.

```text
; /etc/bind/zones/db.192.168.1
$TTL 86400
@   IN  SOA   ns1.example.local. admin.example.local. (
            2026010101 3600 1800 604800 86400 )
    IN  NS    ns1.example.local.
10  IN  PTR   ns1.example.local.
20  IN  PTR   web.example.local.
```

```text
// named.conf.local
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

*Tip:* The reverse zone name is the IP's octets reversed with `.in-addr.arpa` appended. For `192.168.1.0/24`, that's `1.168.192.in-addr.arpa`, which trips up almost everyone the first time they write one by hand.

## Testing with named-checkzone and dig

Once BIND is running, confirm it's actually answering correctly by querying it directly, bypassing whatever resolver your system would normally use.

```bash
# Query your own server explicitly instead of the system resolver
dig @localhost web.example.local

# Reverse lookup
dig @localhost -x 192.168.1.20

# Query a secondary/different server for comparison
dig @8.8.8.8 example.com
```

*Tip:* Always query `@localhost` (or the server's own IP) explicitly while testing a new zone. Querying without specifying a server uses `/etc/resolv.conf`'s resolver, which may not even be pointed at the server you just configured.

## Running BIND as a Caching/Forwarding Resolver

Not every BIND instance needs to be authoritative for a zone. A pure caching resolver that forwards everything else upstream is a common, much simpler role.

```text
// /etc/bind/named.conf.options
options {
    recursion yes;
    allow-recursion { 192.168.1.0/24; };
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    dnssec-validation auto;
};
```

*Tip:* `allow-recursion` scoped to your own network is essential if `recursion` is enabled. An open recursive resolver reachable from the internet is a classic building block for DNS amplification attacks, and it will get found.

## dnsmasq as a Lightweight Alternative

BIND is powerful but heavyweight for a home network or small office. `dnsmasq` combines a caching DNS forwarder with a DHCP server in one lightweight package, which is exactly what most small networks actually need.

```bash
# Install
sudo apt install dnsmasq

# /etc/dnsmasq.conf, minimal useful config
# Serve local hostnames from /etc/hosts
expand-hosts
domain=lan.local

# Forward everything else upstream
server=1.1.1.1
server=8.8.8.8

# Also hand out DHCP leases
dhcp-range=192.168.1.100,192.168.1.200,12h
```

```bash
sudo systemctl restart dnsmasq
```

*Tip:* `dnsmasq` reading `/etc/hosts` means you manage your local network's name-to-IP mappings in the one file you're probably already familiar with, rather than learning zone file syntax, for the common small-network case.

## Securing BIND

An authoritative or recursive DNS server exposed to the internet is a frequent target; a handful of settings close off the most common abuse vectors.

```bash
# Use rndc to control a running named instance without restarting it
sudo rndc status
sudo rndc reload
sudo rndc flush

# Restrict zone transfers to known secondaries only
# allow-transfer { 192.168.1.11; };

# Rate-limit responses to reduce amplification attack usefulness
# rate-limit { responses-per-second 10; };
```

*Tip:* `allow-transfer` defaulting to "anyone" on an authoritative server hands your entire zone, every hostname and IP in it, to anyone who asks with `dig axfr`. Restrict it explicitly to your actual secondary name servers.

## Bonus: Troubleshooting DNS Propagation and TTLs

"DNS propagation" delay is really just caching: resolvers and clients hold onto an answer for as long as the record's TTL says they can, and nothing you do can force a remote cache to drop it sooner.

```bash
# Check the TTL a record is being served with
dig example.com | grep -A1 "ANSWER SECTION"

# Query several public resolvers to see how widely a change has propagated
for ns in 8.8.8.8 1.1.1.1 9.9.9.9; do dig @$ns example.com +short; done

# Lower a record's TTL well before a planned change, so caches expire faster when you do change it
```

*Tip:* Plan ahead for DNS changes: lower the TTL on a record days before you intend to change it, let that lower TTL itself propagate, then make the change. A low TTL set at the same moment as the change does nothing for caches that already have the old value with the old TTL.
