---
layout: post
title: "Linux Firewalls: nftables, iptables, firewalld, and ufw"
date: Mar-24-2012
categories: [Linux, Networking, Security]
tags: [linux, firewall, nftables, iptables, firewalld, ufw, security]
image: /assets/img/Linux_unix_bsd.png
---

Linux has four firewall front-ends in common use, and they are not interchangeable. `iptables` is legacy, `nftables` is its replacement, `firewalld` is a dynamic manager on top of nftables, and `ufw` is a simplified front-end for Ubuntu. This post maps them and shows the commands you actually need.

## Which One Am I Running?

```bash
# Is firewalld active?
systemctl status firewalld

# Is ufw active?
sudo ufw status

# Is nftables active?
systemctl status nftables

# Are there iptables rules?
sudo iptables -L -n -v

# Are there nftables rules?
sudo nft list ruleset
```

*Tip:* On modern systems, `iptables` commands are usually translated to nftables by `iptables-nft`. The rules you see with `iptables -L` may actually live in the nftables ruleset.

## nftables: The Modern Default

nftables uses tables, chains, and rules, with a cleaner syntax than iptables.

```bash
# Show the full ruleset
sudo nft list ruleset

# Create a table and chain
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'

# Allow established connections
sudo nft add rule inet filter input ct state established,related accept

# Allow SSH
sudo nft add rule inet filter input tcp dport 22 accept

# Allow loopback
sudo nft add rule inet filter input iif lo accept

# Save the ruleset
sudo nft list ruleset > /etc/nftables.conf
```

*Tip:* `nft` rules are applied immediately. If you lock yourself out over SSH, you need console access to recover, so always allow SSH before setting the default policy to `drop`.

## iptables: Legacy but Still Everywhere

```bash
# List rules with counters
sudo iptables -L -n -v

# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Drop everything else
sudo iptables -P INPUT DROP

# Save rules (Debian)
sudo iptables-save > /etc/iptables/rules.v4
```

*Tip:* `iptables -L -n -v` shows packet counters. A rule with zero matches is not the rule blocking you, and chasing it wastes time.

## firewalld: Dynamic Zones

firewalld organises rules into zones, each with its own trust level. It is the default on RHEL family systems.

```bash
# Show the active zone and its rules
sudo firewall-cmd --list-all

# List all zones
sudo firewall-cmd --get-zones

# Allow a service permanently
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

# Allow a port
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Allow a source subnet
sudo firewall-cmd --permanent --add-source=10.0.0.0/24 --zone=trusted
sudo firewall-cmd --reload
```

*Tip:* `--permanent` changes are not active until you run `--reload`. Forgetting the reload is the most common firewalld mistake.

## ufw: Simple Front-End

ufw is designed to be easy, and it is the default on Ubuntu.

```bash
# Enable and check status
sudo ufw enable
sudo ufw status verbose

# Allow SSH
sudo ufw allow 22/tcp

# Allow a service by name
sudo ufw allow 'Nginx Full'

# Allow from a specific source
sudo ufw allow from 10.0.0.0/24 to any port 5432

# Delete a rule
sudo ufw delete allow 22/tcp
```

*Tip:* `ufw allow OpenSSH` before `ufw enable` is the standard way to avoid locking yourself out on a fresh Ubuntu server.

## Debugging Firewall Drops

```bash
# nftables: log dropped packets
sudo nft add rule inet filter input log prefix "nft-drop: " drop

# iptables: log dropped packets
sudo iptables -A INPUT -j LOG --log-prefix "iptables-drop: "

# Watch the logs
sudo journalctl -k -f | grep -i drop
sudo dmesg -w | grep -i drop
```

*Tip:* Logging every drop is noisy. Add a temporary log rule, reproduce the problem, then remove it. Permanent drop logging will flood your disk.

## Quick Reference

| Task | nftables | iptables | firewalld | ufw |
| :--- | :--- | :--- | :--- | :--- |
| Show rules | `nft list ruleset` | `iptables -L -n -v` | `firewall-cmd --list-all` | `ufw status verbose` |
| Allow SSH | `nft add rule ... tcp dport 22 accept` | `iptables -A INPUT -p tcp --dport 22 -j ACCEPT` | `firewall-cmd --add-service=ssh` | `ufw allow 22/tcp` |
| Reload | `nft -f /etc/nftables.conf` | `iptables-restore` | `firewall-cmd --reload` | `ufw reload` |

*Tip:* Work in order: identify the active firewall, list its rules, add the allow rule, then verify with a connection test. Most "firewall is blocking me" incidents are a rule that was added but never reloaded.
