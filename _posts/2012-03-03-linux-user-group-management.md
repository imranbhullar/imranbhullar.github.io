---
layout: post
title: "Linux User and Group Management: useradd, usermod, and PAM Basics"
date: Mar-03-2012
categories: [Linux, SysAdmin]
tags: [linux, users, groups, useradd, pam, security]
image: /assets/img/Linux_unix_bsd.png
---

User and group management is one of those topics that looks trivial until you need to audit who has sudo, why a service cannot read its own files, or how to lock an account without deleting it. This post covers the commands, the files behind them, and the PAM layer that ties it all together.

## The Core Files

Everything about local users lives in four files under `/etc`.

```bash
# User accounts
cat /etc/passwd

# Password hashes and aging
sudo cat /etc/shadow

# Groups
cat /etc/group

# Group passwords (rarely used)
sudo cat /etc/gshadow
```

*Tip:* `/etc/passwd` is world-readable and must never contain a real password hash. The `x` in the second field means "look in `/etc/shadow`", which is readable only by root.

## Creating Users

```bash
# Create a user with a home directory and bash shell
sudo useradd -m -s /bin/bash alice

# Set the password
sudo passwd alice

# Create a system user with no login shell
sudo useradd -r -s /usr/sbin/nologin myservice

# Create a user and add to supplementary groups
sudo useradd -m -G sudo,docker alice
```

*Tip:* `useradd` does not create a home directory unless you pass `-m`. On Debian and Ubuntu, `adduser` is a friendlier wrapper that does this by default.

## Modifying Users

```bash
# Change the shell
sudo usermod -s /bin/zsh alice

# Change the home directory and move files
sudo usermod -d /home/alice2 -m alice

# Add to a supplementary group
sudo usermod -aG docker alice

# Lock and unlock an account
sudo usermod -L alice
sudo usermod -U alice

# Set an expiry date
sudo usermod -e 2026-12-31 alice
```

*Tip:* Always use `-aG` (append) when adding groups. Plain `-G` replaces the user's entire supplementary group list, which silently removes access you did not intend to touch.

## Deleting Users

```bash
# Remove the user but keep the home directory
sudo userdel alice

# Remove the user and their home directory
sudo userdel -r alice
```

*Tip:* Before deleting a user, check what they own with `find / -user alice`. Orphaned files with a numeric UID are a classic source of permission problems later.

## Managing Groups

```bash
# Create a group
sudo groupadd devops

# Add a user to a group
sudo gpasswd -a alice devops

# Remove a user from a group
sudo gpasswd -d alice devops

# List a user's groups
groups alice
id alice

# List members of a group
getent group devops
```

*Tip:* `id alice` shows both the primary group and all supplementary groups in one line, which is the fastest way to confirm a group change actually took effect.

## Password Aging with chage

```bash
# Show aging information
sudo chage -l alice

# Force a password change at next login
sudo chage -d 0 alice

# Set maximum password age to 90 days
sudo chage -M 90 alice

# Set account expiry
sudo chage -E 2026-12-31 alice
```

*Tip:* `chage -d 0` is the standard way to force a password reset without knowing the current password. It is what most "reset your password" workflows do under the hood.

## PAM Basics

PAM (Pluggable Authentication Modules) is the layer between applications and authentication. Every login, `su`, `sudo`, and `sshd` session goes through a PAM stack defined in `/etc/pam.d/`.

```bash
# List PAM configuration files
ls /etc/pam.d/

# Show the stack for sshd
cat /etc/pam.d/sshd

# Show the stack for su
cat /etc/pam.d/su
```

Each line has a type, a control flag, and a module:

* **Type:** `auth`, `account`, `password`, `session`
* **Control:** `required`, `requisite`, `sufficient`, `optional`
* **Module:** the `.so` file that implements the check

*Tip:* `required` runs the module and records failure but continues the stack. `requisite` fails immediately. This distinction matters when you are debugging why a login is rejected.

## Auditing Who Has Access

```bash
# Who can sudo?
getent group sudo
getent group wheel

# Who has a login shell?
getent passwd | awk -F: '$7 !~ /(nologin|false)$/ {print $1, $7}'

# Who has UID 0 besides root?
awk -F: '$3 == 0 {print $1}' /etc/passwd

# Recently modified accounts
sudo grep -E "useradd|usermod|userdel" /var/log/auth.log
```

*Tip:* The UID 0 check is a standard hardening step. Any account other than `root` with UID 0 is a backdoor, whether intentional or not.

## PAM Deep Dive

PAM stacks are more flexible than the four types suggest. These are the patterns you will use most often.

```bash
# /etc/pam.d/sshd

# Require a TOTP token
auth required pam_google_authenticator.so

# Run a script on login
session optional pam_exec.so /usr/local/bin/on-login.sh

# Restrict login by group
account required pam_access.so

# Enforce password quality
password requisite pam_pwquality.so retry=3 minlen=12
```

*Tip:* `pam_exec.so` is the cleanest way to run a script on login or logout. It is how most "notify on login" and "audit session" workflows are implemented.

## sssd: Centralised Identity

`sssd` connects a host to LDAP, Kerberos, or Active Directory, and caches the results.

```bash
# Install
sudo apt install sssd sssd-tools

# Configure
# /etc/sssd/sssd.conf
# [sssd]
# services = nss, pam
# domains = example.com
#
# [domain/example.com]
# id_provider = ldap
# auth_provider = ldap
# ldap_uri = ldap://ldap.example.com
# ldap_search_base = dc=example,dc=com

# Restart
sudo systemctl restart sssd

# Clear the cache
sudo sss_cache -E

# Check the cache
sudo sssctl user-checks alice
```

*Tip:* `sss_cache -E` is the fastest way to force a refresh after a change in the directory. Without it, the host keeps serving stale data.

## systemd-homed and userdbctl

`systemd-homed` manages home directories as portable, encrypted images, and `userdbctl` queries the user database.

```bash
# Create a homed user
sudo homectl create alice --storage=luks

# List homed users
homectl list

# Activate a home directory
sudo homectl activate alice

# Query the user database
userdbctl user alice

# List all users
userdbctl user
```

*Tip:* `userdbctl` is the modern replacement for `getent passwd`. It queries every configured user source, including `systemd-homed` and `sssd`.

## Quick Reference

```bash
# Create a user with a home directory
sudo useradd -m -s /bin/bash alice

# Add to a group without removing existing groups
sudo usermod -aG docker alice

# Lock an account
sudo usermod -L alice

# Force a password change
sudo chage -d 0 alice

# Audit UID 0 accounts
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

*Tip:* Work in order: create, add to groups, verify with `id`, then audit. Most access problems are a missing `-aG` or a group change that never took effect because the user did not log out and back in.
