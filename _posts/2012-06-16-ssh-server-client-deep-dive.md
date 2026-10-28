---
layout: post
title: "SSH Deep Dive: sshd_config, authorized_keys, and Match Blocks"
date: Jun-16-2012
categories: [Linux, Security]
tags: [linux, ssh, sshd, security, hardening]
image: /assets/img/Linux_unix_bsd.png
---

SSH is the front door to every Linux host, and the default configuration is designed for compatibility, not security. This post covers the server and client settings that matter, the `authorized_keys` options that are almost never used, and the `Match` blocks that let you apply different rules to different users.

## The Server Configuration

```bash
# Main config
cat /etc/ssh/sshd_config

# Drop-in configs (newer OpenSSH)
ls /etc/ssh/sshd_config.d/

# Validate before restarting
sudo sshd -t

# Restart
sudo systemctl restart sshd
```

*Tip:* Always run `sshd -t` before restarting. A syntax error locks you out, and `sshd -t` catches it before you commit.

## Hardening sshd_config

```bash
# /etc/ssh/sshd_config

# Disable password auth entirely
PasswordAuthentication no
KbdInteractiveAuthentication no

# Disable root login
PermitRootLogin no

# Limit attempts
MaxAuthTries 3
MaxSessions 5

# Idle timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# Restrict users and groups
AllowGroups ssh-users

# Disable unused features
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no

# Modern algorithms only
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

*Tip:* `AllowTcpForwarding no` breaks SSH tunnels. If you rely on `-L` or `-D`, leave it at `yes` or restrict it to `local` only.

## authorized_keys Options

`authorized_keys` supports per-key options that restrict what a key can do. They are the cleanest way to grant narrow access.

```bash
# Restrict a key to a single command
command="/usr/local/bin/backup.sh",no-port-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA... backup@host

# Restrict a key to a source address
from="10.0.0.0/24",no-pty ssh-ed25519 AAAA... deploy@ci

# Expire a key with an expiry-time option (OpenSSH 7.6+)
expiry-time="20261231" ssh-ed25519 AAAA... temp@host
```

*Tip:* `command=` is the standard way to give a backup or CI system SSH access without giving it a shell. Combine it with `no-pty` and `no-port-forwarding` to close every other door.

## Match Blocks

`Match` blocks apply settings to a subset of connections, which is how you give one user a different policy without duplicating the whole config.

```bash
# /etc/ssh/sshd_config

# Defaults for everyone
PasswordAuthentication no

# Allow password auth only from the internal network
Match Address 10.0.0.0/8
    PasswordAuthentication yes

# Restrict a specific user to SFTP only
Match User backup
    ForceCommand internal-sftp
    ChrootDirectory /srv/backup
    AllowTcpForwarding no
    X11Forwarding no
```

*Tip:* `Match` blocks must come at the end of `sshd_config`. Anything after a `Match` block is treated as part of it, which is a common source of "my settings are ignored" confusion.

## Chroot SFTP

A chrooted SFTP account is the standard way to give a partner or a backup job file access without a shell.

```bash
# Create the user
sudo useradd -m -s /usr/sbin/nologin backup
sudo passwd -l backup

# Set up the chroot directory
sudo mkdir -p /srv/backup/upload
sudo chown root:root /srv/backup
sudo chmod 755 /srv/backup
sudo chown backup:backup /srv/backup/upload

# In sshd_config
Match User backup
    ForceCommand internal-sftp
    ChrootDirectory /srv/backup
    AllowTcpForwarding no
```

*Tip:* The chroot directory and every directory above it must be owned by root and not writable by the user. If they are not, `sshd` refuses the connection with a cryptic error.

## Client Configuration

```bash
# ~/.ssh/config

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    HashKnownHosts yes
    AddKeysToAgent yes
    IdentitiesOnly yes

Host bastion
    HostName bastion.example.com
    User alice
    IdentityFile ~/.ssh/id_ed25519

Host internal
    HostName 10.0.0.5
    User alice
    ProxyJump bastion
```

*Tip:* `IdentitiesOnly yes` stops the client from offering every key in the agent, which avoids "too many authentication failures" on servers with a low `MaxAuthTries`.

## Key Management

```bash
# Generate a modern key
ssh-keygen -t ed25519 -a 100 -C "alice@laptop"

# Copy the public key to a server
ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@host

# Show the fingerprint of a key
ssh-keygen -lf ~/.ssh/id_ed25519.pub

# Remove a host from known_hosts
ssh-keygen -R host.example.com

# List known hosts
ssh-keygen -F host.example.com
```

*Tip:* `ssh-keygen -R` is the correct way to remove a host from `known_hosts`. Editing the file by hand leaves hashed entries behind and the warning keeps coming back.

## Quick Reference

```bash
# Validate sshd config
sudo sshd -t

# Restrict a key to one command
command="/usr/local/bin/backup.sh",no-pty ssh-ed25519 AAAA...

# Chroot an SFTP user
Match User backup
    ForceCommand internal-sftp
    ChrootDirectory /srv/backup

# Remove a stale host key
ssh-keygen -R host.example.com
```

*Tip:* Work in order: harden `sshd_config`, restrict keys with `authorized_keys` options, then add `Match` blocks for exceptions. Most SSH incidents are a missing `sshd -t` or a `Match` block in the wrong place.
