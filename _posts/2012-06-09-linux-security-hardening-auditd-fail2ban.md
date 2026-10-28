---
layout: post
title: "Linux Security Hardening: auditd, fail2ban, and sshd"
date: Jun-09-2012
categories: [Linux, Security]
tags: [linux, security, hardening, auditd, fail2ban, sshd]
image: /assets/img/Linux_unix_bsd.png
---

Hardening a Linux host is not a single task; it is a set of small, well-understood changes that together make the system dramatically harder to compromise. This post covers the three that give the most value for the least effort: `sshd` hardening, `fail2ban`, and `auditd`.

## Hardening sshd

The SSH daemon is the most attacked service on any internet-facing host.

```bash
# /etc/ssh/sshd_config

# Disable password authentication
PasswordAuthentication no

# Disable root login
PermitRootLogin no

# Only allow specific users or groups
AllowUsers alice bob
AllowGroups ssh-users

# Limit authentication attempts
MaxAuthTries 3

# Disable unused authentication methods
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no

# Use only modern algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

```bash
# Validate the configuration before restarting
sudo sshd -t

# Restart
sudo systemctl restart sshd
```

*Tip:* Always run `sshd -t` before restarting. A syntax error in `sshd_config` will lock you out of the host, and `sshd -t` catches it before you commit.

## fail2ban: Blocking Brute Force

`fail2ban` watches log files and bans IPs that fail authentication repeatedly.

```bash
# Install
sudo apt install fail2ban

# Enable and start
sudo systemctl enable --now fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

A minimal jail configuration:

```bash
# /etc/fail2ban/jail.local
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = systemd
```

*Tip:* Never edit `/etc/fail2ban/jail.conf`. It is overwritten on upgrade. Put your changes in `jail.local` or a file under `/etc/fail2ban/jail.d/`.

## auditd: Auditing System Calls

`auditd` records security-relevant events: file access, privilege changes, and system calls.

```bash
# Install
sudo apt install auditd

# Enable and start
sudo systemctl enable --now auditd

# Check status
sudo auditctl -s

# List current rules
sudo auditctl -l
```

Useful rules:

```bash
# Watch /etc/passwd for changes
-w /etc/passwd -p wa -k passwd_changes

# Watch /etc/sudoers
-w /etc/sudoers -p wa -k sudoers_changes

# Log every execve
-a always,exit -F arch=b64 -S execve -k exec

# Log privilege escalation
-a always,exit -F arch=b64 -S setuid -S setgid -k privilege
```

*Tip:* Auditing every `execve` generates enormous volume. Start with file watches on the sensitive files, then add syscall rules only when you have a specific question to answer.

## Searching Audit Logs

```bash
# Search by key
sudo ausearch -k passwd_changes

# Search by time
sudo ausearch -ts today

# Search by user
sudo ausearch -ua 1000

# Generate a summary report
sudo aureport --summary
sudo aureport -au
```

*Tip:* `aureport --summary` gives you a one-page overview of authentication, file access, and system events. It is the fastest way to spot anomalies.

## Additional Hardening

```bash
# Disable unused services
systemctl list-unit-files --state=enabled
sudo systemctl disable --now cups

# Restrict cron to root
echo "root" | sudo tee /etc/cron.allow

# Set a restrictive umask
echo "umask 027" | sudo tee -a /etc/profile

# Enable automatic security updates (Debian/Ubuntu)
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

*Tip:* Disabling unused services is the highest-value hardening step. Every running service is an attack surface, and most servers run far more than they need.

## Quick Reference

```bash
# Validate sshd config
sudo sshd -t

# Check fail2ban
sudo fail2ban-client status sshd

# Search audit logs
sudo ausearch -k passwd_changes

# Summary report
sudo aureport --summary

# List enabled services
systemctl list-unit-files --state=enabled
```

*Tip:* Work in order: harden `sshd`, add `fail2ban`, enable `auditd`, then disable unused services. Those four steps cover the majority of real-world Linux hardening.
